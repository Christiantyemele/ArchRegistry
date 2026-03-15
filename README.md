# ArchRegistry

> The architectural knowledge layer for AI-assisted software development.

ArchRegistry is an open-source MCP (Model Context Protocol) server that exposes a curated registry of production-grade architectural patterns to coding agents. Instead of letting AI generate structurally naive code, ArchRegistry gives agents — Cursor, Claude, Windsurf, and any MCP-compatible tool — the knowledge of a senior engineer before a single line is written.

---

## The Problem

Vibe coding has collapsed the barrier to starting an application. It has not collapsed the barrier to building one that survives production. Coding agents generate working code. They do not, by default, generate code that respects ACID transactions, multi-level caching strategies, idempotency guarantees, or the dozens of other architectural patterns that separate apps users keep from apps users abandon.

Most developers using AI tools today do not know what they do not know. ArchRegistry fixes the supply side of that problem — not by educating the developer, but by making the agent architecturally informed before it generates anything.

---

## How It Works

ArchRegistry is an MCP server. When an agent is about to generate code, it queries the registry with its intent, stack, and context. The registry performs semantic search over a curated pattern library and returns the most relevant architectural patterns — with stack-specific implementation guidance — before the agent writes a single line.

```
Developer prompt → Coding agent → ArchRegistry MCP server
                                          ↓
                              Semantic search over pattern registry
                              Relationship graph resolution
                              Stack-specific implementation retrieval
                                          ↓
                   Agent receives patterns → Generates governed code
```

The developer does not need to know what a database transaction is, what idempotency means, or why multi-level caching matters. The agent knows because ArchRegistry told it.

---

## Project Status

🔨 **Active development — pre-launch.**
Core team building the foundation. Public contributions open after Phase 3.

Star the repo to follow progress. If you want to be part of the founding contributor group, open an issue introducing yourself and the stack you know best.

---

## Documentation

| Document | Description |
|---|---|
| [Design](docs/design.md) | Architecture, retrieval pipeline, database schema, design decisions |
| [Implementation Plan](docs/implementation-plan.md) | Phased delivery plan with milestones and checkboxes |
| [MCP Tools](docs/mcp-tools.md) | All tools the server exposes, with full type signatures |
| [Pattern Spec](docs/pattern-spec.md) | How patterns are structured and how to author them |
| [Use Case Flows](docs/use-case-flows.md) | End-to-end flows showing the system in action |
| [Testing](docs/testing.md) | Testing strategy, test types, and example test code |
| [Installation](docs/installation.md) | Self-hosted setup guide |
| [Monetisation](docs/monetisation.md) | Business model and revenue strategy |

---

## Project Structure

```
archregistry/
├── README.md
├── CONTRIBUTING.md
├── registry/
│   └── patterns/              ← all patterns live here as Markdown
│       ├── database/
│       ├── caching/
│       ├── http/
│       ├── auth/
│       └── resilience/
├── server/                    ← Rust MCP server
│   ├── Cargo.toml
│   └── src/
│       ├── main.rs
│       ├── tools/
│       ├── retrieval/
│       ├── db/
│       └── models/
├── sync/                      ← CI pipeline: Markdown → DB
├── tests/
└── docs/
```

---

## Roadmap

- [ ] Rust + Axum stack (seed patterns)
- [ ] TypeScript + Next.js stack (seed patterns)
- [ ] Python + FastAPI stack (seed patterns)
- [ ] Go + Chi/Gin stack (community contribution)
- [ ] Private registry support (team tier)
- [ ] Pattern analytics dashboard
- [ ] VS Code extension for direct pattern browsing
- [ ] Verified publisher program for framework and library authors

---

## License

MIT — see `LICENSE`
