# Implementation Plan

Phased delivery plan for the ArchRegistry core team. Phases 1–3 bring the project to public launch. Phase 4 opens the hosted tier.

Each phase has a clear exit criterion — do not start the next phase until the current one is complete and stable.

---

## Phase 1 — Foundation (Weeks 1–4)

Core team only. No public contributors. Goal: a working end-to-end retrieval pipeline with 20 seed patterns and a real agent connected.

### Server

- [ ] Scaffold Rust MCP server with Axum
- [ ] Implement SSE transport for MCP protocol
- [ ] Implement `get_patterns` tool with hybrid retrieval (vector + structured filter)
- [ ] Implement `get_pattern_by_id` tool
- [ ] Basic error handling and logging

### Database

- [ ] Provision Supabase project
- [ ] Enable pgvector extension
- [ ] Run initial migrations (patterns table, relationship table, indexes)
- [ ] Verify vector similarity query performance on seed data

### Embedding Pipeline

- [ ] Integrate local embedding model (`nomic-embed-text` via Ollama)
- [ ] OpenAI `text-embedding-3-small` as fallback
- [ ] In-memory embedding cache keyed by query hash
- [ ] Embedding generation script for seeding

### CI Sync Pipeline

- [ ] Markdown pattern parser (frontmatter + sections)
- [ ] Embedding generation on parse
- [ ] Upsert to PostgreSQL + pgvector on merge to `main`
- [ ] Fail CI on malformed pattern files

### Seed Patterns — 20 patterns minimum across

- [ ] `database/` — transaction-atomic, connection-pooling
- [ ] `caching/` — multi-level-cache, request-level-cache, session-level-cache, application-level-cache
- [ ] `http/` — idempotency-key, rate-limiting, pagination
- [ ] `auth/` — jwt-validation, session-management
- [ ] `resilience/` — retry-with-backoff, circuit-breaker

### Tests

- [ ] Retrieval quality tests: 10 (query, expected_pattern_ids) pairs
- [ ] `get_patterns` tool contract test
- [ ] `get_pattern_by_id` tool contract test

### Exit Criterion

A real Cursor agent, connected to the local MCP server, retrieves the correct patterns for a payment handler and a user profile endpoint without any manual intervention.

---

## Phase 2 — Complete Core Tools (Weeks 5–8)

Goal: all four MCP tools working, relationship graph active, retrieval quality measurably good.

### Tools

- [ ] Implement `get_related_patterns` tool
- [ ] Implement `validate_intent` tool with anti-pattern detection
- [ ] Implement proposed_approach evaluation in `validate_intent`

### Retrieval

- [ ] Relationship graph resolver (walks `requires` edges, pulls missing patterns)
- [ ] Reranking layer (score fusion: vector similarity + stack specificity + tag overlap + relationship boost)
- [ ] Relationship boost: required patterns always appear in results regardless of vector score

### Patterns

- [ ] Expand to 50 patterns, adding TypeScript stack variants to all existing patterns
- [ ] Audit and improve intent signals on all Phase 1 patterns based on retrieval testing

### Tests

- [ ] Retrieval quality tests expanded to 30 (query, expected) pairs
- [ ] `get_related_patterns` tool contract test
- [ ] `validate_intent` tool contract test
- [ ] Relationship graph test: `requires` edges always pull in required pattern even when vector score is below cutoff
- [ ] Anti-pattern detection test: known dangerous combinations surface the correct warning
- [ ] End-to-end integration test with a real Cursor agent across all four tools

### Exit Criterion

All four tools pass their contract tests. Retrieval quality tests pass with recall@5 ≥ 0.85 across the full eval set. The relationship graph correctly pulls in `required` patterns in 100% of test cases.

---

## Phase 3 — Polish & Open for Contributors (Weeks 9–12)

Goal: the project is ready for public contributors. Quality bar is established and enforced automatically.

### Contributor Infrastructure

- [ ] `CONTRIBUTING.md` — full authoring guide, PR process, review criteria
- [ ] `PATTERN_SPEC.md` is finalised and linked from CONTRIBUTING
- [ ] Pattern validation CI — automated checks on all PRs:
  - frontmatter schema validation
  - required sections present
  - relationship ID references exist
  - at least one stack variant with non-empty code block
  - intent signals list has at least 5 entries
- [ ] Issue templates: new pattern, new stack variant, retrieval bug report

### Developer Experience

- [ ] Docker Compose for full local development (server + Supabase local + Ollama)
- [ ] `make seed` command to bootstrap a fresh DB with all patterns
- [ ] `make eval` command to run the full retrieval quality test suite
- [ ] README installation guide tested end-to-end on a clean machine

### Patterns

- [ ] Expand to 100 patterns
- [ ] Rust and TypeScript variants on all patterns
- [ ] Python (FastAPI) variants on the 20 most common patterns
- [ ] All relationship edges audited and verified

### Tests

- [ ] Retrieval eval set expanded to 60 (query, expected) pairs
- [ ] Regression test suite: one test per known retrieval bug found during Phase 1–2
- [ ] Performance test: p99 latency under 50ms for `get_patterns` with warm cache

### Launch

- [ ] Public GitHub repository
- [ ] Launch post (dev.to / Hacker Show HN)
- [ ] First issue batch labelled `good first issue` for incoming contributors

### Exit Criterion

A developer with no prior context can clone the repo, run Docker Compose, and have a working local MCP server connected to their Cursor agent in under 15 minutes. Pattern contribution PR process works end-to-end with automated validation.

---

## Phase 4 — Hosted Tier (Post-launch)

Goal: a managed hosted instance that developers and teams can use without self-hosting.

### Infrastructure

- [ ] Deploy MCP server to production (Fly.io or Railway)
- [ ] Production Supabase instance with connection pooling (pgBouncer)
- [ ] Rate limiting per API key
- [ ] API key issuance and management

### Auth & Billing

- [ ] API key generation on signup
- [ ] Usage tracking per key (requests/month)
- [ ] Free tier: 500 requests/month
- [ ] Paid tier: $9–19/month for unlimited individual use
- [ ] Stripe integration for billing

### Observability

- [ ] Prometheus metrics: request count, latency histograms, cache hit rate, retrieval scores
- [ ] Grafana dashboard
- [ ] Alerting on p99 latency spikes and error rate

### Team Features (enables Team Tier)

- [ ] Private registry: teams can add patterns visible only to their API key
- [ ] Pattern enforcement reporting: which patterns is your team's agent pulling?
- [ ] Admin controls: mark patterns as mandatory for a team

### Exit Criterion

Hosted instance has been running for 30 days with p99 latency under 40ms, zero data loss incidents, and at least 10 paying individual subscribers.
