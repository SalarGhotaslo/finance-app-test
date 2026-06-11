# FinAlly — AI Trading Workstation

FinAlly (Finance Ally) is an AI-powered trading workstation that streams live market data, lets you trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades on your behalf. It looks and feels like a modern Bloomberg terminal with an AI copilot.

## Features

- **Live price streaming** — prices flash green/red on uptick/downtick via SSE
- **Sparkline mini-charts** — per-ticker price history accumulated from the live stream
- **Simulated portfolio** — start with $10,000 in virtual cash, execute instant market orders
- **Portfolio heatmap** — treemap of positions sized by weight and colored by P&L
- **P&L chart** — portfolio value over time
- **Positions table** — ticker, quantity, avg cost, current price, unrealized P&L
- **AI chat assistant** — ask questions, get analysis, and have the AI execute trades and manage your watchlist via natural language

## Quick Start

### Prerequisites

- [Docker](https://www.docker.com/) installed and running
- An [OpenRouter](https://openrouter.ai/) API key

### Setup

1. Clone the repository:
   ```bash
   git clone https://github.com/salarghotaslo/finance-app-test.git
   cd finance-app-test
   ```

2. Copy the example env file and add your API key:
   ```bash
   cp .env.example .env
   # Edit .env and set OPENROUTER_API_KEY=your-key-here
   ```

3. Start the app:

   **macOS / Linux:**
   ```bash
   ./scripts/start_mac.sh
   ```

   **Windows (PowerShell):**
   ```powershell
   .\scripts\start_windows.ps1
   ```

4. Open [http://localhost:8000](http://localhost:8000) in your browser.

### Stopping

```bash
./scripts/stop_mac.sh        # macOS/Linux
.\scripts\stop_windows.ps1   # Windows
```

Data (SQLite database and event logs) persists in named Docker volumes across restarts.

## Environment Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `OPENROUTER_API_KEY` | **Yes** | — | OpenRouter API key for LLM chat |
| `MASSIVE_API_KEY` | No | — | Polygon.io key for real market data; omit to use the simulator |
| `LLM_MODEL` | No | `openrouter/cerebras/llama-4-scout-17b-16e-instruct` | Override the LLM model |
| `LLM_MOCK` | No | `false` | Set to `true` for deterministic mock LLM responses (testing) |
| `SNAPSHOT_INTERVAL_SECONDS` | No | `30` | How often portfolio snapshots are recorded |

## Architecture

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

- **Frontend**: Next.js + TypeScript, built as a static export, served by FastAPI
- **Backend**: FastAPI (Python), managed with `uv`
- **Database**: SQLite at `db/finally.db`, volume-mounted for persistence
- **Real-time data**: Server-Sent Events (SSE) — one-way server→client push
- **AI**: LiteLLM → OpenRouter (Cerebras inference) with structured outputs
- **Market data**: Built-in GBM simulator by default; real data via Massive API if key is provided

## Development

### Project Structure

```
finally/
├── frontend/         # Next.js TypeScript project
├── backend/          # FastAPI uv project
│   ├── schema/       # SQL schema and seed data
│   └── logs/         # Event log output (events.jsonl)
├── planning/         # Project documentation
├── scripts/          # Start/stop scripts
├── test/             # Playwright E2E tests
├── db/               # SQLite volume mount target
├── Dockerfile
└── docker-compose.yml
```

### Running Tests

E2E tests use Playwright against a running container with `LLM_MOCK=true`:

```bash
cd test
docker compose -f docker-compose.test.yml up --build
```

Backend unit tests:

```bash
cd backend
uv run pytest
```

Frontend unit/component tests:

```bash
cd frontend
npm test
```

## API Reference

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates |
| GET | `/api/portfolio` | Positions, cash balance, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a trade `{ticker, quantity, side}` |
| GET | `/api/portfolio/history` | Portfolio value snapshots (for P&L chart) |
| GET | `/api/watchlist` | Watchlist tickers with latest prices |
| POST | `/api/watchlist` | Add a ticker `{ticker}` |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker |
| POST | `/api/chat` | Send a chat message, receive AI response + executed actions |
| GET | `/api/health` | Health check |
