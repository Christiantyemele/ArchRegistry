# Pattern Spec

This document defines the canonical structure every pattern in the registry must follow. It is the authoritative reference for authoring new patterns and the basis for automated validation in CI.

Every pattern is a Markdown file in `registry/patterns/{category}/`. The CI sync pipeline parses these files, generates embeddings, and upserts them into the database on every merge to `main`.

---

## File Location

```
registry/patterns/
├── database/
│   ├── transaction-atomic.md
│   ├── optimistic-locking.md
│   └── connection-pooling.md
├── caching/
│   ├── multi-level-cache.md
│   ├── request-level-cache.md
│   ├── session-level-cache.md
│   └── application-level-cache.md
├── http/
│   ├── idempotency-key.md
│   ├── rate-limiting.md
│   └── pagination.md
├── auth/
│   ├── jwt-validation.md
│   └── session-management.md
└── resilience/
    ├── circuit-breaker.md
    └── retry-with-backoff.md
```

File names must be kebab-case and match the pattern `id` in the frontmatter.

---

## Full Pattern Schema

```markdown
---
id: database-transaction-atomic
title: Atomic Database Transactions
tags: [write, acid, consistency, rollback, multi-step, postgresql]
operation_type: write
scope: backend
stack:
  languages: [rust, typescript, python]
  frameworks: [axum, express, fastapi, django]
  databases: [postgresql, mysql, sqlite]
relationships:
  requires: [idempotency-key-http]
  pairs_with: [multi-level-cache]
  conflicts_with: []
---

## Problem

Multi-step database operations executed as individual queries without a
transaction leave the database in a partial state if any step fails.
A network hiccup, a constraint violation, or an application error mid-sequence
can commit the first steps while the rest never run — leaving orphaned records,
missing audit logs, or inconsistent counters that are expensive to detect and
painful to repair.

## Solution

Wrap all related operations in an explicit transaction. Every grouped
operation succeeds entirely or rolls back entirely. There is no partial commit.

## Intent Signals

These are the natural-language phrases that should trigger retrieval of this
pattern during semantic search. Write them as a developer would describe
their task, not as an engineer would describe the pattern.

- multi-step write operation
- create related records together
- update balance with audit log
- process payment and record transaction
- any operation where partial success is worse than total failure
- rollback if anything fails

## Stack Variants

### rust-axum

```rust
use sqlx::PgPool;

pub async fn create_order_with_payment(
    pool: &PgPool,
    user_id: Uuid,
    amount: i64,
) -> Result<Order, AppError> {
    let mut tx = pool.begin().await?;

    let order = sqlx::query_as!(
        Order,
        "INSERT INTO orders (user_id, amount, status)
         VALUES ($1, $2, 'pending') RETURNING *",
        user_id, amount
    )
    .fetch_one(&mut *tx)
    .await?;

    sqlx::query!(
        "INSERT INTO audit_log (entity_id, action)
         VALUES ($1, 'order_created')",
        order.id
    )
    .execute(&mut *tx)
    .await?;

    tx.commit().await?;
    Ok(order)
}
```

### node-express

```typescript
import { pool } from './db';

async function createOrderWithPayment(
  userId: string,
  amount: number
): Promise<Order> {
  const client = await pool.connect();

  try {
    await client.query('BEGIN');

    const { rows: [order] } = await client.query(
      `INSERT INTO orders (user_id, amount, status)
       VALUES ($1, $2, 'pending') RETURNING *`,
      [userId, amount]
    );

    await client.query(
      `INSERT INTO audit_log (entity_id, action)
       VALUES ($1, 'order_created')`,
      [order.id]
    );

    await client.query('COMMIT');
    return order;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}
```

## Anti-Patterns

- **Running sequential queries without BEGIN/COMMIT.** If step 2 fails, step 1 has already committed. The database is now in an inconsistent state.
- **Catching errors without rolling back.** Swallowing an error inside a transaction block without calling ROLLBACK leaves the transaction open and the connection in a broken state.
- **Using transactions for single-statement operations.** A transaction around a single INSERT adds overhead with no safety benefit. Transactions are for groups of operations that must succeed or fail together.
- **Sending external side effects (email, webhooks) inside a transaction block.** External calls cannot be rolled back. Commit the database state first, then emit side effects after the transaction closes.

## Why This Matters

PostgreSQL guarantees ACID properties. The A — atomicity — means a transaction is all or nothing. Without wrapping related operations in a transaction, you are not using this guarantee. You are relying on luck and network reliability instead.
```

---

## Frontmatter Field Reference

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | Yes | Unique identifier, kebab-case, matches filename |
| `title` | string | Yes | Human-readable title |
| `tags` | string[] | Yes | Semantic tags used in both embedding and filtering |
| `operation_type` | string | Yes | `"read"` or `"write"` or `"both"` |
| `scope` | string | Yes | `"backend"` or `"frontend"` or `"both"` |
| `stack.languages` | string[] | Yes | Compatible languages. Use `["agnostic"]` if language-neutral |
| `stack.frameworks` | string[] | Yes | Compatible frameworks |
| `stack.databases` | string[] | No | Required if pattern is database-specific |
| `relationships.requires` | string[] | Yes | Pattern IDs that MUST also be applied |
| `relationships.pairs_with` | string[] | Yes | Pattern IDs that strongly complement this one |
| `relationships.conflicts_with` | string[] | Yes | Pattern IDs that are incompatible with this one |

---

## Required Sections

Every pattern must include all five of these sections in this order:

1. **Problem** — the failure mode this pattern prevents. Written in plain language. No jargon.
2. **Solution** — the approach in one to three sentences. Not implementation detail — the principle.
3. **Intent Signals** — a bullet list of natural-language phrases a developer would use when they need this pattern. These directly influence retrieval quality. Write them as a developer would describe their task.
4. **Stack Variants** — at least one concrete implementation. Code must be runnable, not pseudocode.
5. **Anti-Patterns** — what NOT to do, with a one-sentence explanation of why each is dangerous.

---

## What Makes a Good Pattern

A good pattern solves one specific problem. Not two. Not a category of problems. One problem with a clear failure mode if it is not applied.

A good intent signals section reads like a developer describing their task, not like an engineer describing a pattern. "multi-step write operation" is good. "ACID atomicity requirement" is not — no developer types that into a prompt.

A good stack variant is code you would actually deploy. Not a simplified toy example. Not pseudocode. Real error handling. Real types. The kind of code a senior engineer would put in a PR.

A good anti-patterns section names the exact mistake, not a vague warning. "Running sequential queries without a transaction" is good. "Not handling errors properly" is not.

---

## What Does Not Belong in the Registry

- Language syntax guides or tutorials
- Library documentation (link to it instead)
- Patterns so general they apply to everything (useless for retrieval)
- Patterns so narrow they apply to one codebase (belongs in a private registry)
- Anything that is an opinion rather than a pattern with a clear failure mode
