# Massive API (formerly Polygon.io) — Integration Reference

## Overview

Massive (rebranded from Polygon.io in late 2025) provides REST and WebSocket APIs for US equity market data. The base URL is `https://api.polygon.io` (the old domain still resolves; `https://api.massive.com` is the new canonical domain). All existing API keys continue to work unchanged.

This document covers the endpoints used by FinAlly: bulk price snapshots for the watchlist poller, single-ticker detail, and daily OHLCV bars.

---

## Authentication

Pass the API key in the `Authorization` header (preferred) or as a query parameter.

```python
import requests

BASE_URL = "https://api.polygon.io"
API_KEY  = "your_key_here"

# Option 1: Authorization header (preferred)
headers = {"Authorization": f"Bearer {API_KEY}"}
resp = requests.get(f"{BASE_URL}/v2/...", headers=headers)

# Option 2: Query parameter
resp = requests.get(f"{BASE_URL}/v2/...", params={"apiKey": API_KEY})
```

---

## Rate Limits

| Plan | Rate limit | Data freshness | Notes |
|------|-----------|----------------|-------|
| Free | **5 req/min** | End-of-day only | Suitable for dev/testing with historical data only |
| Starter (~$29/mo) | Unlimited | 15-min delayed | Sufficient for watchlist polling every 15 s |
| Developer (~$200/mo) | Unlimited | Real-time | Required for live price feeds |
| Advanced (~$500/mo) | Unlimited | Real-time + pre/after-hours | Full data access |

**For FinAlly**: the Starter plan supports the 15-second polling interval described in PLAN.md. A single bulk snapshot call covers all 10–50 watchlist tickers — well within even free-tier limits if polling less frequently.

---

## Endpoint Reference

### 1. Bulk Snapshot — All Watchlist Tickers in One Request

This is the primary endpoint for the polling loop. One request returns current price, previous close, and today's open for up to 250 tickers.

**`GET /v2/snapshot/locale/us/markets/stocks/tickers`**

| Parameter | Type | Description |
|-----------|------|-------------|
| `tickers` | string | Comma-separated list, e.g. `AAPL,MSFT,TSLA`. Omit to get all tickers. |
| `include_otc` | boolean | Include OTC securities. Default: `false`. |

```python
import requests
from datetime import datetime

BASE_URL = "https://api.polygon.io"

def fetch_bulk_snapshot(tickers: list[str], api_key: str) -> dict:
    """
    Fetch current price data for a list of tickers in a single request.
    Returns a dict mapping ticker → price data.
    """
    url = f"{BASE_URL}/v2/snapshot/locale/us/markets/stocks/tickers"
    headers = {"Authorization": f"Bearer {api_key}"}
    params  = {"tickers": ",".join(tickers)}

    resp = requests.get(url, headers=headers, params=params, timeout=10)
    resp.raise_for_status()
    data = resp.json()

    results = {}
    for t in data.get("tickers", []):
        ticker     = t["ticker"]
        last_price = t["lastTrade"]["p"]          # latest trade price
        prev_close = t["prevDay"]["c"]             # previous session close
        day_open   = t["day"]["o"]                 # today's open
        updated_ns = t["updated"]                  # nanosecond timestamp
        updated_at = datetime.utcfromtimestamp(updated_ns / 1e9)

        results[ticker] = {
            "price":           last_price,
            "previous_price":  prev_close,         # used for flash direction
            "day_open_price":  day_open,
            "updated_at":      updated_at.isoformat() + "Z",
        }
    return results

# Usage
prices = fetch_bulk_snapshot(
    ["AAPL", "MSFT", "TSLA", "NVDA", "GOOGL", "META", "AMZN", "JPM", "V", "NFLX"],
    api_key="YOUR_KEY"
)
for ticker, data in prices.items():
    chg_pct = (data["price"] - data["day_open_price"]) / data["day_open_price"] * 100
    print(f"{ticker}: ${data['price']:.2f}  ({chg_pct:+.2f}% today)")
```

**Response structure (one item in `tickers` array):**
```json
{
  "ticker": "AAPL",
  "day": {
    "o": 191.00,
    "h": 194.50,
    "l": 190.75,
    "c": 193.20,
    "v": 55000000,
    "vw": 192.84
  },
  "prevDay": {
    "o": 190.10,
    "h": 193.00,
    "l": 189.95,
    "c": 192.53,
    "v": 48000000,
    "vw": 191.70
  },
  "lastTrade": {
    "p": 193.20,
    "s": 100,
    "t": 1705312799123456789,
    "x": 4
  },
  "lastQuote": {
    "p": 193.18,
    "P": 193.22,
    "s": 3,
    "S": 5,
    "t": 1705312799987654321
  },
  "todaysChange": 0.67,
  "todaysChangePerc": 0.348,
  "updated": 1705312799123456789
}
```

Key fields:
- `lastTrade.p` — latest trade price (the "current price")
- `prevDay.c` — previous session close (for daily % change)
- `day.o` — today's open (for intraday % change)
- `updated` — nanosecond timestamp of last update

> **Note**: snapshot data clears at 3:30 AM EST and repopulates from 4:00 AM EST. Responses may be empty during that window.

---

### 2. Unified Snapshot (v3) — Alternative Bulk Endpoint

The v3 endpoint uses descriptive field names and supports up to 250 tickers. It is the newer API but v2 remains fully supported.

**`GET /v3/snapshot`**

| Parameter | Type | Description |
|-----------|------|-------------|
| `ticker.any_of` | string | Comma-separated list of tickers |
| `type` | string | Asset class filter: `stocks`, `options`, `fx`, `crypto`, `indices` |
| `limit` | integer | Results per page, max 250 |

```python
import requests

def fetch_bulk_snapshot_v3(tickers: list[str], api_key: str) -> dict:
    url     = "https://api.polygon.io/v3/snapshot"
    headers = {"Authorization": f"Bearer {api_key}"}
    params  = {"ticker.any_of": ",".join(tickers), "type": "stocks", "limit": 250}

    resp = requests.get(url, headers=headers, params=params, timeout=10)
    resp.raise_for_status()
    data = resp.json()

    results = {}
    for r in data.get("results", []):
        results[r["ticker"]] = {
            "price":          r["last_trade"]["price"],
            "previous_price": r["session"]["previous_close"],
            "day_open_price": r["session"]["open"],
            "updated_at":     ...,  # derive from r["last_trade"]["sip_timestamp"]
        }
    return results
```

**Response item structure (v3):**
```json
{
  "ticker": "AAPL",
  "type": "stocks",
  "last_trade": {
    "price": 193.20,
    "size": 100,
    "sip_timestamp": 1705312799123456789,
    "timeframe": "REAL-TIME"
  },
  "last_quote": {
    "bid": 193.18,
    "ask": 193.22,
    "bid_size": 3,
    "ask_size": 5
  },
  "session": {
    "open": 191.00,
    "high": 194.50,
    "low": 190.75,
    "close": 193.20,
    "previous_close": 192.53,
    "change": 0.67,
    "change_percent": 0.348,
    "volume": 55000000
  }
}
```

---

### 3. Single Ticker Snapshot

For fetching detailed data for one ticker (e.g., when a user clicks a ticker to view its chart).

**`GET /v2/snapshot/locale/us/markets/stocks/tickers/{ticker}`**

```python
import requests

def fetch_single_snapshot(ticker: str, api_key: str) -> dict:
    url = f"https://api.polygon.io/v2/snapshot/locale/us/markets/stocks/tickers/{ticker}"
    headers = {"Authorization": f"Bearer {api_key}"}

    resp = requests.get(url, headers=headers, timeout=10)
    resp.raise_for_status()
    t = resp.json()["ticker"]

    return {
        "price":          t["lastTrade"]["p"],
        "previous_price": t["prevDay"]["c"],
        "day_open_price": t["day"]["o"],
        "day_high":       t["day"]["h"],
        "day_low":        t["day"]["l"],
        "day_volume":     t["day"]["v"],
        "change_pct":     t["todaysChangePerc"],
    }
```

---

### 4. Previous Day Bar

Returns the complete OHLCV bar for the most recent trading session. Use this when initializing the price cache before the day's first snapshot poll.

**`GET /v2/aggs/ticker/{ticker}/prev`**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `adjusted` | boolean | `true` | Adjust for stock splits |

```python
import requests

def fetch_previous_close(ticker: str, api_key: str) -> dict:
    url = f"https://api.polygon.io/v2/aggs/ticker/{ticker}/prev"
    headers = {"Authorization": f"Bearer {api_key}"}
    params  = {"adjusted": "true"}

    resp = requests.get(url, headers=headers, params=params, timeout=10)
    resp.raise_for_status()
    bar = resp.json()["results"][0]

    return {
        "open":   bar["o"],
        "high":   bar["h"],
        "low":    bar["l"],
        "close":  bar["c"],   # this is the "previous_price" / prev close
        "volume": bar["v"],
        "vwap":   bar["vw"],
    }
```

**Response:**
```json
{
  "ticker": "AAPL",
  "adjusted": true,
  "resultsCount": 1,
  "results": [
    {
      "T": "AAPL",
      "o": 190.10,
      "h": 193.00,
      "l": 189.95,
      "c": 192.53,
      "v": 48000000,
      "vw": 191.70,
      "n": 750000,
      "t": 1705276800000
    }
  ]
}
```

---

### 5. Historical OHLCV Bars (Aggregates)

For rendering the main chart area when a user selects a ticker. Returns an array of OHLCV bars for any time range and resolution.

**`GET /v2/aggs/ticker/{ticker}/range/{multiplier}/{timespan}/{from}/{to}`**

| Parameter | Type | Description |
|-----------|------|-------------|
| `ticker` | path | Ticker symbol |
| `multiplier` | path | Bar size multiplier (e.g., `1`, `5`, `15`) |
| `timespan` | path | `minute`, `hour`, `day`, `week`, `month`, `quarter`, `year` |
| `from` | path | Start: `YYYY-MM-DD` or Unix milliseconds |
| `to` | path | End: `YYYY-MM-DD` or Unix milliseconds |
| `adjusted` | query | boolean, default `true` |
| `sort` | query | `asc` or `desc`, default `asc` |
| `limit` | query | Max results, default `5000`, hard cap `50000` |

```python
import requests

def fetch_ohlcv(
    ticker: str,
    api_key: str,
    from_date: str,
    to_date: str,
    multiplier: int = 1,
    timespan: str = "day",
) -> list[dict]:
    """Fetch OHLCV bars. Returns list of {t, o, h, l, c, v, vw} dicts."""
    url = (
        f"https://api.polygon.io/v2/aggs/ticker/{ticker}"
        f"/range/{multiplier}/{timespan}/{from_date}/{to_date}"
    )
    headers = {"Authorization": f"Bearer {api_key}"}
    params  = {"adjusted": "true", "sort": "asc", "limit": 50000}

    bars = []
    while url:
        resp = requests.get(url, headers=headers, params=params, timeout=10)
        resp.raise_for_status()
        data = resp.json()
        bars.extend(data.get("results", []))
        url    = data.get("next_url")   # follow pagination cursor
        params = {}                      # cursor encodes all params
    return bars

# Usage: intraday 1-minute bars for today
bars = fetch_ohlcv("AAPL", "YOUR_KEY", "2024-01-15", "2024-01-15", 1, "minute")
for b in bars[:5]:
    print(f"t={b['t']}  o={b['o']}  h={b['h']}  l={b['l']}  c={b['c']}  v={b['v']}")
```

**Response (one bar):**
```json
{
  "t":  1705312800000,
  "o":  193.00,
  "h":  193.25,
  "l":  192.95,
  "c":  193.20,
  "v":  25000,
  "vw": 193.12,
  "n":  150
}
```

Pagination: when the response includes `next_url`, fetch that URL with only the auth header (all filter params are embedded in the cursor).

---

### 6. End-of-Day Price for a Specific Date

**`GET /v1/open-close/{ticker}/{date}`**

```python
import requests

def fetch_daily_bar(ticker: str, date: str, api_key: str) -> dict:
    """date format: YYYY-MM-DD"""
    url     = f"https://api.polygon.io/v1/open-close/{ticker}/{date}"
    headers = {"Authorization": f"Bearer {api_key}"}

    resp = requests.get(url, headers=headers, params={"adjusted": "true"}, timeout=10)
    resp.raise_for_status()
    data = resp.json()

    return {
        "open":        data["open"],
        "high":        data["high"],
        "low":         data["low"],
        "close":       data["close"],
        "volume":      data["volume"],
        "pre_market":  data.get("preMarket"),
        "after_hours": data.get("afterHours"),
    }
```

**Response:**
```json
{
  "status":     "OK",
  "from":       "2024-01-15",
  "symbol":     "AAPL",
  "open":       191.00,
  "high":       194.50,
  "low":        190.75,
  "close":      193.20,
  "volume":     55000000,
  "preMarket":  191.50,
  "afterHours": 192.80,
  "otc":        false
}
```

---

## WebSocket Streaming (Optional, Developer Plan+)

The REST polling approach described in PLAN.md is the primary integration. WebSocket is documented here as a future upgrade path.

**Connection URL:** `wss://socket.polygon.io/stocks`

```python
import websocket
import json

def on_open(ws):
    ws.send(json.dumps({"action": "auth", "params": "YOUR_API_KEY"}))

def on_message(ws, message):
    events = json.loads(message)
    for ev in events:
        if ev["ev"] == "status" and ev["status"] == "auth_success":
            # Subscribe to trades for specific tickers
            ws.send(json.dumps({
                "action": "subscribe",
                "params": "T.AAPL,T.MSFT,T.TSLA"  # prefix T.* for all trades
            }))
        elif ev["ev"] == "T":   # Trade event
            ticker    = ev["sym"]
            price     = ev["p"]    # trade price
            size      = ev["s"]    # trade size
            timestamp = ev["t"]    # SIP timestamp (ms)
            print(f"{ticker}: ${price}  size={size}")

ws = websocket.WebSocketApp("wss://socket.polygon.io/stocks",
                             on_open=on_open, on_message=on_message)
ws.run_forever()
```

**WebSocket channel prefixes:**
| Prefix | Description |
|--------|-------------|
| `T.*` | Every executed trade |
| `Q.*` | NBBO quote updates (bid/ask) |
| `A.*` | Per-second aggregate bars |
| `AM.*` | Per-minute aggregate bars |

---

## Error Handling

All endpoints return HTTP 4xx/5xx with a JSON body on error:

```json
{
  "status":  "ERROR",
  "request_id": "abc123",
  "error":   "Ticker not found."
}
```

Common status codes:
| Code | Meaning |
|------|---------|
| 200 | OK |
| 400 | Bad request (invalid parameters) |
| 401 | Unauthorized (bad/missing API key) |
| 403 | Forbidden (plan does not include this endpoint) |
| 404 | Not found (ticker, date not found) |
| 429 | Rate limit exceeded |
| 500 | Server error |

```python
import requests

def safe_get(url: str, headers: dict, params: dict = None) -> dict | None:
    try:
        resp = requests.get(url, headers=headers, params=params or {}, timeout=10)
        resp.raise_for_status()
        return resp.json()
    except requests.HTTPError as e:
        if e.response.status_code == 429:
            # Back off — free tier hit 5 req/min limit
            raise
        elif e.response.status_code in (401, 403):
            raise RuntimeError(f"API key invalid or plan does not include endpoint: {url}") from e
        else:
            raise
    except requests.Timeout:
        return None  # caller logs anomaly and uses stale cache value
```

---

## Polling Architecture for FinAlly

The Massive poller runs as a single `asyncio` background task. It calls the v2 bulk snapshot endpoint for all tickers in the watchlist, updates the shared in-memory price cache, and sleeps until the next interval.

```python
import asyncio
import httpx                  # async HTTP client (preferred over requests in asyncio)
from datetime import datetime, timezone

async def massive_poll_loop(
    api_key: str,
    price_cache: dict,         # shared dict: ticker → PriceData
    get_tickers: callable,     # async fn returning current watchlist
    interval_seconds: float = 15.0,
):
    """
    Background asyncio task. Polls Massive API and updates price_cache.
    Interval: 15 s (free tier), 2–5 s (paid tier).
    """
    url = "https://api.polygon.io/v2/snapshot/locale/us/markets/stocks/tickers"
    headers = {"Authorization": f"Bearer {api_key}"}

    async with httpx.AsyncClient(timeout=10.0) as client:
        while True:
            try:
                tickers = await get_tickers()
                if tickers:
                    params = {"tickers": ",".join(tickers)}
                    resp = await client.get(url, headers=headers, params=params)
                    resp.raise_for_status()
                    data = resp.json()

                    for t in data.get("tickers", []):
                        ticker = t["ticker"]
                        prev   = price_cache.get(ticker)
                        price_cache[ticker] = {
                            "price":          t["lastTrade"]["p"],
                            "previous_price": prev["price"] if prev else t["prevDay"]["c"],
                            "day_open_price": t["day"]["o"],
                            "updated_at":     datetime.now(timezone.utc).isoformat(),
                        }
            except Exception as exc:
                # Log anomaly; do not crash the loop
                print(f"Massive poll error: {exc}")

            await asyncio.sleep(interval_seconds)
```

See `MARKET_INTERFACE.md` for how this integrates with the unified `MarketDataProvider` interface.

---

## Endpoint Quick Reference

| Use case | Endpoint | Notes |
|----------|----------|-------|
| Bulk prices (10–50 tickers) | `GET /v2/snapshot/…/tickers?tickers=A,B,C` | Primary polling endpoint |
| Bulk prices (v3, alt) | `GET /v3/snapshot?ticker.any_of=A,B,C` | Cleaner field names, up to 250 |
| Single ticker detail | `GET /v2/snapshot/…/tickers/{ticker}` | On-click detail view |
| Previous session close | `GET /v2/aggs/ticker/{ticker}/prev` | Cache init |
| Historical OHLCV bars | `GET /v2/aggs/ticker/{ticker}/range/1/day/{from}/{to}` | Main chart rendering |
| Specific day EOD | `GET /v1/open-close/{ticker}/{date}` | Backfill |
