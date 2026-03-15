# MCP Tools Reference

This document describes every tool the ArchRegistry MCP server exposes. These are the tool signatures the agent calls — all retrieval logic runs server-side and is invisible to the agent.

---

## Shared Types

```typescript
interface Stack {
  language: string      // "rust" | "typescript" | "python" | "go" | ...
  framework: string     // "axum" | "nextjs" | "fastapi" | "gin" | ...
  database: string      // "postgresql" | "mongodb" | "mysql" | "sqlite"
  infra: string[]       // ["supabase", "redis", "stripe", "aws", ...]
}

interface ResolvedPattern {
  id: string
  title: string
  reason_matched: string       // why this pattern was returned
  relevance_score: number      // cosine similarity score from vector search
  relationship_type: string    // "direct_match" | "required" | "recommended"
  implementation: string       // stack-specific implementation guidance
  pairs_with: string[]         // IDs of complementary patterns
  anti_patterns: AntiPattern[]
}

interface AntiPattern {
  description: string
  why: string
}

interface PatternResponse {
  patterns: ResolvedPattern[]
  warnings: string[]
  anti_patterns: AntiPattern[]
}
```

---

## `get_patterns`

**Purpose:** Primary retrieval tool. Call this when the agent needs architectural guidance before generating code for a task.

**When to call:** Any time the agent is about to write a function, handler, service, or module that involves database operations, HTTP interactions, caching, authentication, or any multi-step operation.

```typescript
get_patterns({
  intent: string,         // what the agent is trying to do
  stack: Stack,
  context?: string,       // current file contents or task description
  tags?: string[]         // optional hints: ["write", "auth", "payment"]
}) → PatternResponse
```

**Example call:**

```typescript
get_patterns({
  intent: "process a payment and create an order record",
  stack: {
    language: "rust",
    framework: "axum",
    database: "postgresql",
    infra: ["supabase", "stripe"]
  },
  tags: ["payment", "write", "multi-step"]
})
```

**Example response:**

```json
{
  "patterns": [
    {
      "id": "database-transaction-atomic",
      "title": "Atomic Database Transactions",
      "reason_matched": "multi-step write operation involving payment and record creation",
      "relevance_score": 0.94,
      "relationship_type": "direct_match",
      "implementation": "...",
      "pairs_with": ["idempotency-key-http"]
    },
    {
      "id": "idempotency-key-http",
      "title": "HTTP Idempotency Keys",
      "reason_matched": "required by database-transaction-atomic — payment operations must handle retries safely",
      "relevance_score": 0.91,
      "relationship_type": "required",
      "implementation": "..."
    }
  ],
  "warnings": [
    "payment operations must always be paired with idempotency-key-http to handle network retries safely"
  ],
  "anti_patterns": []
}
```

---

## `get_pattern_by_id`

**Purpose:** Direct lookup for a known pattern ID. Used when a pattern references another, or when the agent wants the full implementation detail for a specific pattern it already knows about.

```typescript
get_pattern_by_id({
  id: string,
  stack_variant: string   // "rust-axum" | "node-express" | "python-fastapi" | ...
}) → ResolvedPattern
```

**Example call:**

```typescript
get_pattern_by_id({
  id: "idempotency-key-http",
  stack_variant: "rust-axum"
})
```

---

## `get_related_patterns`

**Purpose:** Given a pattern the agent is already applying, return everything it should also consider. This is how the composability graph is exposed to the agent — it can walk the graph explicitly after an initial `get_patterns` call.

```typescript
get_related_patterns({
  pattern_id: string,
  relationship: "requires" | "pairs_with" | "conflicts_with"
}) → ResolvedPattern[]
```

**Example call:**

```typescript
// What does database-transaction-atomic require?
get_related_patterns({
  pattern_id: "database-transaction-atomic",
  relationship: "requires"
})

// What does it pair well with?
get_related_patterns({
  pattern_id: "database-transaction-atomic",
  relationship: "pairs_with"
})
```

**Relationship semantics:**

| Relationship | Meaning |
|---|---|
| `requires` | The related pattern MUST also be applied. The current pattern is incomplete or unsafe without it. |
| `pairs_with` | The related pattern is strongly recommended alongside this one. Not mandatory, but the combination is much stronger than either alone. |
| `conflicts_with` | The related pattern should NOT be applied at the same time. Applying both leads to contradictory behaviour or architecture. |

---

## `validate_intent`

**Purpose:** Pre-generation architectural check. The agent describes what it is about to write before generating any code. The registry returns applicable patterns, anti-patterns to avoid, and any warnings about the proposed approach.

This is the most powerful tool in the registry — it catches architectural mistakes before code exists, not after.

```typescript
validate_intent({
  description: string,         // what the agent is about to generate
  stack: Stack,
  proposed_approach?: string   // optional: what the agent is planning to do
}) → {
  applicable_patterns: ResolvedPattern[],
  anti_patterns: AntiPattern[],
  warnings: string[]
}
```

**Example call:**

```typescript
validate_intent({
  description: "background job that sends a welcome email and increments the user onboarding_step counter",
  stack: {
    language: "rust",
    framework: "axum",
    database: "postgresql",
    infra: ["supabase"]
  },
  proposed_approach: "wrap both operations in a single transaction and send the email inside it"
})
```

**Example response:**

```json
{
  "applicable_patterns": [
    { "id": "database-transaction-atomic", ... },
    { "id": "idempotency-key-background-job", ... }
  ],
  "anti_patterns": [
    {
      "description": "do not send the email inside the transaction block",
      "why": "external side effects like email cannot be rolled back. if the transaction rolls back, the email has already been sent. commit the DB state first, emit the email after."
    }
  ],
  "warnings": [
    "incrementing a counter is not idempotent — use upsert or a conditional update to make retries safe"
  ]
}
```
