# Market Data Interface

Unified Python API for market data in FinAlly. Two implementations — a GBM simulator (default) and a Massive REST poller (when `MASSIVE_API_KEY` is set) — behind one abstract interface. All downstream code (SSE streaming, portfolio valuation, trade execution) is source-agnostic.

## Module Location

```
backend/app/market/
├── __init__.py          # Public re-exports
├── models.py            # PriceUpdate dataclass
├── interface.py         # MarketDataSource ABC
├── cache.py             # PriceCache
├── factory.py           # create_market_data_source()
├── simulator.py         # SimulatorDataSource + GBMSimulator
├── massive_client.py    # MassiveDataSource
├── seed_prices.py       # Seed prices and per-ticker GBM params
└── stream.py            # FastAPI SSE endpoint
```

## Public API

Import from `app.market`:

```python
from app.market import (
    PriceUpdate,                  # Data model
    PriceCache,                   # Shared price store
    MarketDataSource,             # Abstract interface
    create_market_data_source,    # Factory
    create_stream_router,         # SSE endpoint factory
)
```

---

## PriceUpdate

Immutable frozen dataclass representing a single ticker's price at a point in time.

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float          # Unix seconds (default: time.time())

    # Computed properties (not stored fields)
    @property
    def change(self) -> float: ...          # price - previous_price, rounded to 4dp
    @property
    def change_percent(self) -> float: ...  # % change, rounded to 4dp
    @property
    def direction(self) -> str: ...         # "up", "down", or "flat"

    def to_dict(self) -> dict: ...          # Serialized for JSON / SSE
```

`to_dict()` output:
```json
{
  "ticker": "AAPL",
  "price": 190.50,
  "previous_price": 190.25,
  "timestamp": 1714200000.0,
  "change": 0.25,
  "change_percent": 0.1314,
  "direction": "up"
}
```

On the first update for a ticker, `previous_price == price` and `direction == "flat"`.

---

## PriceCache

Thread-safe in-memory store. One writer (the active data source), multiple readers (SSE, portfolio, trades).

```python
class PriceCache:
    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate
    def get(self, ticker: str) -> PriceUpdate | None
    def get_all(self) -> dict[str, PriceUpdate]    # Shallow copy
    def get_price(self, ticker: str) -> float | None
    def remove(self, ticker: str) -> None

    @property
    def version(self) -> int    # Monotonic counter; increments on every update
```

The `version` counter is used by the SSE generator for change detection — it only emits an event when `version` has advanced since the last check, avoiding redundant pushes when prices haven't changed.

Usage in downstream code:

```python
# Read one price
price = cache.get_price("AAPL")     # float or None

# Read full update (includes direction, change_percent, etc.)
update = cache.get("AAPL")          # PriceUpdate or None

# Read all current prices (for SSE or portfolio valuation)
all_prices = cache.get_all()        # dict[str, PriceUpdate]
```

---

## MarketDataSource (Abstract Interface)

```python
from abc import ABC, abstractmethod

class MarketDataSource(ABC):

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates. Call exactly once."""

    @abstractmethod
    async def stop(self) -> None:
        """Stop the background task and release resources. Safe to call multiple times."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the active set. No-op if already present."""

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker from the active set. Also removes from PriceCache."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return the current list of actively tracked tickers."""
```

Both implementations write to the shared `PriceCache` on their own schedule. The interface does not return prices directly.

---

## Factory

```python
# backend/app/market/factory.py

def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        return SimulatorDataSource(price_cache=price_cache)
```

Selection logic: if `MASSIVE_API_KEY` is set and non-empty → `MassiveDataSource`; otherwise → `SimulatorDataSource`.

---

## MassiveDataSource

Polls the Massive REST API every `poll_interval` seconds (default: 15.0 for free tier).

```python
class MassiveDataSource(MarketDataSource):
    def __init__(
        self,
        api_key: str,
        price_cache: PriceCache,
        poll_interval: float = 15.0,
    ) -> None: ...
```

Key behaviors:
- Fires an **immediate first poll** in `start()` so the cache is populated before the first SSE push
- Subsequent polls happen after each `poll_interval` sleep
- The synchronous `RESTClient` runs in `asyncio.to_thread()` to avoid blocking the event loop
- Per-snapshot errors (malformed API response for one ticker) are logged and skipped; the poll continues for remaining tickers
- Poll failures (network error, 429, etc.) are logged but not re-raised; the loop retries on the next interval
- New tickers added via `add_ticker()` appear in the cache after the next poll cycle

See `MASSIVE_API.md` for full API documentation.

---

## SimulatorDataSource

Wraps `GBMSimulator` in an async loop, calling `step()` every `update_interval` seconds (default: 0.5).

```python
class SimulatorDataSource(MarketDataSource):
    def __init__(
        self,
        price_cache: PriceCache,
        update_interval: float = 0.5,
        event_probability: float = 0.001,
    ) -> None: ...
```

Key behaviors:
- Creates a `GBMSimulator` in `start()` and seeds the cache with initial prices immediately
- New tickers added via `add_ticker()` are seeded into the cache right away (no wait for next step)
- `remove_ticker()` removes from both the simulator and the cache
- Exceptions in the step loop are logged and swallowed; the loop continues

See `MARKET_SIMULATOR.md` for full GBM math and simulator design.

---

## SSE Integration

```python
# backend/app/market/stream.py

def create_stream_router(price_cache: PriceCache) -> APIRouter:
    """Returns a FastAPI router with GET /api/stream/prices."""
```

The SSE generator:
1. Sends a `retry: 1000` directive (browser auto-reconnects after 1s on disconnect)
2. Checks `price_cache.version` every 500ms
3. Only emits an event when the version has advanced (new prices exist)
4. Stops when the client disconnects (detected via `request.is_disconnected()`)

Each event payload is a JSON object keyed by ticker:

```
data: {"AAPL": {"ticker":"AAPL","price":190.5,...}, "GOOGL": {...}, ...}
```

**Frontend consumption**:
```js
const es = new EventSource("/api/stream/prices");
es.onmessage = (event) => {
    const prices = JSON.parse(event.data);  // { AAPL: PriceUpdate, ... }
};
```

---

## Lifecycle

```python
from app.market import PriceCache, create_market_data_source, create_stream_router

# App startup
cache = PriceCache()
source = create_market_data_source(cache)
await source.start(["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA", "NVDA", "META", "JPM", "V", "NFLX"])

# Register SSE router
router = create_stream_router(cache)
app.include_router(router)

# Dynamic watchlist changes
await source.add_ticker("PYPL")
await source.remove_ticker("NFLX")

# Read prices (any time after start)
update = cache.get("AAPL")           # PriceUpdate or None
price  = cache.get_price("AAPL")     # float or None
all_p  = cache.get_all()             # dict[str, PriceUpdate]

# App shutdown
await source.stop()
```

---

## Architecture Diagram

```
                           ┌─────────────────────────────┐
                           │  MarketDataSource (ABC)      │
                           │  start / stop                │
                           │  add_ticker / remove_ticker  │
                           └──────────────┬──────────────┘
                        ┌─────────────────┴──────────────────┐
                        │                                    │
              ┌─────────▼────────┐               ┌──────────▼──────────┐
              │ SimulatorDataSource│              │  MassiveDataSource  │
              │ GBM step every 0.5s│              │  REST poll every 15s│
              └─────────┬────────┘               └──────────┬──────────┘
                        │                                   │
                        └──────────────┬────────────────────┘
                                       │ cache.update()
                               ┌───────▼──────────┐
                               │    PriceCache     │
                               │  (thread-safe)    │
                               └───────┬──────────┘
                     ┌─────────────────┼──────────────────────┐
                     │                 │                      │
             ┌───────▼──────┐  ┌───────▼──────┐   ┌──────────▼──────┐
             │  SSE stream  │  │  Portfolio   │   │  Trade exec     │
             │  /api/stream │  │  valuation   │   │  current price  │
             └──────────────┘  └──────────────┘   └─────────────────┘
```
