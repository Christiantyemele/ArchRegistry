# Use Case Flows

End-to-end flows showing ArchRegistry in action. Each flow traces the full journey from developer prompt to governed code generation.

---

## Flow 1 — Payment Handler

**Developer prompt:** *"implement a function to process a payment and create an order"*

**Stack:** Rust + Axum + PostgreSQL + Supabase + Stripe

```
Step 1 — Agent calls get_patterns()

  get_patterns({
    intent: "process payment and create order record",
    stack: {
      language: "rust",
      framework: "axum",
      database: "postgresql",
      infra: ["supabase", "stripe"]
    },
    tags: ["payment", "write", "multi-step"]
  })

Step 2 — Server embeds the query
  "process payment and create order record. Tags: payment write multi-step"
  → vector: [0.031, -0.887, 0.398, ...]

Step 3 — pgvector similarity search
  Top results by cosine similarity:
  - database-transaction-atomic    score: 0.94
  - idempotency-key-http           score: 0.91
  - multi-level-cache              score: 0.71

Step 4 — Structured filter
  operation_type = "write" → multi-level-cache eliminated
  (cache pattern is for reads, not payment writes)

Step 5 — Relationship resolution
  database-transaction-atomic.requires → [idempotency-key-http]
  idempotency-key-http already in result set → boost rank, mark as "required"

Step 6 — Server composes response and returns to agent

  {
    patterns: [
      {
        id: "database-transaction-atomic",
        relationship_type: "direct_match",
        reason_matched: "multi-step write involving payment and record creation",
        implementation: "<rust-axum code guidance>"
      },
      {
        id: "idempotency-key-http",
        relationship_type: "required",
        reason_matched: "required by database-transaction-atomic —
                         payment operations must handle retries safely",
        implementation: "<rust-axum code guidance>"
      }
    ],
    warnings: [
      "payment operations must always be paired with idempotency-key-http
       to handle network retries safely"
    ]
  }

Step 7 — Agent generates code

  ✓ Wraps the DB write in a transaction (BEGIN / COMMIT / ROLLBACK)
  ✓ Accepts and validates an Idempotency-Key header
  ✓ Returns cached response for duplicate requests within TTL
  ✓ Does NOT send webhook or email inside the transaction block
```

---

## Flow 2 — User Profile Endpoint

**Developer prompt:** *"create a GET endpoint that returns the authenticated user's profile"*

**Stack:** Rust + Axum + PostgreSQL + Supabase

```
Step 1 — Agent calls get_patterns()

  get_patterns({
    intent: "fetch and return user profile data for authenticated user",
    stack: {
      language: "rust",
      framework: "axum",
      database: "postgresql",
      infra: ["supabase"]
    },
    tags: ["read", "user", "profile", "auth"]
  })

Step 2 — Vector search returns
  - request-level-cache    score: 0.89
  - multi-level-cache      score: 0.85
  - jwt-validation         score: 0.82

Step 3 — Structured filter
  operation_type = "read" → all three pass
  scope = "backend" → all three pass

Step 4 — Relationship resolution
  jwt-validation.requires → [session-management] — pulled in
  multi-level-cache.pairs_with → [request-level-cache] — already present

Step 5 — Agent generates code

  ✓ Validates the JWT token before touching the database
  ✓ Checks request-level cache before issuing any DB query
  ✓ Falls back through cache levels (request → session → application → DB)
  ✓ Populates all cache levels on a DB hit
```

---

## Flow 3 — Pre-generation Validation

**Developer prompt:** *"write a background job that sends a welcome email and increments the user's onboarding_step counter"*

**Proposed approach (agent's plan):** *"wrap both operations in a single transaction and send the email inside it"*

```
Step 1 — Agent calls validate_intent() before writing any code

  validate_intent({
    description: "background job that sends a welcome email and
                  increments the user onboarding_step counter",
    stack: {
      language: "rust",
      framework: "axum",
      database: "postgresql",
      infra: ["supabase"]
    },
    proposed_approach: "wrap both operations in a single transaction
                        and send the email inside it"
  })

Step 2 — Server evaluates the proposed approach against the registry

  Applicable patterns identified:
  - database-transaction-atomic
  - idempotency-key-background-job

  Proposed approach triggers anti-pattern detection:
  - "send email inside transaction" matches known anti-pattern:
    external side effects inside transaction blocks

Step 3 — Server returns to agent

  {
    applicable_patterns: [
      { id: "database-transaction-atomic", ... },
      { id: "idempotency-key-background-job", ... }
    ],
    anti_patterns: [
      {
        description: "do not send the email inside the transaction block",
        why: "email cannot be rolled back. if the transaction rolls back,
              the email has already been sent. commit the DB write first,
              emit the email after the transaction closes."
      }
    ],
    warnings: [
      "incrementing a counter is not idempotent — use upsert or a
       conditional update so retries do not double-increment"
    ]
  }

Step 4 — Agent corrects its approach before writing code

  ✓ Wraps only the DB increment in a transaction
  ✓ Sends the email AFTER the transaction commits successfully
  ✓ Uses a conditional update for the counter to make retries safe
  ✓ Applies idempotency key so job retries do not re-send the email
```

---

## Flow 4 — New Contributor Adds a Pattern

**Contributor:** adding an `optimistic-locking` pattern for PostgreSQL

```
Step 1 — Contributor creates file
  registry/patterns/database/optimistic-locking.md
  (following PATTERN_SPEC.md)

Step 2 — Contributor opens a PR
  PR triggers CI validation:
  - frontmatter schema check → passes
  - required sections check → passes
  - referenced relationship IDs exist → passes
  - at least one stack variant with real code → passes

Step 3 — Core team reviews
  - Intent signals are written from developer perspective?
  - Anti-patterns are specific, not vague?
  - Code is deployable, not pseudocode?
  - Relationships are correct?

Step 4 — PR merges to main

Step 5 — CI sync pipeline runs automatically
  - Parses optimistic-locking.md
  - Generates embedding from semantic core
  - Upserts pattern into PostgreSQL
  - Upserts vector into pgvector
  - Updates relationship graph edges

Step 6 — Pattern is live in the registry
  Any agent querying for "concurrent update conflicts" or
  "prevent lost updates" will now receive this pattern
```
