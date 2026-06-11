# Market Simulator

## Overview

The simulator generates realistic-looking stock price movements using Geometric Brownian Motion (GBM). It runs entirely in-process as an asyncio background task — no external dependencies, no network calls. It is the default provider when `MASSIVE_API_KEY` is not set.

The simulator is designed to:
- Produce plausible price trajectories that look like real market data
- Flash green and red convincingly (correlated sector moves, occasional dramatic events)
- Never produce negative prices
- Support adding new tickers dynamically at runtime
- Reset the day-open price at UTC midnight for multi-day sessions

---

## Geometric Brownian Motion

GBM is the standard model for equity prices. The discrete-time update formula is:

```
P(t+dt) = P(t) * exp((μ - σ²/2) * dt + σ * √dt * Z)
```

where:
- `P(t)` — current price
- `μ` (mu) — drift per second (annualised drift / seconds_per_year). For the simulator, drift is small and slightly positive to mimic long-term market behaviour.
- `σ` (sigma) — volatility per second (annualised vol / √seconds_per_year). Controls how much prices move each tick.
- `dt` — elapsed time in seconds since last tick (typically ~0.5 s)
- `Z` — standard normal random variable, `Z ~ N(0, 1)`

The exponential form guarantees prices stay positive — a multiplicative shock can approach zero but never cross it.

```python
import math
import numpy as np

def gbm_step(price: float, mu: float, sigma: float, dt: float) -> float:
    """Single GBM step. mu and sigma are per-second rates."""
    z = np.random.standard_normal()
    return price * math.exp((mu - 0.5 * sigma**2) * dt + sigma * math.sqrt(dt) * z)
```

---

## Seed Prices and Per-Ticker Parameters

Each ticker has a realistic seed price and calibrated volatility. More volatile tickers (TSLA, NVDA) have higher `sigma`; stable ones (JPM, V) have lower. All tickers are grouped into correlated sectors.

```python
# backend/market/simulator.py

from dataclasses import dataclass, field

@dataclass
class TickerConfig:
    seed_price: float    # Starting price (also used as initial day_open_price)
    mu:         float    # Annual drift, e.g. 0.05 = 5%/year
    sigma:      float    # Annual volatility, e.g. 0.25 = 25%/year
    sector:     str      # For correlation grouping

# Annualised → per-second conversion
SECONDS_PER_YEAR = 365.25 * 24 * 3600

def annual_to_per_second(annual_rate: float, is_vol: bool = False) -> float:
    if is_vol:
        return annual_rate / math.sqrt(SECONDS_PER_YEAR)
    return annual_rate / SECONDS_PER_YEAR

TICKER_CONFIGS: dict[str, TickerConfig] = {
    "AAPL": TickerConfig(seed_price=190.00, mu=0.07,  sigma=0.25, sector="tech"),
    "MSFT": TickerConfig(seed_price=415.00, mu=0.08,  sigma=0.22, sector="tech"),
    "GOOGL":TickerConfig(seed_price=175.00, mu=0.07,  sigma=0.24, sector="tech"),
    "META": TickerConfig(seed_price=510.00, mu=0.09,  sigma=0.30, sector="tech"),
    "AMZN": TickerConfig(seed_price=185.00, mu=0.08,  sigma=0.26, sector="tech"),
    "NVDA": TickerConfig(seed_price=875.00, mu=0.12,  sigma=0.45, sector="tech"),
    "TSLA": TickerConfig(seed_price=230.00, mu=0.05,  sigma=0.55, sector="ev"),
    "JPM":  TickerConfig(seed_price=200.00, mu=0.06,  sigma=0.18, sector="finance"),
    "V":    TickerConfig(seed_price=275.00, mu=0.07,  sigma=0.16, sector="finance"),
    "NFLX": TickerConfig(seed_price=630.00, mu=0.06,  sigma=0.32, sector="media"),
}

# Default config for unknown tickers added dynamically
DEFAULT_CONFIG = TickerConfig(seed_price=100.00, mu=0.06, sigma=0.28, sector="other")
```

---

## Sector Correlation

Stocks within the same sector share a common factor shock each tick. This creates the correlated moves (e.g., all tech stocks dipping together) that make the simulator feel real.

The model uses a one-factor decomposition:
```
Z_i = ρ * Z_market + √(1-ρ²) * Z_idiosyncratic
```

where:
- `ρ` (rho) — correlation between stocks in the same sector (0.6 for tech, 0.5 for finance, etc.)
- `Z_market` — common sector shock, one per sector per tick
- `Z_idiosyncratic` — independent noise, one per ticker per tick

```python
SECTOR_CORRELATIONS: dict[str, float] = {
    "tech":    0.6,
    "ev":      0.3,
    "finance": 0.5,
    "media":   0.3,
    "other":   0.0,
}

def correlated_normal(sector: str, sector_shocks: dict[str, float]) -> float:
    """
    Returns a correlated standard normal draw for a ticker in the given sector.
    sector_shocks: pre-drawn sector-level Z values, one per sector per tick.
    """
    rho    = SECTOR_CORRELATIONS.get(sector, 0.0)
    z_mkt  = sector_shocks.get(sector, 0.0)
    z_idio = np.random.standard_normal()
    return rho * z_mkt + math.sqrt(1 - rho**2) * z_idio
```

---

## Random Event Shocks

Occasional dramatic events add visual drama to the simulator and demonstrate the frontend's flash animation. A random shock is applied multiplicatively — guaranteed to keep prices positive.

```python
import random

@dataclass
class ShockConfig:
    probability_per_tick: float   # probability any given ticker gets a shock on this tick
    min_magnitude:        float   # minimum absolute shock size (e.g. 0.02 = 2%)
    max_magnitude:        float   # maximum absolute shock size (e.g. 0.05 = 5%)

EVENT_SHOCK = ShockConfig(
    probability_per_tick = 0.001,    # ~0.1% chance per ticker per 500ms tick ≈ one event every ~1000 s
    min_magnitude        = 0.02,
    max_magnitude        = 0.05,
)

def maybe_apply_shock(price: float) -> float:
    """
    With low probability, apply a sudden 2–5% move (up or down).
    Always multiplicative; always enforces the $0.01 price floor.
    """
    if random.random() < EVENT_SHOCK.probability_per_tick:
        magnitude = random.uniform(EVENT_SHOCK.min_magnitude, EVENT_SHOCK.max_magnitude)
        direction = random.choice([-1, 1])
        price = price * (1 + direction * magnitude)
    return max(price, 0.01)   # hard floor — prices can never go to zero
```

---

## Day-Open Reset

The day-open price is used by the frontend to compute the daily % change. It is reset to the current price at UTC midnight so sessions spanning midnight show a sensible daily change.

```python
from datetime import datetime, timezone

def should_reset_day_open(last_reset_date: str, now: datetime) -> bool:
    """Returns True if UTC date has changed since last_reset_date (YYYY-MM-DD)."""
    today = now.strftime("%Y-%m-%d")
    return today != last_reset_date
```

---

## Full Simulator Implementation

```python
# backend/market/simulator.py

import asyncio
import math
import random
from datetime import datetime, timezone
from dataclasses import dataclass

import numpy as np

from .interface import MarketDataProvider, PriceData


SECONDS_PER_YEAR   = 365.25 * 24 * 3600
TICK_INTERVAL      = 0.5      # seconds between GBM steps
PRICE_FLOOR        = 0.01     # hard minimum price

SECTOR_CORRELATIONS: dict[str, float] = {
    "tech":    0.6,
    "ev":      0.3,
    "finance": 0.5,
    "media":   0.3,
    "other":   0.0,
}


@dataclass
class _TickerState:
    config:         "TickerConfig"
    current_price:  float
    day_open_price: float
    day_open_date:  str           # YYYY-MM-DD UTC


@dataclass
class TickerConfig:
    seed_price: float
    mu:         float    # annual drift
    sigma:      float    # annual volatility
    sector:     str


TICKER_CONFIGS: dict[str, TickerConfig] = {
    "AAPL": TickerConfig(190.00, 0.07, 0.25, "tech"),
    "MSFT": TickerConfig(415.00, 0.08, 0.22, "tech"),
    "GOOGL":TickerConfig(175.00, 0.07, 0.24, "tech"),
    "META": TickerConfig(510.00, 0.09, 0.30, "tech"),
    "AMZN": TickerConfig(185.00, 0.08, 0.26, "tech"),
    "NVDA": TickerConfig(875.00, 0.12, 0.45, "tech"),
    "TSLA": TickerConfig(230.00, 0.05, 0.55, "ev"),
    "JPM":  TickerConfig(200.00, 0.06, 0.18, "finance"),
    "V":    TickerConfig(275.00, 0.07, 0.16, "finance"),
    "NFLX": TickerConfig(630.00, 0.06, 0.32, "media"),
}

DEFAULT_CONFIG = TickerConfig(100.00, 0.06, 0.28, "other")

EVENT_PROB     = 0.001     # per-ticker shock probability per tick
EVENT_MIN_MAG  = 0.02
EVENT_MAX_MAG  = 0.05


class SimulatorMarketDataProvider(MarketDataProvider):

    def __init__(self):
        self._price_cache: dict[str, PriceData] | None = None
        self._states:      dict[str, _TickerState]     = {}
        self._task:        asyncio.Task | None          = None

    async def start(self, price_cache: dict[str, PriceData]) -> None:
        self._price_cache = price_cache
        # Seed all tickers that are already in the cache (loaded from DB at startup)
        # plus any from TICKER_CONFIGS that are not yet in cache
        existing = set(price_cache.keys())
        for ticker in list(existing) + [t for t in TICKER_CONFIGS if t not in existing]:
            self._seed_ticker(ticker)

        self._task = asyncio.create_task(self._tick_loop(), name="simulator_tick")

    async def stop(self) -> None:
        if self._task:
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass

    async def add_ticker(self, ticker: str) -> None:
        if ticker not in self._states:
            self._seed_ticker(ticker)

    async def remove_ticker(self, ticker: str) -> None:
        # Do not remove from _states — the price might still be needed for open positions.
        # The SSE loop already broadcasts all cache entries; callers filter by watchlist.
        pass

    # ── Internal helpers ─────────────────────────────────────────────────────

    def _seed_ticker(self, ticker: str) -> None:
        config = TICKER_CONFIGS.get(ticker, DEFAULT_CONFIG)
        now    = datetime.now(timezone.utc)
        state  = _TickerState(
            config        = config,
            current_price = config.seed_price,
            day_open_price= config.seed_price,
            day_open_date = now.strftime("%Y-%m-%d"),
        )
        self._states[ticker] = state
        self._write_cache(ticker, state, previous_price=config.seed_price)

    def _write_cache(
        self, ticker: str, state: _TickerState, previous_price: float
    ) -> None:
        self._price_cache[ticker] = PriceData(
            ticker         = ticker,
            price          = state.current_price,
            previous_price = previous_price,
            day_open_price = state.day_open_price,
            updated_at     = datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ"),
        )

    async def _tick_loop(self) -> None:
        while True:
            await asyncio.sleep(TICK_INTERVAL)
            self._tick()

    def _tick(self) -> None:
        now    = datetime.now(timezone.utc)
        today  = now.strftime("%Y-%m-%d")
        dt     = TICK_INTERVAL

        # Draw one sector shock per active sector
        active_sectors = {s.config.sector for s in self._states.values()}
        sector_shocks  = {sec: np.random.standard_normal() for sec in active_sectors}

        for ticker, state in list(self._states.items()):
            # Day-open reset at UTC midnight
            if state.day_open_date != today:
                state.day_open_price = state.current_price
                state.day_open_date  = today

            cfg = state.config
            mu_s    = cfg.mu    / SECONDS_PER_YEAR
            sigma_s = cfg.sigma / math.sqrt(SECONDS_PER_YEAR)

            # Correlated GBM step
            rho    = SECTOR_CORRELATIONS.get(cfg.sector, 0.0)
            z_mkt  = sector_shocks.get(cfg.sector, 0.0)
            z_idio = np.random.standard_normal()
            z      = rho * z_mkt + math.sqrt(max(1 - rho**2, 0)) * z_idio

            new_price = state.current_price * math.exp(
                (mu_s - 0.5 * sigma_s**2) * dt + sigma_s * math.sqrt(dt) * z
            )

            # Random event shock
            if random.random() < EVENT_PROB:
                magnitude = random.uniform(EVENT_MIN_MAG, EVENT_MAX_MAG)
                direction = random.choice([-1, 1])
                new_price = new_price * (1 + direction * magnitude)

            # Enforce price floor
            new_price = max(new_price, PRICE_FLOOR)

            previous = state.current_price
            state.current_price = new_price
            self._write_cache(ticker, state, previous_price=previous)
```

---

## Adding Unknown Tickers Dynamically

When a user adds a ticker that is not in `TICKER_CONFIGS` (e.g., `PYPL`, `RBLX`, `SNAP`), the simulator creates a `DEFAULT_CONFIG` entry with a $100 seed price and median volatility parameters. The ticker immediately begins simulating and appears in the next SSE broadcast.

```python
async def add_ticker(self, ticker: str) -> None:
    if ticker not in self._states:
        self._seed_ticker(ticker)   # uses DEFAULT_CONFIG for unknown tickers
```

To add a known ticker with better-calibrated parameters, extend `TICKER_CONFIGS`.

---

## Anomaly Detection Hooks

The simulator must emit `anomaly_detected` events for abnormal price jumps. The tick loop checks each price update against the 10%-in-1-second rule.

```python
def _check_price_anomaly(
    ticker: str, old_price: float, new_price: float, dt: float
) -> bool:
    """Returns True if the price moved more than 10% within 1 second."""
    if dt <= 0 or old_price <= 0:
        return False
    pct_change = abs(new_price - old_price) / old_price
    # Scale to 1-second window: if dt=0.5s and move is 10%+, flag it
    return pct_change > 0.10 * (dt / 1.0)
```

When `_check_price_anomaly` returns `True`, the tick loop emits an `anomaly_detected` event to the event log (see `PLAN.md §13.4`). The event shock configuration (`EVENT_MAX_MAG = 0.05`) keeps normal shocks below this threshold; only bugs or extreme parameter values should trigger it.

---

## Simulator vs Massive: Behavioral Differences

| Behaviour | Simulator | Massive |
|-----------|-----------|---------|
| Data freshness | Synthetic, ~500 ms ticks | Real market prices (delayed or live depending on plan) |
| After-hours prices | Continues simulating 24/7 | Returns last known price outside market hours |
| Unknown tickers | Accepted, simulated at default params | Accepted but shows no price until next poll |
| Price floor | Enforced at $0.01 | Not applicable (real prices don't go to zero) |
| Day-open reset | At UTC midnight | Pulled from Massive `day.o` field each poll |
| Correlation | Sector-based GBM factor model | Real market correlation (implicit in actual prices) |
| Network dependency | None | Requires MASSIVE_API_KEY and internet access |

---

## Testing the Simulator

Unit tests should verify:

1. **GBM math**: `gbm_step` produces prices within expected statistical bounds over N steps.
2. **Price floor**: A deliberate large negative shock never produces a price < $0.01.
3. **Shock constraint**: `EVENT_MAX_MAG = 0.05` is below the 10%-anomaly threshold — event shocks do not trigger false anomaly events.
4. **Day-open reset**: `should_reset_day_open` flips on a date change and not before.
5. **add_ticker**: Calling `add_ticker("UNKNOWN")` populates the price cache within one tick interval.
6. **Correlation**: Over 1000 ticks, tech stocks show positive pairwise correlation significantly above zero.

```python
# backend/tests/test_simulator.py (excerpt)

import pytest
from backend.market.simulator import SimulatorMarketDataProvider, gbm_step, PRICE_FLOOR

def test_gbm_never_goes_negative():
    price = 100.0
    for _ in range(10_000):
        price = gbm_step(price, mu=0.0, sigma=2.0, dt=0.5)   # extreme vol
        assert price >= PRICE_FLOOR

def test_shock_below_anomaly_threshold():
    from backend.market.simulator import EVENT_MAX_MAG
    # 5% max shock in 0.5s: annualised to 1s = 5% * (0.5/1) still < 10% threshold
    assert EVENT_MAX_MAG < 0.10, "Event shocks must not exceed anomaly detection threshold"
```
