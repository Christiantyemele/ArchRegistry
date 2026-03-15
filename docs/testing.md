# Testing Strategy

Testing in ArchRegistry is primarily about **retrieval quality** — ensuring the right patterns are returned for the right queries. This is different from most software testing. There is no single correct output, no deterministic function to verify. The tests are evaluation pairs that measure how well the semantic search is working.

---

## Test Categories

### 1. Retrieval Quality Tests

The most important tests in the project. Each test is a `(query, expected_pattern_ids)` pair asserting that the full retrieval pipeline — embedding, vector search, filter, reranker, relationship resolver — returns the expected patterns for a given intent.

These tests catch regressions when embeddings are regenerated, when the reranker weights are adjusted, or when new patterns are added that might displace existing ones.

```rust
// tests/retrieval/payment_handler.rs

#[tokio::test]
async fn payment_handler_returns_transaction_and_idempotency() {
    let client = test_client().await;

    let results = client.get_patterns(PatternQuery {
        intent: "process a payment and create an order record".into(),
        stack: Stack {
            language: "rust".into(),
            framework: "axum".into(),
            database: "postgresql".into(),
            infra: vec!["supabase".into(), "stripe".into()],
        },
        tags: vec!["payment".into(), "write".into()],
        context: None,
    }).await.unwrap();

    let ids: Vec<&str> = results.patterns.iter()
        .map(|p| p.id.as_str())
        .collect();

    assert!(ids.contains(&"database-transaction-atomic"),
        "payment handler must return transaction pattern");
    assert!(ids.contains(&"idempotency-key-http"),
        "payment handler must return idempotency pattern");
}

#[tokio::test]
async fn user_profile_endpoint_returns_cache_and_auth() {
    let client = test_client().await;

    let results = client.get_patterns(PatternQuery {
        intent: "fetch and return authenticated user profile".into(),
        stack: Stack {
            language: "rust".into(),
            framework: "axum".into(),
            database: "postgresql".into(),
            infra: vec!["supabase".into()],
        },
        tags: vec!["read".into(), "user".into(), "auth".into()],
        context: None,
    }).await.unwrap();

    let ids: Vec<&str> = results.patterns.iter()
        .map(|p| p.id.as_str())
        .collect();

    assert!(ids.contains(&"jwt-validation"));
    assert!(ids.contains(&"request-level-cache") || ids.contains(&"multi-level-cache"));
}
```

**Recall target:** retrieval quality tests must pass with recall@5 ≥ 0.85 across the full eval set. This means for at least 85% of queries, the expected pattern appears in the top 5 results.

---

### 2. MCP Tool Contract Tests

Assert that each tool returns the correct response schema and that required fields are always present. These are fast, deterministic unit tests that do not depend on the embedding model or vector search.

```rust
// tests/tools/get_patterns_contract.rs

#[tokio::test]
async fn get_patterns_response_has_required_fields() {
    let client = test_client().await;

    let results = client.get_patterns(minimal_query()).await.unwrap();

    // Every pattern in the response must have these fields
    for pattern in &results.patterns {
        assert!(!pattern.id.is_empty(), "pattern.id must not be empty");
        assert!(!pattern.title.is_empty(), "pattern.title must not be empty");
        assert!(!pattern.reason_matched.is_empty(),
            "pattern.reason_matched must explain why it was returned");
        assert!(pattern.relevance_score > 0.0,
            "pattern.relevance_score must be positive");
        assert!(
            ["direct_match", "required", "recommended"]
                .contains(&pattern.relationship_type.as_str()),
            "pattern.relationship_type must be a valid value"
        );
    }
}

#[tokio::test]
async fn get_patterns_returns_empty_not_error_for_unknown_stack() {
    let client = test_client().await;

    // Unknown stack should return empty results, not a 500
    let results = client.get_patterns(PatternQuery {
        intent: "some task".into(),
        stack: Stack {
            language: "cobol".into(),
            framework: "unknown".into(),
            database: "unknown".into(),
            infra: vec![],
        },
        ..Default::default()
    }).await;

    assert!(results.is_ok());
    assert!(results.unwrap().patterns.is_empty());
}
```

---

### 3. Relationship Graph Tests

Assert that `requires` relationships always pull in the required pattern, even when the vector search scores it below the cutoff threshold. These tests specifically verify the graph safety net.

```rust
// tests/retrieval/relationship_graph.rs

#[tokio::test]
async fn required_pattern_included_even_when_vector_score_is_low() {
    let client = test_client().await;

    // This query is phrased to score "database-transaction-atomic" highly
    // but NOT to score "idempotency-key-http" highly through vector search alone.
    // The relationship graph must pull it in because transaction requires it.
    let results = client.get_patterns(PatternQuery {
        intent: "wrap database writes in a transaction for consistency".into(),
        stack: rust_axum_stack(),
        tags: vec!["transaction".into(), "acid".into()],
        context: None,
    }).await.unwrap();

    let ids: Vec<&str> = results.patterns.iter()
        .map(|p| p.id.as_str())
        .collect();

    assert!(ids.contains(&"database-transaction-atomic"),
        "transaction pattern must be in results");
    assert!(ids.contains(&"idempotency-key-http"),
        "idempotency pattern must be pulled in via requires relationship \
         even if not scored highly by vector search");

    // Verify the relationship_type is marked correctly
    let idempotency = results.patterns.iter()
        .find(|p| p.id == "idempotency-key-http")
        .unwrap();

    assert_eq!(idempotency.relationship_type, "required");
}
```

---

### 4. Anti-Pattern Detection Tests

Assert that `validate_intent` surfaces the correct warnings for known dangerous combinations.

```rust
// tests/tools/validate_intent.rs

#[tokio::test]
async fn email_in_transaction_triggers_anti_pattern_warning() {
    let client = test_client().await;

    let result = client.validate_intent(ValidateIntentQuery {
        description: "send welcome email and update user status in a transaction".into(),
        stack: rust_axum_stack(),
        proposed_approach: Some(
            "wrap email send and DB update in a single transaction block".into()
        ),
    }).await.unwrap();

    let anti_pattern_descriptions: Vec<&str> = result.anti_patterns.iter()
        .map(|a| a.description.as_str())
        .collect();

    assert!(
        anti_pattern_descriptions.iter().any(|d|
            d.contains("email") && d.contains("transaction")
        ),
        "should warn about sending email inside a transaction block"
    );
}
```

---

### 5. Regression Tests

Any time a retrieval bug is found — in development, testing, or production — a corresponding `(query, expected)` test is added to the eval set before the fix is merged. The rule is simple: **the bug cannot recur without breaking a test.**

Regression tests live in `tests/retrieval/regressions/` and are tagged with the issue number that surfaced the bug.

```rust
// tests/retrieval/regressions/issue_42.rs
// Issue #42: "create user account" query was not returning jwt-validation pattern

#[tokio::test]
async fn issue_42_create_user_returns_jwt_validation() {
    let client = test_client().await;

    let results = client.get_patterns(PatternQuery {
        intent: "create a new user account endpoint".into(),
        stack: rust_axum_stack(),
        ..Default::default()
    }).await.unwrap();

    let ids: Vec<&str> = results.patterns.iter()
        .map(|p| p.id.as_str())
        .collect();

    assert!(ids.contains(&"jwt-validation"),
        "regression: create user endpoint must surface jwt-validation pattern");
}
```

---

### 6. Performance Tests

Assert that latency targets are met under load. Run in CI before any release.

```rust
// tests/performance/latency.rs

#[tokio::test]
async fn get_patterns_p99_latency_under_50ms_warm_cache() {
    let client = test_client().await;

    // Warm the embedding cache
    client.get_patterns(payment_query()).await.unwrap();

    // Measure 100 repeated calls
    let mut latencies = Vec::with_capacity(100);
    for _ in 0..100 {
        let start = std::time::Instant::now();
        client.get_patterns(payment_query()).await.unwrap();
        latencies.push(start.elapsed().as_millis());
    }

    latencies.sort();
    let p99 = latencies[98]; // 99th percentile

    assert!(p99 < 50,
        "p99 latency was {}ms, must be under 50ms with warm cache", p99);
}
```

---

## Eval Set Management

The retrieval eval set lives in `tests/fixtures/eval_pairs.json`. Each entry is:

```json
{
  "id": "payment-handler-rust",
  "query": {
    "intent": "process a payment and create an order record",
    "stack": {
      "language": "rust",
      "framework": "axum",
      "database": "postgresql",
      "infra": ["supabase", "stripe"]
    },
    "tags": ["payment", "write", "multi-step"]
  },
  "expected_pattern_ids": [
    "database-transaction-atomic",
    "idempotency-key-http"
  ],
  "minimum_rank": 5
}
```

Run the full eval suite with:

```bash
make eval
```

Output shows recall@5 per query and aggregate recall across the full set. A build is considered passing when aggregate recall@5 ≥ 0.85.
