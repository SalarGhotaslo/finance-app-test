# FinAlly — AI Trading Workstation

## Project Specification

## 1. Vision

FinAlly (Finance Ally) is a visually stunning AI-powered trading workstation that streams live market data, lets users trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades on the user's behalf. It looks and feels like a modern Bloomberg terminal with an AI copilot.

This is the capstone project for an agentic AI coding course. It is built entirely by Coding Agents demonstrating how orchestrated AI agents can produce a production-quality full-stack application. Agents interact through files in `planning/`.

## 2. User Experience

### First Launch

The user runs a single Docker command (or a provided start script). A browser opens to `http://localhost:8000`. No login, no signup. They immediately see:

- A watchlist of 10 default tickers with live-updating prices in a grid
- $10,000 in virtual cash
- A dark, data-rich trading terminal aesthetic
- An AI chat panel ready to assist

### What the User Can Do

- **Watch prices stream** — prices flash green (uptick) or red (downtick) with subtle CSS animations that fade
- **View sparkline mini-charts** — price action beside each ticker in the watchlist, accumulated on the frontend from the SSE stream since page load (sparklines fill in progressively)
- **Click a ticker** to see a larger detailed chart in the main chart area
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog
- **Monitor their portfolio** — a heatmap (treemap) showing positions sized by weight and colored by P&L, plus a P&L chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change
- **Chat with the AI assistant** — ask about their portfolio, get analysis, and have the AI execute trades and manage the watchlist through natural language
- **Manage the watchlist** — add/remove tickers manually or via the AI chat

### Visual Design

- **Dark theme**: backgrounds around `#0d1117` or `#1a1a2e`, muted gray borders, no pure black
- **Price flash animations**: brief green/red background highlight on price change, fading over ~500ms via CSS transitions
- **Connection status indicator**: a small colored dot (green = connected, yellow = reconnecting, red = disconnected) visible in the header
- **Professional, data-dense layout**: inspired by Bloomberg/trading terminals — every pixel earns its place
- **Responsive but desktop-first**: optimized for wide screens, functional on tablet

### Color Scheme
- Accent Yellow: `#ecad0a`
- Blue Primary: `#209dd7`
- Purple Secondary: `#753991` (submit buttons)

## 3. Architecture Overview

### Single Container, Single Port

```
┌─────────────────────────────────────────────────┐
│  Docker Container (port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST endpoints             │
│  ├── /api/stream/*   SSE streaming              │
│  └── /*              Static file serving         │
│                      (Next.js export)            │
│                                                 │
│  SQLite database (volume-mounted)               │
│  Background task: market data polling/sim        │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file at `db/finally.db`, volume-mounted for persistence
- **Real-time data**: Server-Sent Events (SSE) — simpler than WebSockets, one-way server→client push, works everywhere
- **AI integration**: LiteLLM → OpenRouter (Cerebras for fast inference), with structured outputs for trade execution
- **Market data**: Environment-variable driven — simulator by default, real data via Massive API if key provided

### Concurrency Model

All background tasks (market data simulator, Massive API poller, portfolio snapshot recorder, SSE broadcast loop) **must** be implemented as `asyncio` tasks registered via FastAPI's lifespan context manager (`asyncio.create_task`), not `threading.Thread`. This ensures the shared in-memory price cache is only accessed from within the single asyncio event loop, avoiding race conditions without requiring explicit locks.

`request_id` propagation note: `contextvars.ContextVar` values are copied into child tasks at creation time. Any `asyncio.create_task` call within a request handler must explicitly pass the current `request_id` into the spawned task as a parameter — it cannot rely on implicit context inheritance.

### Why These Choices

| Decision | Rationale |
|---|---|
| SSE over WebSockets | One-way push is all we need; simpler, no bidirectional complexity, universal browser support |
| Static Next.js export | Single origin, no CORS issues, one port, one container, simple deployment |
| SQLite over Postgres | No auth = no multi-user = no need for a database server; self-contained, zero config |
| Single Docker container | Students run one command; no docker-compose for production, no service orchestration |
| uv for Python | Fast, modern Python project management; reproducible lockfile; what students should learn |
| Market orders only | Eliminates order book, limit order logic, partial fills — dramatically simpler portfolio math |

---

## 4. Directory Structure

```
finally/
├── frontend/                 # Next.js TypeScript project (static export)
├── backend/                  # FastAPI uv project (Python)
│   ├── schema/               # Schema definitions, seed data, migration logic
│   └── logs/                 # Event log output (volume-mounted at runtime)
│       └── .gitkeep          # Directory exists in repo; events.jsonl is gitignored
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document
│   └── ...                   # Additional agent reference docs
├── scripts/
│   ├── start_mac.sh          # Launch Docker container (macOS/Linux)
│   ├── stop_mac.sh           # Stop Docker container (macOS/Linux)
│   ├── start_windows.ps1     # Launch Docker container (Windows PowerShell)
│   └── stop_windows.ps1      # Stop Docker container (Windows PowerShell)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # Volume mount target (SQLite file lives here at runtime)
│   └── .gitkeep              # Directory exists in repo; finally.db is gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── docker-compose.yml        # Optional convenience wrapper
├── .env                      # Environment variables (gitignored)
├── .env.example              # Committed template with all required variable names
└── .gitignore
```

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents.
- **`backend/schema/`** contains schema SQL definitions and seed logic. The backend initializes the database on startup — creating tables and seeding default data if the SQLite file doesn't exist or is empty.
- **`db/`** at the top level is the runtime volume mount point. The SQLite file (`db/finally.db`) is created here by the backend and persists across container restarts via Docker volume.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains start/stop scripts that wrap Docker commands.

---

## 5. Environment Variables

```bash
# Required: OpenRouter API key for LLM chat functionality
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: Massive (Polygon.io) API key for real market data
# If not set, the built-in market simulator is used (recommended for most users)
MASSIVE_API_KEY=

# Optional: Override the LLM model used for chat (default shown below)
# Must be a model served by Cerebras on OpenRouter (e.g. llama-4-scout-17b-16e-instruct)
LLM_MODEL=openrouter/cerebras/llama-4-scout-17b-16e-instruct

# Optional: Set to "true" for deterministic mock LLM responses (testing)
LLM_MOCK=false

# Optional: Snapshot recording interval in seconds (default 30; set to 1 in tests)
SNAPSHOT_INTERVAL_SECONDS=30
```

### Behavior

- If `MASSIVE_API_KEY` is set and non-empty → backend uses Massive REST API for market data
- If `MASSIVE_API_KEY` is absent or empty → backend uses the built-in market simulator
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (for E2E tests)
- `SNAPSHOT_INTERVAL_SECONDS` controls how often portfolio snapshots are recorded (default 30; set to 1 in tests to avoid 30-second waits)
- The backend reads `.env` from the project root (mounted into the container or read via docker `--env-file`)
- The backend must validate that `OPENROUTER_API_KEY` is present and non-empty at startup, logging a clear error and refusing to start if it is absent

---

## 6. Market Data

### Two Implementations, One Interface

Both the simulator and the Massive client implement the same abstract interface. The backend selects which to use based on the environment variable. All downstream code (SSE streaming, price cache, frontend) is agnostic to the source.

### Simulator (Default)

- Generates prices using geometric Brownian motion (GBM) with configurable drift and volatility per ticker
- Updates at ~500ms intervals
- Correlated moves across tickers (e.g., tech stocks move together)
- Occasional random "events" — sudden 2-5% moves on a ticker for drama. All shocks must be applied multiplicatively (`price *= (1 + shock)`) to guarantee prices stay positive. A minimum floor of $0.01 must be enforced as a hard guard.
- Starts from realistic seed prices (e.g., AAPL ~$190, GOOGL ~$175, etc.) — these seed prices serve as the initial day open price
- Resets the day open price at UTC midnight to support multi-day sessions
- Runs as an in-process background task — no external dependencies

### Massive API (Optional)

- REST API polling (not WebSocket) — simpler, works on all tiers
- Polls for the union of all watched tickers on a configurable interval
- Free tier (5 calls/min): poll every 15 seconds
- Paid tiers: poll every 2-15 seconds depending on tier
- Parses REST response into the same format as the simulator

### Shared Price Cache

- A single background task (simulator or Massive poller) writes to an in-memory price cache
- The cache holds the latest price, previous price, day open price (first price of the current calendar day), and timestamp for each ticker
- SSE streams read from this cache and push updates to connected clients
- This architecture supports future multi-user scenarios without changes to the data layer

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- Server pushes price updates for all tickers known to the system at a regular cadence (~500ms) — in the single-user model this is equivalent to the user's watchlist
- Each SSE event contains ticker, price, previous price, day open price, timestamp, and change direction
- Client handles reconnection automatically (EventSource has built-in retry)
- Adding a ticker to the watchlist also adds it to the price cache; it appears in the SSE stream within the next broadcast cycle (~500ms) — no client reconnect is needed
- The server broadcasts on every tick regardless of whether a price changed

---

## 7. Database

### SQLite with Startup Initialization

The backend initializes the SQLite database on application startup. If the file doesn't exist or tables are missing, it creates the schema and seeds default data before accepting requests. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

**Required startup configuration**: immediately after opening the connection, the backend must run:
```sql
PRAGMA journal_mode=WAL;
PRAGMA busy_timeout=5000;
```
WAL mode allows concurrent reads alongside writes. `busy_timeout` prevents `OperationalError: database is locked` when the background snapshot task and an HTTP trade request write simultaneously.

### Schema

All tables include a `user_id` column defaulting to `"default"`. This is hardcoded for now (single-user) but enables future multi-user support without schema migration.

**users_profile** — User state (cash balance)
- `id` TEXT PRIMARY KEY (default: `"default"`)
- `cash_balance` REAL (default: `10000.0`)
- `created_at` TEXT (ISO timestamp)

**watchlist** — Tickers the user is watching
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `added_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**positions** — Current holdings (one row per ticker per user)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `quantity` REAL (fractional shares supported)
- `avg_cost` REAL
- `updated_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

Realized P&L is **out of scope** — only unrealized P&L is tracked and displayed. The `trades` table is an append-only audit log and can reconstruct realized P&L if needed in a future version.

**trades** — Trade history (append-only log)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `side` TEXT (`"buy"` or `"sell"`)
- `quantity` REAL (fractional shares supported)
- `price` REAL
- `executed_at` TEXT (ISO timestamp)
- Index: `CREATE INDEX idx_trades_user_executed ON trades(user_id, executed_at)` — required for efficient audit/reconstruction queries

**Quantity comparison rule**: all sell-validation comparisons against owned quantity must use a small epsilon (`abs(qty_owned - qty_sell) < 1e-9` counts as equal) rather than raw `>=`, to avoid floating-point false-negatives (e.g. `0.1 + 0.2 != 0.3` in Python).

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Recorded every 30 seconds by a background task, and immediately after each trade execution.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `total_value` REAL
- `recorded_at` TEXT (ISO timestamp)

**chat_messages** — Conversation history with LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `role` TEXT (`"user"` or `"assistant"`)
- `content` TEXT
- `actions` TEXT (JSON — trades executed, watchlist changes made; null for user messages)
- `created_at` TEXT (ISO timestamp)

### Default Seed Data

- One user profile: `id="default"`, `cash_balance=10000.0`
- Ten watchlist entries: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

---

## 8. API Endpoints

### Market Data
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates |

### Portfolio
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio` | Current positions, cash balance, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a trade: `{ticker, quantity, side}` |
| GET | `/api/portfolio/history` | Portfolio value snapshots over time (for P&L chart) |

**`GET /api/portfolio`** response:
```json
{
  "cash_balance": 7430.50,
  "total_value": 12540.75,
  "positions": [
    {
      "ticker": "AAPL",
      "quantity": 10,
      "avg_cost": 189.50,
      "current_price": 193.20,
      "unrealized_pnl": 37.00,
      "pnl_pct": 1.95
    }
  ]
}
```

**`POST /api/portfolio/trade`** request: `{"ticker": "AAPL", "quantity": 5, "side": "buy"}`
Response:
```json
{
  "ticker": "AAPL",
  "side": "buy",
  "quantity": 5,
  "price": 193.20,
  "cash_balance": 5463.50,
  "executed_at": "2024-01-15T10:00:00Z"
}
```

**`GET /api/portfolio/history`** accepts an optional `?limit=N` query parameter (default 500). The database applies the limit, returning the most recent N snapshots ordered by `recorded_at` ascending. Response:
```json
{
  "snapshots": [
    {"recorded_at": "2024-01-15T10:00:00Z", "total_value": 10000.00},
    {"recorded_at": "2024-01-15T10:00:30Z", "total_value": 10045.50}
  ]
}
```

**Error responses** (all endpoints): `{"detail": "human-readable error message"}` with appropriate HTTP status (400 for validation errors, 404 for not found, 500 for server errors).

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist tickers with latest prices |
| POST | `/api/watchlist` | Add a ticker: `{ticker}` |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker |

**`GET /api/watchlist`** response:
```json
{
  "tickers": [
    {"ticker": "AAPL", "price": 193.20, "previous_price": 192.80, "day_open_price": 191.00, "updated_at": "2024-01-15T10:00:00Z"}
  ]
}
```
`day_open_price` is required so the frontend can compute daily change % (`(current - day_open) / day_open * 100`) on initial page load before any SSE events arrive.

**`POST /api/watchlist`** request: `{"ticker": "PYPL"}` → responds with the new watchlist entry (same shape as one item above) and HTTP 201. Maximum 50 tickers per watchlist; returns HTTP 400 if the limit is exceeded.

Ticker validation rules: ticker must match `^[A-Z]{1,5}$` (1–5 uppercase letters; use `^[A-Z.]{1,5}$` to also allow `.` for symbols like `BRK.B`). Returns HTTP 400 with a descriptive error if the format is invalid. In simulator mode any valid-format ticker is accepted and immediately begins simulating. In Massive API mode an unknown ticker is accepted but will show no price until the next poll; this is an accepted limitation.

**`DELETE /api/watchlist/{ticker}`** → HTTP 204 No Content on success. If the user holds an open position in that ticker, the deletion is **allowed** — the position remains in the `positions` table and continues to appear in the Positions table UI, but the ticker will no longer appear in the Watchlist panel or receive a price feed entry in the UI. This is an accepted limitation for the demo scope.

### Chat
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/chat` | Send a message, receive complete JSON response (message + executed actions) |

**`POST /api/chat`** request: `{"message": "Buy 5 shares of AAPL"}`
Response:
```json
{
  "message": "Bought 5 shares of AAPL at $193.20. Your cash balance is now $5,463.50.",
  "trades": [{"ticker": "AAPL", "side": "buy", "quantity": 5, "price": 193.20, "status": "executed"}],
  "watchlist_changes": []
}
```
Trade `status` values: `"executed"` or `"failed"` (with a `"reason"` field added on failure).

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check (for Docker/deployment) |

**`GET /api/health`** response: `{"status": "ok"}`

---

## 9. LLM Integration

When writing code to make calls to LLMs, use cerebras-inference skill to use LiteLLM via OpenRouter to the model specified in `LLM_MODEL` (default: `openrouter/cerebras/llama-4-scout-17b-16e-instruct`) with Cerebras as the inference provider. The default model **must** be one that Cerebras serves on OpenRouter — do not use OpenAI model IDs with the Cerebras provider. Structured Outputs should be used to interpret the results.

There is an OPENROUTER_API_KEY in the .env file in the project root.

### How It Works

When the user sends a chat message, the backend:

1. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
2. Loads the last 20 messages from `chat_messages` as conversation history (10 exchanges — sufficient context while staying well within token limits)
3. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
4. Calls the LLM via LiteLLM → OpenRouter, requesting structured output, using the cerebras-inference skill
5. Parses the complete structured JSON response
6. Auto-executes any trades or watchlist changes specified in the response
7. Stores the message and executed actions in `chat_messages`
8. Returns the complete JSON response to the frontend (no token-by-token streaming — Cerebras inference is fast enough that a loading indicator is sufficient)

### Structured Output Schema

The LLM is instructed to respond with JSON matching this schema:

```json
{
  "message": "Your conversational response to the user",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add"}
  ]
}
```

- `message` (required): The conversational text shown to the user
- `trades` (optional): Array of trades to auto-execute. Each trade goes through the same validation as manual trades (sufficient cash for buys, sufficient shares for sells)
- `watchlist_changes` (optional): Array of watchlist modifications. Valid `action` values: `"add"` or `"remove"`

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

If a trade fails validation (e.g., insufficient cash), the backend executes all trades synchronously before returning the response. Each trade result carries a `status` of `"executed"` or `"failed"` with a `"reason"` field. The LLM's `message` is returned as-is; the frontend is responsible for surfacing any `"failed"` trade statuses inline in the chat. No second LLM call is made.

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true`, the backend returns deterministic mock responses instead of calling OpenRouter. This enables:
- Fast, free, reproducible E2E tests
- Development without an API key
- CI/CD pipelines

The canonical mock response (returned for every input regardless of message content) is:
```json
{
  "message": "Mock response: I can help you manage your portfolio. What would you like to do?",
  "trades": [],
  "watchlist_changes": []
}
```
All agents implementing the mock must return exactly this structure so E2E tests have a stable, predictable response to assert against.

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), daily change % (`(current - day_open) / day_open * 100`), and a sparkline mini-chart (accumulated from SSE since page load)
- **Main chart area** — larger chart for the currently selected ticker, with at minimum price over time. Clicking a ticker in the watchlist selects it here.
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by P&L (green = profit, red = loss)
- **P&L chart** — line chart showing total portfolio value over time, using data from `portfolio_snapshots`. Display all available snapshots (effectively the current session since data starts on first launch)
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, % change
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill.
- **AI chat panel** — docked/collapsible sidebar. Message input, scrolling conversation history, loading indicator while waiting for LLM response. Trade executions and watchlist changes shown inline as confirmations.
- **Header** — portfolio total value (updating live), connection status indicator, cash balance

### Technical Notes

- Use `EventSource` for SSE connection to `/api/stream/prices`
- Canvas-based charting library preferred (Lightweight Charts or Recharts) for performance
- Price flash effect: on receiving a new price, briefly apply a CSS class with background color transition, then remove it
- All API calls go to the same origin (`/api/*`) — no CORS configuration needed
- Tailwind CSS for styling with a custom dark theme

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node 20 slim
  - Copy frontend/
  - npm install && npm run build (produces static export at frontend/out/)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/
  - uv sync (install Python dependencies from lockfile)
  - Copy frontend/out/ → /app/static/
  - Expose port 8000
  - CMD: uvicorn serving FastAPI app
```

FastAPI mounts `/app/static` via `StaticFiles` and serves all API routes on port 8000. The static path `/app/static` is the canonical agreed location — both the Dockerfile and the FastAPI `StaticFiles` mount must use this exact path.

### Docker Volume

The SQLite database and event logs persist via named Docker volumes:

```bash
docker run \
  -v finally-data:/app/db \
  -v finally-logs:/app/backend/logs \
  -p 8000:8000 \
  --env-file .env \
  finally
```

The `db/` directory maps to `/app/db` in the container; the backend writes `finally.db` here. The `backend/logs/` directory maps to `/app/backend/logs`; the backend writes `events.jsonl` here. Both volumes persist across container restarts. Without the logs volume, event logs are lost on restart — contradicting the event stream's role as debugging source of truth.

### Start/Stop Scripts

**`scripts/start_mac.sh`** (macOS/Linux):
- Builds the Docker image if not already built (or if `--build` flag passed)
- Runs the container with the volume mount, port mapping, and `.env` file
- Prints the URL to access the app
- Optionally opens the browser

**`scripts/stop_mac.sh`** (macOS/Linux):
- Stops and removes the running container
- Does NOT remove the volume (data persists)

**`scripts/start_windows.ps1`** / **`scripts/stop_windows.ps1`**: PowerShell equivalents for Windows.

All scripts should be idempotent — safe to run multiple times.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Market data: simulator generates valid prices, GBM math is correct, Massive API response parsing works, both implementations conform to the abstract interface
- Portfolio: trade execution logic, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, selling at a loss)
- LLM: structured output parsing handles all valid schemas, graceful handling of malformed responses, trade validation within chat flow
- API routes: correct status codes, response shapes, error handling

**Frontend (Vitest or Jest)**:
- Pure logic: P&L calculations, price change direction, sparkline data accumulation
- SSE event parsing and price cache updates
- Trade form validation (negative quantity, non-numeric input)

### Component Tests (within `frontend/`)

Use React Testing Library. Each major UI component is tested in isolation with mock data and mock API responses:
- `WatchlistPanel`: renders tickers, applies flash class on price update, removes it after transition
- `PositionsTable`: correct P&L sign, color coding, zero-quantity rows hidden
- `TradeBar`: buy/sell button states, disabled while request in flight
- `ChatPanel`: renders message history, shows loading indicator, displays inline trade confirmations
- `PortfolioHeatmap`: treemap renders with correct number of rectangles for given positions
- `PnLChart`: line chart renders with snapshot data, handles empty state

### Integration Tests (within `backend/`)

Test the backend services wired together against a real (in-memory or temp file) SQLite database. No mocked DB layer:
- Full trade flow: POST `/api/portfolio/trade` → verify DB state, cash balance, position row, trade log entry, portfolio snapshot
- Watchlist add/remove → verify SSE stream includes/excludes the ticker on next broadcast. Use `httpx-sse` (or `aiohttp`) to consume the SSE endpoint within pytest async tests.
- Chat flow with `LLM_MOCK=true`: POST `/api/chat` → verify trade executed in DB, message stored in `chat_messages`
- DB startup initialization: fresh SQLite file gets correct schema and seed data on app start
- `request_id` propagates through a full request and appears in `events.jsonl`
- Anomaly detection: one unit test per rule in Section 13.4 verifying that the correct `anomaly_detected` event is emitted (e.g. inject a price jump >10%, assert the event appears in the log)

### Feature Tests (E2E in `test/`, against running container)

Feature tests exercise complete user-facing scenarios via Playwright, treating the app as a black box. Run with `LLM_MOCK=true`:
- Fresh start: default watchlist appears, $10k balance shown, prices streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio heatmap renders with correct colors after a trade
- P&L chart has data points (run with `SNAPSHOT_INTERVAL_SECONDS=1` so the test does not wait 30 seconds)
- AI chat (mocked): send a message, receive a response, trade execution shown inline
- SSE resilience: simulate disconnect, verify reconnection and price updates resume

### Browser Tests (E2E in `test/`)

Run the full Playwright feature test suite against three browser engines to catch browser-specific rendering and API differences:
- **Chromium** (primary — most users)
- **Firefox**
- **WebKit** (Safari proxy)

All three browsers must pass the full feature test suite. Browser matrix runs in the same `docker-compose.test.yml` using Playwright's multi-browser support.

### Accessibility Tests (within `frontend/` and E2E)

**Static (component level)** — use `jest-axe` or `vitest-axe` in component tests:
- Every interactive component passes axe-core rules with zero violations
- All images and icons have meaningful `alt` text or `aria-label`
- Form inputs have associated `<label>` elements

**Dynamic (E2E level)** — run `@axe-core/playwright` on key page states:
- Initial load state
- Post-trade state (positions table visible)
- Chat panel open state

**Manual checklist** (verified once, not automated):
- Full keyboard navigation: tab order is logical, all interactive elements reachable
- Focus is visible at all times (no `:focus { outline: none }` without replacement)
- Color contrast meets WCAG AA (4.5:1 for normal text, 3:1 for large text) — the accent yellow `#ecad0a` on dark backgrounds must be verified
- Price flash colors (green/red) are not the sole indicator of direction — include a `▲`/`▼` symbol or similar for color-blind users

### Security Tests

**Dependency CVE scanning** — run on every build and as a scheduled CI check:
- **Backend**: `pip-audit` scans Python dependencies from `uv.lock` for known CVEs
- **Frontend**: `npm audit` scans Node dependencies for known vulnerabilities
- Both tools must exit 0 (no high/critical vulnerabilities). If a CVE is found, the build fails and the agent must upgrade or patch the dependency before proceeding.

**General security checks** — automated via `bandit` (Python) and `eslint-plugin-security` (TypeScript):
- No use of `eval()`, `exec()`, or `subprocess.shell=True`
- No hardcoded secrets or API keys in source files
- SQL queries use parameterized statements — no string interpolation into queries
- SSE endpoint does not reflect unsanitized user input into the event stream
- No `dangerouslySetInnerHTML` in React components unless explicitly reviewed

**OWASP Top 10 checklist** — verified manually or via `zap-baseline` scan against running container:
- Injection: all DB queries parameterized; no shell injection in ticker validation
- Broken access control: not applicable (single user, no auth)
- Security misconfiguration: no debug endpoints in production; no stack traces exposed in API error responses
- Vulnerable components: covered by CVE scanning above
- Insecure design: trade validation always server-side; client input never trusted

**Auto-fix policy**: when CVE scanning or static analysis finds an issue, the responsible agent must fix it immediately — either by upgrading the dependency to a patched version or removing the insecure pattern — before marking the task complete.

---

## 13. Event Logging

### 13.1 Event Storage

All events are appended to a JSONL stream:

`/backend/logs/events.jsonl`

Events must be:

- append-only
- immutable
- timestamped
- correlated via `request_id` or `session_id`

**Rotation policy**: a dedicated background asyncio task checks the file size every 60 seconds. When `events.jsonl` exceeds 50 MB, it renames it to `events.jsonl.old` and opens a fresh `events.jsonl`. At most two files exist at any time. Rotation must be performed only by this single background task — never inline per-event — to eliminate the TOCTOU race where two concurrent writers both attempt to rename simultaneously.

---

### 13.2 Base Event Schema

Every event must include:

- `timestamp`
- `event_type`
- `request_id`
- `user_id` (default: `"default"`)
- `context` (route, ticker, or subsystem)
- `metadata` (object)

**`request_id` generation**: FastAPI middleware generates a UUID4 per incoming HTTP request and stores it in a `contextvars.ContextVar`. All event logging within that request reads from this context automatically. Background tasks (simulator ticks, snapshot recorder, SSE broadcasts) generate their own `request_id` as `"<task_type>:<unix_timestamp_ms>"` (e.g., `"sse_broadcast:1705312800123"`).

---

### 13.3 Core Event Types

#### Market Data

Event types:
- `market_data_request`
- `market_data_response`
- `market_data_error`
- `market_data_cache_hit`

Fields:
- `ticker`
- `provider`
- `latency_ms`
- `price`
- `stale` (boolean)

---

#### Portfolio Events

Event types:
- `trade_requested`
- `trade_executed`
- `trade_failed`
- `portfolio_recalculated`

Fields:
- `ticker`
- `side`
- `quantity`
- `price`
- `cash_balance_after`

---

#### Watchlist Events

Event types:
- `watchlist_add`
- `watchlist_remove`

Fields:
- `ticker`

---

#### LLM / Agent Events

Event types:
- `llm_request`
- `llm_response`
- `llm_trade_executed`
- `llm_error`

Fields:
- `prompt_tokens` (optional)
- `response_valid` (boolean)
- `trades_generated`

---

#### System Events

Event types:
- `api_latency`
- `db_query`
- `sse_broadcast`
- `cache_update`

Fields:
- `duration_ms`
- `endpoint` (if applicable)

**SSE broadcast logging**: `sse_broadcast` events must be throttled — log at most one per 10 seconds per connected client, not on every 500ms tick. Logging every tick would generate ~7,200 events/minute and exhaust the 50 MB rotation limit within minutes under normal operation. Only errors and anomalies within the broadcast loop are logged immediately.

---

### 13.4 Deterministic Anomaly Rules

The system must emit anomaly events when conditions are violated:

- price jump > 10% in < 1 second
- API latency > 1500ms
- stale market data > 60 seconds
- `trade_executed` event where cash was insufficient (means validation was bypassed — must never happen; a normal validation rejection produces `trade_failed`, not this anomaly)
- SSE disconnect > 5 seconds

Event type: `anomaly_detected`

Fields:
- `anomaly_type`
- `severity` (`low` / `medium` / `high`)
- `context`
- `expected_value`
- `actual_value`

---

### 13.5 Agent Usage Rule

Agents (or debugging systems) should NOT rely on console logs.

They must use the event stream as the source of truth for:

- debugging trading issues
- diagnosing price inconsistencies
- tracing user actions
- validating portfolio correctness

---

### 13.6 Design Principle

Events should make the system replayable.

Given the event stream, it should be possible to reconstruct:

- user portfolio state at any time
- price evolution
- all trades and their causes
- LLM decisions and actions

---

## 14. Decisions & Notes

- **P&L chart data cap**: `GET /api/portfolio/history` accepts `?limit=N` (default 500) and applies the limit server-side, returning the most recent N snapshots. The frontend passes the default limit on every call — no client-side slicing is needed. This avoids fetching thousands of rows over the network only to discard them. (~4 hours of data at 30-second intervals fits within the default 500-point limit.)