# Design

This document covers the system architecture, retrieval pipeline, database schema, and the reasoning behind key design decisions.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                     AGENT SIDE                          │
│   Cursor / Claude / Windsurf / any MCP-compatible tool  │
│   Calls: get_patterns() | get_related_patterns()        │
│          validate_intent() | get_pattern_by_id()        │
└──────────────────────┬──────────────────────────────────┘
                       │ MCP (SSE transport)
┌──────────────────────▼──────────────────────────────────┐
│                   MCP SERVER (Rust / Axum)              │
│                                                         │
│  ┌─────────────────┐     ┌──────────────────────────┐   │
│  │  Query Enricher │     │   Response Composer       │   │
│  │  - extract stack│     │   - builds structured     │   │
│  │  - extract intent│    │     artifact for agent    │   │
│  │  - tag inference│     │   - includes warnings     │   │
│  └────────┬────────┘     └──────────────┬───────────┘   │
│           │                             │               │
│  ┌────────▼─────────────────────────────▼───────────┐   │
│  │              Hybrid Retrieval Engine              │   │
│  │   Vector search  +  Structured filters            │   │
│  │   Reranker  +  Relationship graph resolver        │   │
│  └──────────┬──────────────────────────┬────────────┘   │
│             │                          │               │
│  ┌──────────▼──────────┐  ┌────────────▼────────────┐   │
│  │  pgvector (Supabase)│  │  PostgreSQL (Supabase)  │   │
│  │  - pattern vectors  │  │  - full pattern docs    │   │
│  │  - similarity search│  │  - relationship graph   │   │
│  │  - payload filters  │  │  - stack variants       │   │
│  └─────────────────────┘  └─────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## The Three-Layer Retrieval Pipeline

All retrieval logic runs entirely on the MCP server side. The agent calls a tool, waits, and receives a ranked list of patterns. Everything between those two events — embedding, search, filtering, reranking — is invisible to the agent and can be improved without changing the agent-facing interface.

### Layer 1 — Vector Search (Semantic)

Each pattern is embedded as a vector from a focused concatenation of its title, description, problem statement, intent signals, and tags. Incoming agent queries are embedded with the same model and compared via cosine similarity against all stored pattern vectors. Semantically close queries find the right pattern regardless of exact vocabulary.

This means "process a payment and create an order" and "multi-step write operation involving a charge" return the same patterns, even though the words share nothing in common.

### Layer 2 — Structured Filtering (Hard Constraints)

After vector search returns the top-K candidates, payload filters eliminate patterns incompatible with the agent's declared stack. A Rust pattern is not returned to a Python agent. A write-operation pattern is not returned to a read-only query context. These are not preferences — they are hard constraints enforced before reranking.

### Layer 3 — Relationship Resolution (Graph)

Every pattern declares typed relationships to other patterns: `requires`, `pairs_with`, `conflicts_with`. After retrieval and filtering, the server walks this graph and pulls in any `required` patterns the vector search may have scored below the cutoff. The graph is a safety net — even if the agent's query wording did not surface a critical pattern semantically, the relationship graph catches it and includes it anyway.

---

## What The Vector Database Stores

Each pattern is stored as a Qdrant/pgvector point with three parts:

```
Point
├── id       → "database-transaction-atomic"
├── vector   → [0.023, -0.891, 0.412, ...]   (1536 floats)
└── payload  → { tags, stack, operation_type, scope, ... }
```

The vector is generated from this concatenation of the pattern's semantic core:

```
"{title}. {description}. {problem_statement}. 
 Intent: {intent_signals joined}. 
 Tags: {tags joined}."
```

The full implementation guidance, code snippets, and anti-patterns are stored in PostgreSQL and fetched by ID after the vector search identifies the relevant patterns. This keeps the vector space clean — you are not embedding code, you are embedding meaning.

---

## Query Latency

```
Operation                          Latency (steady state)
──────────────────────────────────────────────────────────
Embed query (local model)          ~5–10ms
pgvector similarity search         ~5–20ms
Fetch full pattern documents       ~2–5ms
Relationship graph resolution      ~1–3ms
Response composition               ~1–2ms
──────────────────────────────────────────────────────────
Total                              ~15–40ms
```

Query embeddings are cached by hash. Warm cache hits (common queries) resolve the embedding step in under 1ms. The agent spends several seconds generating code — retrieval latency is not the bottleneck.

---

## Database Schema

```sql
-- Enable pgvector (available on Supabase by default)
CREATE EXTENSION IF NOT EXISTS vector;

-- Core patterns table
CREATE TABLE patterns (
    id              TEXT PRIMARY KEY,
    title           TEXT NOT NULL,
    description     TEXT NOT NULL,
    tags            TEXT[],
    operation_type  TEXT,
    scope           TEXT,
    stack           JSONB,
    relationships   JSONB,
    anti_patterns   JSONB,
    stack_variants  JSONB,
    intent_signals  TEXT[],
    embedding       vector(1536),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Vector similarity index
CREATE INDEX ON patterns
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- Structured filter indexes
CREATE INDEX ON patterns USING GIN (tags);
CREATE INDEX ON patterns USING GIN (stack);
CREATE INDEX ON patterns (operation_type);
CREATE INDEX ON patterns (scope);

-- Relationship graph
CREATE TABLE pattern_relationships (
    from_pattern_id TEXT REFERENCES patterns(id),
    to_pattern_id   TEXT REFERENCES patterns(id),
    relationship    TEXT CHECK (
        relationship IN ('requires', 'pairs_with', 'conflicts_with')
    ),
    PRIMARY KEY (from_pattern_id, to_pattern_id, relationship)
);
```

---

## Design Decisions

### Why Rust for the MCP server?

Performance, memory safety, and correctness under concurrent load. The retrieval pipeline runs multiple operations in parallel — embedding lookup, vector search, relationship resolution — and Rust's ownership model makes this safe without a garbage collector introducing latency spikes.

### Why Markdown as the pattern source of truth?

Markdown lives in Git. Git gives you history, review, blame, and a contribution workflow for free. Every pattern change is a PR. Every PR can be reviewed for quality before merge. The sync pipeline handles the translation to the database automatically — contributors never touch SQL or manually generate embeddings.

### Why pgvector over a dedicated vector database like Qdrant?

At registry scale — even 100,000 patterns — pgvector on Supabase handles the load comfortably and eliminates operational complexity. You are already on Supabase. pgvector is already enabled. There is no second database to provision, pay for, or keep in sync. The migration path to Qdrant is straightforward if scale ever demands it, but optimising for that before it is needed is premature.

### Why not embed the full pattern document?

Embedding the full document including code snippets and implementation notes pollutes the semantic signal with vocabulary that is irrelevant to meaning — variable names, library names, syntax. The embedding is generated from a focused concatenation of the semantic core only: description, intent signals, problem statement, and tags. The full document is stored in PostgreSQL and fetched by ID after retrieval.

### Why two separate stores (pgvector + PostgreSQL tables)?

pgvector is optimised for one thing — fast cosine similarity search. PostgreSQL relational tables are optimised for structured lookups, joins, and the relationship graph. Each is used for what it does best. Since both live inside the same Supabase instance, there is no cross-service latency — it is two queries to the same database.

### Response generation — does an LLM synthesise the response?

Not by default. The MCP server composes the response directly from retrieved pattern documents — structured JSON returned to the agent. The agent's own LLM then reasons about the patterns to generate code. This is faster, cheaper, and fully deterministic.

A server-side LLM synthesis step (where the registry pre-digests patterns into a context-specific recommendation before returning to the agent) is a planned Phase 4 enhancement for when the registry is large enough that returning raw pattern documents risks overwhelming the agent's context window.
