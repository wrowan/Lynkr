# Architectural Patterns & Conventions

Recurring patterns observed across the Lynkr codebase. Reference this when adding new features to maintain consistency.

## 1. Middleware Pipeline

Express middleware is ordered deliberately in `src/server.js:65-163`. The pipeline is:

```
Load Shedding → Request Logging → Metrics → Compression → JSON Parse → Session → Logging → Budget → Routes → 404 → Error Handler
```

Each middleware is a separate module in `src/api/middleware/`. When adding new middleware, place it in the correct position relative to others (e.g., load shedding must be first to reject early).

Key files: `src/api/middleware/load-shedding.js`, `src/api/middleware/session.js`, `src/api/middleware/budget.js`, `src/api/middleware/metrics.js`

## 2. Provider Adapter Pattern

All LLM providers are normalized through adapter modules so the orchestrator uses a single interface. Each provider has a `*-utils.js` file that converts between the provider's native format and the internal Anthropic-style format.

- `src/clients/bedrock-utils.js` — AWS Bedrock adapter
- `src/clients/ollama-utils.js` — Ollama adapter
- `src/clients/openrouter-utils.js` — OpenRouter adapter
- `src/clients/openai-format.js` — OpenAI format conversion
- `src/clients/responses-format.js` — Response normalization

All providers dispatch through `src/clients/databricks.js:invokeModel()`, which selects the correct adapter based on `config.modelProvider.type`. To add a new provider: create a `*-utils.js` adapter, add a case in the `invokeModel` dispatch, and add config entries in `src/config/index.js`.

## 3. Circuit Breaker

`src/clients/circuit-breaker.js:23-47` implements the classic three-state pattern (CLOSED → OPEN → HALF_OPEN). Each provider gets its own breaker instance via `CircuitBreakerRegistry`. The breaker wraps provider calls in `execute(fn)` — on repeated failures it opens and fails fast, then periodically allows test requests.

Configuration: `failureThreshold` (default 5), `timeout` (60s), `successThreshold` (2 to close from half-open).

Used alongside retry logic at `src/clients/retry.js:6-14` (exponential backoff with jitter, retryable status codes: 429, 500, 502, 503, 504).

## 4. Registry + Alias Resolution

Tools are stored in a `Map` registry at `src/tools/index.js:4`. A large alias map (`src/tools/index.js:18-74`) normalizes tool names (e.g., `bash` → `shell`, `grep` → `workspace_search`). Tools are registered via `registerTool(name, definition)` and looked up case-insensitively via `registryLowercase`.

The MCP server registry at `src/mcp/registry.js:7-8` follows the same `Map`-based pattern with `registerServer()`, `getServer()`, `listServers()`.

## 5. Lazy Loading

`src/tools/lazy-loader.js:19-75` defines tool categories with keyword triggers and priority levels (0=core, 3=optional). Core tools (stubs, workspace, execution) load at startup. Others load on-demand when prompt content matches their keywords via `ensureToolsForPrompt(promptText)`.

Controlled by `LAZY_TOOLS_ENABLED` env var (default: true). The toggle is in `src/server.js:43-63`.

## 6. Singleton via Module-Level Initialization

Stateful services use a module-scoped instance with a getter function pattern:

```
let instance = null;
function getXxx(options) {
  if (!instance) instance = new Xxx(options);
  return instance;
}
```

Examples:
- `src/clients/circuit-breaker.js` — `getCircuitBreakerRegistry()`
- `src/observability/metrics.js` — `getMetricsCollector()`
- `src/server/shutdown.js` — `getShutdownManager()`
- `src/cache/semantic.js` — `getSemanticCache()`
- `src/workers/pool.js` — `getWorkerPool()`

## 7. Orchestrator Loop (Agent Pattern)

`src/orchestrator/index.js` implements the core request→LLM→tool→LLM loop. Each turn:
1. Build system prompt + tools + messages
2. Invoke LLM (via `invokeModel`)
3. If response contains tool_use blocks, execute them
4. Append results and loop (up to `policy.maxStepsPerTurn`, default 8)
5. Return final assistant response

Policy enforcement happens at each step via `src/policy/index.js`.

## 8. Config from Environment

All configuration derives from `.env` via `dotenv`, parsed in `src/config/index.js`. Helper functions handle type conversion: `parseJson()`, `parseList()`, `parseMountList()` (lines 11-48). Config supports hot reload — `src/config/watcher.js` watches `.env` and calls `config.reloadConfig()` on change with debouncing.

Never read `process.env` directly outside of `src/config/index.js`. Always access config via `require('./config')`.

## 9. SQLite with Prepared Statements

Database access uses `better-sqlite3` with prepared statements defined at module scope. See `src/memory/store.js:5-50` for the pattern — statements are prepared once at import time and reused.

SQLite is tuned for performance at `src/db/index.js:19-27`: WAL mode, 64MB cache, memory-mapped I/O, NORMAL synchronous mode.

## 10. Worker Thread Pool

`src/workers/pool.js:15-37` manages a pool of worker threads (auto-sized to CPU count - 1). CPU-intensive tasks (JSON parsing, object cloning, compression) are offloaded when payload exceeds `offloadThreshold` (default 10KB).

Helper functions at `src/workers/helpers.js` provide `asyncClone()`, `asyncTransform()` that transparently fall back to sync processing if the pool isn't available.

## 11. Smart Tool Selection

`src/tools/smart-selection.js` classifies requests (conversational, simple_qa, research, file_reading, file_modification, coding, etc.) using pre-compiled regex patterns (lines 13-22), then maps each type to a minimal tool set (lines 27-50+). This reduces tool token overhead by 50-70% for non-coding queries.

## 12. Output Truncation (Decorator Pattern)

`src/tools/truncate.js:4-16` defines per-tool truncation limits and strategies (head, tail, middle). The `truncateToolOutput(toolName, output)` function wraps tool results, keeping output within token budgets without modifying tool implementations.

## 13. Graceful Shutdown

`src/server/shutdown.js:16-23` implements ordered shutdown: stop accepting connections → drain active requests → run registered shutdown callbacks → force-exit after timeout (30s). Services register via `shutdownManager.onShutdown(callback)` in `src/server.js:211-254`.

## 14. Concurrency Control (Semaphore)

`src/clients/databricks.js:26-60` implements a Semaphore class to limit concurrent requests per provider. Used to prevent rate-limiting from parallel CLI calls. Pattern: `await semaphore.acquire()` → do work → `semaphore.release()` in a `finally` block.

## 15. Health Check Convention

Two-tier health checks at `src/api/health.js`:
- `/health/live` — Is the process running? (always 200 unless shutting down)
- `/health/ready` — Can it serve requests? (checks dependencies)

This follows the Kubernetes liveness/readiness probe convention.

## 16. Metrics Collection

`src/observability/metrics.js:14-50` uses in-memory counters with no I/O overhead. Supports JSON snapshots (`getMetrics()`) and Prometheus text format (`toPrometheus()`). Metrics are collected via middleware at `src/api/middleware/metrics.js`.

Separate subsystem metrics are exposed at dedicated endpoints: `/metrics/circuit-breakers`, `/metrics/load-shedding`, `/metrics/worker-pool`, `/metrics/semantic-cache`, `/metrics/lazy-tools` — all defined in `src/server.js:107-155`.
