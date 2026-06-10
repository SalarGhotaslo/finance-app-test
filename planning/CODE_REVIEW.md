# FinAlly — Code Review

<!-- cSpell:words openrouter watchlist asyncio contextvars Cerebras OPENROUTER sparkline sparklines pytest httpx aiohttp TOCTOU behaviour -->

> Reviewed: 2026-06-09
> Scope: planning/PLAN.md (no implementation code exists yet — this is a specification review)

---

## CRITICAL: Secret Exposed in `.env`

The file `/finance_project/.env` contains a live OpenRouter API key (`sk-or-v1-...`). You should **rotate the key immediately** at openrouter.ai, and confirm `.env` is listed in `.gitignore` before any git operations are performed on this repo.

---

## Section-by-Section Findings

### Section 3 — Architecture

**Issue: SQLite write concurrency under SSE + background tasks**
The plan uses SQLite with multiple concurrent writers: the HTTP request handler (trades, watchlist), a portfolio-snapshot background task (every 30 seconds), the market-data background task (cache updates), and SSE broadcast readers. SQLite in WAL mode handles concurrent reads well, but concurrent writes from background tasks and request handlers will queue on a single write lock. The plan does not mention enabling WAL mode (`PRAGMA journal_mode=WAL`) or setting a `busy_timeout`. Without these, background tasks and simultaneous trade requests will raise `OperationalError: database is locked` under any modest load. Agents implementing the backend must enable WAL mode at startup.

**Issue: In-memory price cache is not thread-safe**
The plan describes a shared in-memory price cache written by one background task and read by SSE broadcast loops and HTTP handlers. If the backend mixes `threading` background tasks with `asyncio` handlers, there is a race condition. The plan should explicitly state whether background tasks must be `asyncio` tasks (using `asyncio.create_task` or FastAPI lifespan events) or threads, and mandate a `asyncio.Lock` if threads are used.

**Issue: `request_id` via `contextvars.ContextVar` and `asyncio`**
The plan (Section 13.2) proposes using `contextvars.ContextVar` for `request_id` propagation. If any code uses `asyncio.create_task`, the child task gets a *copy* of the context at the time of creation, not a live reference. Trades spawned as sub-tasks from a chat handler would lose the original `request_id` unless explicitly passed. Implementing agents should pass `request_id` explicitly to spawned tasks.

---

### Section 5 — Environment Variables

**Issue: `LLM_MODEL` default references a model not available on Cerebras**
The plan says the default `LLM_MODEL` is `openrouter/openai/gpt-oss-120b`. Section 9 says to use Cerebras as the inference provider. These are in tension: Cerebras hosts its own models (e.g., `llama-4-scout-17b-16e-instruct`, `llama3.1-8b`), not OpenAI's models. The default model should be one Cerebras actually serves.

**Issue: No validation of `OPENROUTER_API_KEY` at startup**
The plan does not require the backend to validate that `OPENROUTER_API_KEY` is present and non-empty at startup. A user who omits the key will get a cryptic LiteLLM error only when they first send a chat message. The plan should require a startup check that logs a clear warning (or raises) if the key is absent.

---

### Section 6 — Market Data

**Issue: GBM simulator can produce negative or near-zero prices**
The plan's mention of "sudden 2-5% moves" as additive events could push a price negative if implemented naively as `price += price * shock` vs `price *= (1 + shock)`. The plan should require multiplicative application of all shocks to guarantee prices remain positive, and enforce a minimum floor (e.g., $0.01).

**Issue: Day-open price reset at UTC midnight is ambiguous**
US markets trade in Eastern Time (ET). If a user runs the demo between midnight UTC and market open (e.g., 4am–9:30am ET), the day open resets and the displayed daily change % becomes meaningless. The plan should acknowledge the timezone choice.

**Issue: Massive API polling interval not configurable**
The polling interval is not an environment variable. If a user on a paid tier wants faster updates, there is no way to configure it without code changes. A `MASSIVE_POLL_INTERVAL_SECONDS` env var should be added, or the plan should explicitly note the interval is hardcoded.

---

### Section 7 — Database

**Issue: No index on `trades` for audit/reconstruction queries**
The `trades` table has no index on `user_id` or `executed_at`. Portfolio reconstruction (Section 13.6) would require a full table scan. Consider adding `CREATE INDEX idx_trades_user_executed ON trades(user_id, executed_at)`.

**Issue: Floating-point comparison for sell validation**
The plan says "selling more than owned" is a validation failure, without specifying how floating-point comparison is done. `0.1 + 0.2 == 0.3` is `False` in Python. Agents should use a small epsilon or `round()` for quantity comparisons, not raw `>=`.

**Issue: `portfolio_snapshots` has no purge policy**
Snapshots are recorded every 30 seconds indefinitely. The plan provides no cleanup guidance. For completeness, the plan should note this is acceptable for the demo scope.

**Issue: `chat_messages.actions` stored as JSON text**
Storing `actions` as a `TEXT` column containing JSON means no SQL filtering on it is possible without application-level parsing or SQLite JSON functions. The plan should clarify that this column is write-once and only read back as a blob.

---

### Section 8 — API Endpoints

**Issue: `GET /api/watchlist` response omits `day_open_price`**
The SSE stream sends `day_open_price` but the REST response does not include it. On initial page load, before any SSE events arrive, the frontend cannot compute daily change % without it. The `GET /api/watchlist` response shape should include `day_open_price`.

**Issue: `POST /api/watchlist` does not specify ticker validation rules**
What makes a ticker *valid*? The plan does not define: maximum ticker length, allowed characters (e.g., only `[A-Z.]{1,10}`), or whether the backend checks if the ticker exists in the price cache. In Massive API mode, an unknown ticker would never receive a price. Ticker validation rules should be defined.

**Issue: `DELETE /api/watchlist/{ticker}` — no guidance on positions**
If the user holds a position in a deleted ticker, the ticker would disappear from the watchlist but the position would remain with no price feed visible in the UI. The plan should clarify: either (a) prevent deletion if a position exists, or (b) allow deletion and note positions persist independently.

**Issue: `GET /api/portfolio/history` has no pagination**
In a long-running session this could return thousands of rows. The plan should either add a `?limit=N` query parameter, or document this as an accepted limitation.

**Issue: Trade response does not include `executed_at` timestamp**
The `POST /api/portfolio/trade` response omits `executed_at`. Without it, the frontend cannot display when the trade occurred.

---

### Section 9 — LLM Integration

**Issue: Partial-failure UX gap in auto-execution**
If the LLM says "I'll buy 5 AAPL and 10 GOOGL" but cash is only sufficient for AAPL, the backend executes AAPL (status: executed), GOOGL fails (status: failed), and returns the LLM's original message unchanged — which says both trades happened. The plan should either require server-side annotation of the LLM message when trades fail, or recommend the frontend prominently display failed trade statuses.

**Issue: No rate limiting on `POST /api/chat`**
LLM calls cost money. The plan has no mention of rate limiting. At minimum, the plan should acknowledge the cost risk and suggest `LLM_MOCK=true` for sustained development.

**Issue: Context window not bounded for large portfolios**
For a user with 50 watchlist items and a long conversation, the prompt context could grow large. The plan should recommend a rough token budget check or cap on positions/watchlist items sent in context.

**Issue: `LLM_MOCK=true` mock response is not defined**
The plan says mock mode returns "deterministic mock responses" but does not specify what those responses are. The plan should define at minimum one canonical mock response (e.g., `{"message": "Mock response", "trades": [], "watchlist_changes": []}`) so all agents implement the same mock.

---

### Section 10 — Frontend Design

**Issue: Sparklines are empty on fresh page load**
Sparklines are built entirely from SSE events since page load. On first load every sparkline is empty. The plan should acknowledge the "progressive fill" UX as intentional.

**Issue: Price flash CSS lifecycle unspecified for rapid updates**
At 500ms SSE ticks, two back-to-back updates will fire a second flash before the first 500ms transition completes. The plan should specify: either reset the animation timer on each new price (`animation: none` then re-apply), or debounce the flash.

---

### Section 11 — Docker & Deployment

**Issue: Static files path inside container not specified**
The plan says "Copy frontend build output into a static/ directory" but does not specify the exact container path. FastAPI's `StaticFiles` mount needs to know this path. The plan should pin it (e.g., `/app/static/`).

**Issue: Event log not volume-mounted**
Events are stored at `/backend/logs/events.jsonl` inside the container but this path is not volume-mounted. Container restarts lose all event logs, contradicting the plan's emphasis on logs as "source of truth." Either volume-mount the log directory alongside the database, or explicitly acknowledge logs are ephemeral.

**Issue: Relationship between `docker-compose.yml` and `docker-compose.test.yml` is unclear**
If agents treat `docker-compose.yml` as optional and skip it, the test infrastructure may not have a reference base to derive from. The plan should clarify the relationship.

---

### Section 12 — Testing Strategy

**Issue: SSE integration test guidance missing**
Testing SSE in a pytest integration test requires consuming an async generator. This is non-trivial and the plan provides no guidance. Agents should be pointed to `httpx-sse` or `aiohttp` for consuming SSE in tests.

**Issue: No test coverage for anomaly detection rules**
The testing strategy covers unit, component, integration, E2E, accessibility, and security tests, but does not mention testing the anomaly detection rules from Section 13.4. These are deterministic and easily unit-testable.

**Issue: E2E test "P&L chart has data points after 30 seconds" is slow**
This test literally waits 30 seconds. The plan should either expose a test-only endpoint to force a snapshot, or add a `SNAPSHOT_INTERVAL_SECONDS` env var that tests can set to 1.

---

### Section 13 — Event Logging

**Issue: `backend/logs/` missing from directory structure**
Section 4 defines the directory structure but `backend/logs/` does not appear. Agents may not create it. It should be added to Section 4.

**Issue: Log rotation has a TOCTOU race**
If two concurrent requests both check the file size and both find it >50 MB simultaneously, both will attempt to rename. The second rename would overwrite `events.jsonl.old`. Rotation should be performed by a dedicated background task on a schedule, not inline per-event.

**Issue: `sse_broadcast` events will fill the log in minutes**
The simulator broadcasts every 500ms. If a `sse_broadcast` event is logged on each tick, thousands of log entries per second will hit the 50 MB rotation limit very quickly. SSE broadcast logging should be sampled/throttled (e.g., log once per 10 seconds) or only log errors/anomalies.

---

### Section 14 — Decisions & Notes

**Issue: Client-side 500-point slice is inefficient**
`GET /api/portfolio/history` returns all snapshots and the frontend slices to 500. A user running the app overnight would have thousands of DB rows fetched over the network only to discard most of them. A `?limit=500` or `?since=<timestamp>` parameter should be added to the endpoint.

---

## Summary of Priority Issues

### Must address before implementation begins

1. **Rotate the live API key** in `/finance_project/.env`.
2. **Fix the Cerebras/OpenRouter model name conflict** — default `LLM_MODEL` must be a model Cerebras actually serves.
3. **Add `day_open_price` to `GET /api/watchlist` response** so daily % change is computable on initial load.
4. **Define ticker validation rules** for `POST /api/watchlist` and behavior when a ticker has no price data.
5. **Require SQLite WAL mode + `busy_timeout`** at backend startup to prevent `database is locked` errors.
6. **Define the canonical `LLM_MOCK=true` response** so all agents implement the same mock.

### Should address before agents begin coding

7. Specify background tasks must be `asyncio` tasks (not threads) to avoid cache race conditions, or mandate a lock.
8. Clarify `DELETE /api/watchlist/{ticker}` behaviour when a position exists in that ticker.
9. Add `SNAPSHOT_INTERVAL_SECONDS` env var or a test-trigger endpoint to make the 30-second E2E test feasible in CI.
10. Volume-mount the event log directory alongside the database, or acknowledge logs are ephemeral across restarts.
11. Throttle or sample `sse_broadcast` log events to prevent rapid log rotation.
12. Replace the client-side 500-point slice with a server-side `?limit=` parameter on `GET /api/portfolio/history`.
