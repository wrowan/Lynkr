# Lynkr

Self-hosted LLM proxy server that adapts between AI coding tools (Claude Code CLI, Cursor, Continue.dev) and LLM providers (Databricks, AWS Bedrock, Azure, OpenRouter, Ollama, OpenAI, llama.cpp, LM Studio, Z.AI, Vertex AI). Converts between Anthropic/OpenAI API formats, handles smart routing, caching, and token optimization.

## Tech Stack

- **Runtime**: Node.js 20+ (pure JavaScript, no TypeScript)
- **Framework**: Express 5.1
- **Database**: SQLite via better-sqlite3 (WAL mode, FTS5 search) — `src/db/index.js:19-27`
- **HTTP Client**: undici + native fetch
- **Logging**: pino (structured JSON)
- **Code Parsing**: tree-sitter (JS, Python, TS)
- **Containerization**: Docker (Node 24 Alpine), docker-compose
- **Testing**: Node.js native test runner (`node --test`)
- **Linting**: ESLint (`eslint:recommended`)

## Project Structure

```
index.js              → Entry point, calls src/server.js:start()
bin/cli.js            → CLI entry point (`lynkr` command)
src/
  server.js           → Express app creation, middleware ordering, startup sequence
  config/             → Env-based config (index.js ~900 lines), hot reload watcher
  api/
    router.js         → Main route definitions
    openai-router.js  → OpenAI-format endpoints
    middleware/        → 11 middleware modules (load-shedding, session, budget, metrics, etc.)
    health.js         → /health/live, /health/ready
  orchestrator/       → Request→LLM→Tool→LLM agentic loop (max steps configurable)
  clients/
    databricks.js     → Primary provider router (all providers dispatch through here)
    *-utils.js        → Provider-specific adapters (bedrock, ollama, openrouter, openai-format)
    circuit-breaker.js→ Circuit breaker implementation
    retry.js          → Exponential backoff with jitter
  routing/            → Smart local/cloud routing based on complexity analysis
  tools/              → 20 tool implementations (workspace, git, web, execution, MCP, etc.)
    index.js          → Tool registry with alias resolution
    lazy-loader.js    → On-demand tool loading by keyword triggers
    smart-selection.js→ Request classification → minimal tool set
    truncate.js       → Per-tool output truncation strategies
  cache/              → Prompt cache (LRU+TTL) and semantic cache (embeddings similarity)
  memory/             → Titans-inspired long-term memory (store, extractor, retriever, surprise)
  mcp/                → Model Context Protocol client, registry, Docker sandboxing
  workers/            → Worker thread pool for CPU-intensive tasks (JSON parse, compression)
  headroom/           → Context compression sidecar client (Python Docker container)
  sessions/           → Session management and cleanup
  db/                 → SQLite schemas and prepared statements
  observability/      → Prometheus-compatible metrics collector
  policy/             → Step limits, tool restrictions, file access control
  context/            → Token budget tracking and conversation compression
  agents/             → Agent execution loop, reflection, skillbook
  logger/             → Pino logger setup, audit logging
  server/             → Graceful shutdown manager
test/                 → 27 test files using Node.js native test runner
documentation/        → 20 markdown guides (providers, features, API, deployment, etc.)
data/                 → Runtime SQLite databases (sessions.db, memories.db, workspace-index.db)
headroom-sidecar/     → Python compression service (Docker)
```

## Commands

```bash
npm start              # Start server (auto-starts Headroom Docker container)
npm run dev            # Dev mode with nodemon auto-reload
npm test               # Run unit + performance tests
npm run test:unit      # Unit tests only (19 test files)
npm run test:memory    # Memory system tests only
npm run test:quick     # Quick routing test only
npm run test:benchmark # Detailed performance benchmarks
npm run lint           # ESLint
```

All test scripts inject stub env vars (`DATABRICKS_API_KEY=test-key`) so they run without credentials.

## Configuration

All config is via `.env` (see `.env.example`, ~450 lines). Key variables:
- `MODEL_PROVIDER` — primary provider (ollama, databricks, azure-openai, openrouter, bedrock, etc.)
- `TOOL_EXECUTION_MODE` — server, client, or passthrough
- `MEMORY_ENABLED`, `HEADROOM_ENABLED`, `PREFER_OLLAMA`, `FALLBACK_ENABLED`

Config is loaded at `src/config/index.js` and supports hot reload via `src/config/watcher.js`.

## API Endpoints

- `POST /v1/messages` — Anthropic-format messages (main endpoint)
- `POST /v1/chat/completions` — OpenAI-format chat completions
- `GET /health/live`, `/health/ready` — Health checks
- `GET /metrics`, `/metrics/prometheus` — Observability
- `GET /routing/stats` — Routing decision metrics

## Databases

Three SQLite databases in `data/`:
- `sessions.db` — Session and conversation history (`src/db/index.js`)
- `memories.db` — Long-term memory with FTS5 (`src/db/memories.js`)
- `workspace-index.db` — Code indexing for workspace tools

## Additional Documentation

Check these files when working on relevant areas:

| Topic | File |
|-------|------|
| Architectural patterns & conventions | `.claude/docs/architectural_patterns.md` |
| Provider configuration | `documentation/providers.md` |
| Feature architecture & request flow | `documentation/features.md` |
| API reference | `documentation/api.md` |
| Memory system design | `documentation/memory-system.md` |
| Token optimization strategies | `documentation/token-optimization.md` |
| Context compression (Headroom) | `documentation/headroom.md` |
| Tool calling & execution modes | `documentation/tools.md` |
| Production deployment & hardening | `documentation/production.md` |
| Docker & container setup | `documentation/docker.md` |
| Testing guide | `documentation/testing.md` |
| Troubleshooting | `documentation/troubleshooting.md` |
| Cursor IDE integration | `documentation/cursor-integration.md` |
| Claude Code CLI integration | `documentation/claude-code-cli.md` |
