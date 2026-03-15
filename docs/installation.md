# Installation Guide

This guide walks through setting up ArchRegistry locally for development and self-hosting in production. The entire stack runs on free-tier infrastructure during development.

---

## Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| Rust | 1.75+ | MCP server |
| Docker + Docker Compose | Latest | Local Supabase + Ollama |
| Supabase CLI | Latest | Database migrations |
| `make` | Any | Dev commands |

---

## Quick Start (Docker Compose)

The fastest path to a running local instance. Docker Compose starts Supabase locally, Ollama for local embeddings, and the MCP server together.

```bash
# 1. Clone the repository
git clone https://github.com/your-org/archregistry
cd archregistry

# 2. Copy environment config
cp .env.example .env

# 3. Start all services
docker compose up -d

# 4. Run database migrations
make migrate

# 5. Seed the registry with all patterns
make seed

# 6. Verify the server is running
curl http://localhost:3000/health
# → {"status": "ok", "patterns_loaded": 100}
```

The MCP server is now available at `http://localhost:3000/sse`.

---

## Manual Setup

Use this path if you want to connect to an existing Supabase project rather than running Supabase locally.

### 1. Clone the repository

```bash
git clone https://github.com/your-org/archregistry
cd archregistry
```

### 2. Configure environment

```bash
cp .env.example .env
```

Edit `.env`:

```env
# Database — your Supabase connection string
DATABASE_URL=postgresql://postgres:[password]@[host]:5432/postgres

# Embedding provider
# "local" uses Ollama (free, ~5-10ms latency)
# "openai" uses OpenAI API (requires key, ~50-100ms latency)
EMBEDDING_PROVIDER=local

# Only required if EMBEDDING_PROVIDER=openai
OPENAI_API_KEY=

# Ollama endpoint — only required if EMBEDDING_PROVIDER=local
LOCAL_EMBED_MODEL_URL=http://localhost:11434/api/embeddings
LOCAL_EMBED_MODEL_NAME=nomic-embed-text

# MCP server
MCP_SERVER_PORT=3000

# Embedding cache TTL in seconds (3600 = 1 hour)
EMBEDDING_CACHE_TTL_SECONDS=3600

# Log level: "debug" | "info" | "warn" | "error"
LOG_LEVEL=info
```

### 3. Set up the local embedding model (if using local)

```bash
# Install Ollama
curl -fsSL https://ollama.ai/install.sh | sh

# Pull the embedding model
ollama pull nomic-embed-text

# Verify it works
curl http://localhost:11434/api/embeddings \
  -d '{"model": "nomic-embed-text", "prompt": "test"}'
```

### 4. Run database migrations

```bash
# Install Supabase CLI if you haven't
brew install supabase/tap/supabase  # macOS
# or: npm install -g supabase

# Run migrations against your database
make migrate

# Verify migrations ran
make db-status
```

Migrations are in `server/migrations/` and run in order. The key migration enables pgvector and creates the `patterns` and `pattern_relationships` tables.

### 5. Seed the registry

The seed command parses all Markdown files in `registry/patterns/`, generates embeddings, and upserts them into the database.

```bash
make seed

# Output:
# Parsing patterns...     100 patterns found
# Generating embeddings... 100/100
# Upserting to database... done
# Building relationship graph... done
# Seed complete. 100 patterns loaded.
```

### 6. Start the MCP server

```bash
# Development (with hot reload)
cargo watch -x run

# Production
cargo run --release

# Output:
# INFO archregistry: starting MCP server on port 3000
# INFO archregistry: embedding provider: local (nomic-embed-text)
# INFO archregistry: patterns loaded: 100
# INFO archregistry: server ready at http://0.0.0.0:3000/sse
```

---

## Connecting Your Coding Agent

### Cursor

Add to `.cursor/mcp.json` in your project root (or `~/.cursor/mcp.json` for global):

```json
{
  "mcpServers": {
    "archregistry": {
      "url": "http://localhost:3000/sse"
    }
  }
}
```

Restart Cursor. The ArchRegistry tools will appear in the MCP tools panel.

### Claude Desktop

Add to `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or the equivalent path on your OS:

```json
{
  "mcpServers": {
    "archregistry": {
      "url": "http://localhost:3000/sse"
    }
  }
}
```

Restart Claude Desktop.

### Windsurf

Add to Windsurf's MCP configuration (Settings → MCP Servers):

```
URL: http://localhost:3000/sse
Name: archregistry
```

---

## Verify Everything Works

Run the built-in verification command after setup:

```bash
make verify

# Runs:
# ✓ Database connection
# ✓ pgvector extension enabled
# ✓ Patterns table populated (100 patterns)
# ✓ Embedding model reachable
# ✓ Vector similarity search working
# ✓ get_patterns tool end-to-end (payment handler query)
# ✓ MCP SSE endpoint reachable
# All checks passed.
```

---

## Dev Commands Reference

```bash
make migrate       # Run pending database migrations
make seed          # Parse all patterns and sync to database
make reseed        # Drop all patterns and re-seed from scratch
make eval          # Run the full retrieval quality test suite
make test          # Run all tests (unit + contract + regression)
make verify        # End-to-end health check
make db-status     # Show migration status
make db-reset      # Drop and recreate the database (dev only)
```

---

## Production Deployment

For production self-hosting, the MCP server is a single Rust binary. No special runtime required.

```bash
# Build the release binary
cargo build --release

# The binary is at
./target/release/archregistry

# Run with environment variables
DATABASE_URL=... \
EMBEDDING_PROVIDER=local \
LOCAL_EMBED_MODEL_URL=http://your-ollama-host:11434/api/embeddings \
MCP_SERVER_PORT=3000 \
./target/release/archregistry
```

The server exposes one endpoint: `GET /sse` for the MCP SSE transport, and `GET /health` for health checks.

For container deployments, a `Dockerfile` is provided in the repository root. The image is minimal — a distroless Rust base with the compiled binary only.

---

## Troubleshooting

**`pgvector extension not found`**
Run `make migrate` — the first migration enables the extension. On Supabase, pgvector is available on all plans and just needs to be enabled via migration.

**`embedding model not reachable`**
Verify Ollama is running: `ollama list`. If using OpenAI, verify your `OPENAI_API_KEY` is set and valid.

**`0 patterns loaded after seed`**
Check that `registry/patterns/` exists and contains `.md` files. Run `make seed` with `LOG_LEVEL=debug` for detailed output showing which files are being parsed and why any might be failing validation.

**`MCP server not appearing in Cursor`**
Confirm the server is running (`curl http://localhost:3000/health`), then restart Cursor fully (not just reload window). Check the MCP config file is valid JSON.
