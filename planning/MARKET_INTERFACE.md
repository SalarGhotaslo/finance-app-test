# Market Data Provider Interface

## Overview

FinAlly uses two market data backends: a **Massive API poller** for real market data and a **built-in simulator** for development and demo use. All downstream code — the SSE broadcast loop, portfolio P&L calculations, and the `/api/watchlist` endpoint — talks to one shared in-memory price cache. The cache is populated by whichever provider is active.

The backend selects the provider at startup based on the `MASSIVE_API_KEY` environment variable:
- Set and non-empty → `MassiveMarketDataProvider`
- Absent or empty → `SimulatorMarketDataProvider`

---

## Shared Price Cache

The price cache is a plain Python `dict` stored in application state. It is the single source of truth for current prices throughout the backend. Both providers write to it; everything else reads from it.

```python
# Type alias for a single ticker's cached entry
from dataclasses import dataclass
from datetime import datetime

@dataclass
class PriceData:
    ticker:          str
    price:           float   # most recent price
    previous_price:  float   # price just before the most recent update (for flash direction)
    day_open_price:  float   # first price of the current calendar day (UTC midnight reset)
    updated_at:      str     # ISO 8601 UTC timestamp string, e.g. "2024-01-15T10:00:00Z"

# Shared cache: ticker → PriceData
# Held in FastAPI app.state.price_cache
price_cache: dict[str, PriceData] = {}
```

### Cache Invariants

1. Every ticker in the watchlist table **must** have an entry in the cache (populated at startup and when tickers are added).
2. `previous_price` always holds the price from the **previous broadcast cycle**, not the previous session close. This drives the green/red flash animation on the frontend.
3. `day_open_price` resets to the current price at UTC midnight, enabling the daily % change display even across multi-day sessions.
4. The cache is only written from within the single asyncio event loop — never from threads. No locks are needed.

---

## Abstract Interface

Both providers implement the same abstract base class. The application code never imports a concrete provider directly — only the abstract type.

```python
# backend/market/interface.py

from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import datetime


@dataclass
class PriceData:
    ticker:         str
    price:          float
    previous_price: float
    day_open_price: float
    updated_at:     str   # ISO 8601 UTC


class MarketDataProvider(ABC):
    """
    Abstract market data provider. Both the Massive poller and the simulator
    implement this interface. The application sees only this type.
    """

    @abstractmethod
    async def start(self, price_cache: dict[str, PriceData]) -> None:
        """
        Called once during FastAPI lifespan startup. Implementations should:
        - Pre-populate the price_cache with initial values for all tickers in the watchlist
        - Start any background asyncio tasks (polling loop, simulator tick loop)
        The price_cache dict is owned by the caller; the provider writes into it.
        """
        ...

    @abstractmethod
    async def stop(self) -> None:
        """
        Called during FastAPI lifespan shutdown. Cancel and await background tasks.
        """
        ...

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """
        Called when a new ticker is added to the watchlist. The provider must
        begin tracking it — fetch an initial price (Massive) or seed it with a
        realistic starting price (simulator) — and add it to the price_cache.
        The ticker will appear in the next SSE broadcast cycle (~500 ms).
        """
        ...

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """
        Called when a ticker is removed from the watchlist. The provider stops
        updating it. The entry may remain in the cache (reads are harmless)
        but will no longer receive fresh updates.
        """
        ...
```

---

## Factory Function

A single factory function reads the environment and returns the correct concrete provider. The FastAPI app imports only this function.

```python
# backend/market/factory.py

import os
from .interface import MarketDataProvider
from .massive   import MassiveMarketDataProvider
from .simulator import SimulatorMarketDataProvider


def create_market_data_provider() -> MarketDataProvider:
    """
    Returns MassiveMarketDataProvider if MASSIVE_API_KEY is set and non-empty,
    otherwise returns SimulatorMarketDataProvider.
    """
    api_key = os.getenv("MASSIVE_API_KEY", "").strip()
    if api_key:
        return MassiveMarketDataProvider(api_key=api_key)
    return SimulatorMarketDataProvider()
```

---

## Massive API Provider

The Massive provider polls the v2 bulk snapshot endpoint on a configurable interval.

```python
# backend/market/massive.py

import asyncio
import httpx
from datetime import datetime, timezone
from .interface import MarketDataProvider, PriceData


SNAPSHOT_URL = (
    "https://api.polygon.io/v2/snapshot/locale/us/markets/stocks/tickers"
)


class MassiveMarketDataProvider(MarketDataProvider):
    """
    Polls the Massive (Polygon.io) REST API for real market data.
    Uses the v2 bulk snapshot endpoint — one request covers all watchlist tickers.

    Polling intervals (configured via MASSIVE_POLL_INTERVAL_SECONDS env var):
      Free tier:   15 seconds  (5 req/min limit)
      Paid tiers:  2–5 seconds (no rate cap)
    """

    def __init__(self, api_key: str, poll_interval: float = 15.0):
        self._api_key       = api_key
        self._poll_interval = poll_interval
        self._price_cache:  dict[str, PriceData] | None = None
        self._task:         asyncio.Task | None = None
        self._watched:      set[str] = set()

    async def start(self, price_cache: dict[str, PriceData]) -> None:
        self._price_cache = price_cache
        # Seed the cache immediately before the first poll cycle
        await self._poll_once()
        self._task = asyncio.create_task(self._poll_loop(), name="massive_poll")

    async def stop(self) -> None:
        if self._task:
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass

    async def add_ticker(self, ticker: str) -> None:
        self._watched.add(ticker)
        # Seed immediately so the ticker appears in the next SSE broadcast
        await self._poll_once(tickers=[ticker])

    async def remove_ticker(self, ticker: str) -> None:
        self._watched.discard(ticker)

    async def _poll_loop(self) -> None:
        while True:
            await asyncio.sleep(self._poll_interval)
            await self._poll_once()

    async def _poll_once(self, tickers: list[str] | None = None) -> None:
        targets = tickers or list(self._watched)
        if not targets:
            return

        headers = {"Authorization": f"Bearer {self._api_key}"}
        params  = {"tickers": ",".join(targets)}

        try:
            async with httpx.AsyncClient(timeout=10.0) as client:
                resp = await client.get(SNAPSHOT_URL, headers=headers, params=params)
                resp.raise_for_status()
                data = resp.json()
        except Exception as exc:
            # Log anomaly (stale data > 60 s triggers anomaly_detected event)
            # Do not crash — keep stale cache values until next successful poll
            return

        for t in data.get("tickers", []):
            ticker   = t["ticker"]
            new_price = t["lastTrade"]["p"]
            prev      = self._price_cache.get(ticker)

            self._price_cache[ticker] = PriceData(
                ticker         = ticker,
                price          = new_price,
                previous_price = prev.price if prev else t["prevDay"]["c"],
                day_open_price = t["day"]["o"],
                updated_at     = datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ"),
            )
```

---

## Simulator Provider

See `MARKET_SIMULATOR.md` for the full simulator specification. The simulator's `start()` method seeds the cache with realistic prices and launches the GBM tick loop.

```python
# backend/market/simulator.py  (abbreviated — see MARKET_SIMULATOR.md for full spec)

import asyncio
from .interface import MarketDataProvider, PriceData


class SimulatorMarketDataProvider(MarketDataProvider):

    async def start(self, price_cache: dict[str, PriceData]) -> None:
        self._price_cache = price_cache
        self._seed_initial_prices()
        self._task = asyncio.create_task(self._tick_loop(), name="simulator_tick")

    async def stop(self) -> None:
        ...  # cancel task

    async def add_ticker(self, ticker: str) -> None:
        ...  # seed with a realistic starting price

    async def remove_ticker(self, ticker: str) -> None:
        ...  # stop tracking
```

---

## FastAPI Integration

The provider lifecycle is managed in the FastAPI lifespan context manager. This is the only place where the factory is called.

```python
# backend/main.py

from contextlib import asynccontextmanager
from fastapi import FastAPI
from .market.factory import create_market_data_provider
from .market.interface import PriceData


@asynccontextmanager
async def lifespan(app: FastAPI):
    # ── Startup ──────────────────────────────────────────────────────────────
    app.state.price_cache = {}          # shared in-memory cache
    app.state.provider    = create_market_data_provider()

    # Initialise the watchlist from the database
    from .db import get_watchlist_tickers
    initial_tickers = await get_watchlist_tickers(user_id="default")
    for ticker in initial_tickers:
        app.state.provider._watched.add(ticker)  # register before start()

    # Start provider (seeds cache + launches background task)
    await app.state.provider.start(app.state.price_cache)

    # Start other background tasks (SSE broadcast, snapshot recorder, log rotator)
    ...

    yield

    # ── Shutdown ─────────────────────────────────────────────────────────────
    await app.state.provider.stop()
    # Cancel other background tasks


app = FastAPI(lifespan=lifespan)
```

---

## Reading from the Cache in Route Handlers

Route handlers read from `request.app.state.price_cache` — they never call the provider directly.

```python
# backend/routes/watchlist.py

from fastapi import APIRouter, Request
from ..market.interface import PriceData

router = APIRouter()

@router.get("/api/watchlist")
async def get_watchlist(request: Request):
    cache: dict[str, PriceData] = request.app.state.price_cache
    db_tickers = await fetch_watchlist_from_db(user_id="default")

    result = []
    for ticker in db_tickers:
        entry = cache.get(ticker)
        result.append({
            "ticker":         ticker,
            "price":          entry.price          if entry else None,
            "previous_price": entry.previous_price if entry else None,
            "day_open_price": entry.day_open_price if entry else None,
            "updated_at":     entry.updated_at     if entry else None,
        })
    return {"tickers": result}


@router.post("/api/watchlist", status_code=201)
async def add_to_watchlist(request: Request, body: dict):
    ticker   = body["ticker"].upper()
    provider = request.app.state.provider

    await save_ticker_to_db(ticker, user_id="default")
    await provider.add_ticker(ticker)  # seeds cache, starts tracking

    # Return the new entry (price will be present if seeding succeeded)
    entry = request.app.state.price_cache.get(ticker)
    return {
        "ticker":         ticker,
        "price":          entry.price          if entry else None,
        "previous_price": entry.previous_price if entry else None,
        "day_open_price": entry.day_open_price if entry else None,
        "updated_at":     entry.updated_at     if entry else None,
    }
```

---

## SSE Broadcast Loop

The SSE broadcast loop reads the full cache once per tick and pushes all entries to connected clients. It does not care which provider populated the cache.

```python
# backend/routes/stream.py

import asyncio
import json
from fastapi import APIRouter
from fastapi.responses import StreamingResponse

router = APIRouter()

@router.get("/api/stream/prices")
async def stream_prices(request: Request):
    async def event_generator():
        while True:
            if await request.is_disconnected():
                break

            cache = request.app.state.price_cache
            for ticker, entry in list(cache.items()):
                payload = json.dumps({
                    "ticker":          entry.ticker,
                    "price":           entry.price,
                    "previous_price":  entry.previous_price,
                    "day_open_price":  entry.day_open_price,
                    "updated_at":      entry.updated_at,
                    "direction":       "up" if entry.price >= entry.previous_price else "down",
                })
                yield f"data: {payload}\n\n"

            await asyncio.sleep(0.5)   # broadcast at ~500 ms cadence

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"},
    )
```

---

## Watchlist Add/Remove Flow

```
User POST /api/watchlist          User DELETE /api/watchlist/{ticker}
         │                                      │
         ▼                                      ▼
  Validate ticker format             Remove from DB
         │                                      │
  Write to DB (watchlist table)      provider.remove_ticker(ticker)
         │
  provider.add_ticker(ticker)
         │
  ┌──────┴──────────────────────┐
  │  MassiveProvider            │  SimulatorProvider
  │  → poll snapshot for ticker │  → seed GBM params + initial price
  │  → write to price_cache     │  → write to price_cache
  └──────────────────────────────┘
         │
  Ticker appears in next SSE broadcast (~500 ms)
  Returns HTTP 201 with current price data
```

---

## Configuration Summary

| Environment variable | Default | Effect |
|---------------------|---------|--------|
| `MASSIVE_API_KEY` | (empty) | If set: use Massive API. If empty: use simulator. |
| `MASSIVE_POLL_INTERVAL_SECONDS` | `15` | Polling interval in seconds. Reduce to 2–5 on paid Massive plans. |

The backend must validate `MASSIVE_API_KEY` format at startup if provided (non-empty, looks like a valid key string) and log a warning — but not fail to start — if the key is set but the first API call returns 401/403. In that case the system falls back to the simulator and logs a clear error.
