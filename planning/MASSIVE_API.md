# Massive API Reference (formerly Polygon.io)

Reference documentation for the Massive (formerly Polygon.io) REST API as used in FinAlly.

## Overview

- **Base URL**: `https://api.massive.com` (legacy `https://api.polygon.io` still supported)
- **Python package**: `massive` (install via `uv add massive`)
- **Min Python version**: 3.9+
- **Auth**: API key passed to `RESTClient(api_key=...)` or read from the `MASSIVE_API_KEY` environment variable
- **Auth header**: `Authorization: Bearer <API_KEY>` (handled automatically by the client)

## Rate Limits

| Tier | Limit | Recommended Poll Interval |
|------|-------|--------------------------|
| Free | 5 requests/minute | Every 15 seconds |
| Paid (all tiers) | Unlimited (stay under ~100 req/s) | Every 2–5 seconds |

FinAlly polls on a timer. The default `poll_interval` in `MassiveDataSource` is `15.0` seconds (free tier). Set it lower if you have a paid key.

## Client Initialization

```python
from massive import RESTClient

# Reads MASSIVE_API_KEY from environment automatically
client = RESTClient()

# Or pass explicitly
client = RESTClient(api_key="your_key_here")
```

The `RESTClient` is **synchronous**. In an async context (FastAPI), wrap calls with `asyncio.to_thread()` to avoid blocking the event loop:

```python
import asyncio
from massive import RESTClient

client = RESTClient(api_key=api_key)

snapshots = await asyncio.to_thread(
    client.get_snapshot_all,
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "GOOGL"],
)
```

## Endpoints Used in FinAlly

### 1. Snapshot — All Tickers (Primary Endpoint)

Gets current prices for multiple tickers in a **single API call**. This is the only endpoint FinAlly needs for live price polling.

**REST**: `GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,GOOGL,MSFT`

**Python client**:
```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient()

snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA"],
)

for snap in snapshots:
    price = snap.last_trade.price
    prev_close = snap.day.previous_close
    timestamp_ms = snap.last_trade.timestamp  # Unix milliseconds
    print(f"{snap.ticker}: ${price} (prev close: ${prev_close})")
```

**Response structure** (per ticker):
```json
{
  "ticker": "AAPL",
  "day": {
    "open": 129.61,
    "high": 130.15,
    "low": 125.07,
    "close": 125.07,
    "volume": 111237700,
    "volume_weighted_average_price": 127.35,
    "previous_close": 129.61,
    "change": -4.54,
    "change_percent": -3.50
  },
  "last_trade": {
    "price": 125.07,
    "size": 100,
    "exchange": "XNYS",
    "timestamp": 1675190399000
  },
  "last_quote": {
    "bid_price": 125.06,
    "ask_price": 125.08,
    "bid_size": 500,
    "ask_size": 1000,
    "spread": 0.02,
    "timestamp": 1675190399500
  },
  "prev_daily_bar": { "...": "previous day OHLCV" },
  "minute_volume": { "...": "volume per minute" }
}
```

**Fields extracted by FinAlly**:
- `snap.last_trade.price` — current price used for display and trade execution
- `snap.last_trade.timestamp` — Unix **milliseconds**; divide by 1000 to get Unix seconds
- `snap.day.previous_close` — for day change calculation (currently not used in PriceCache but available)

### 2. Single Ticker Snapshot

For fetching detailed data on one specific ticker.

```python
from massive.rest.models import SnapshotMarketType

snapshot = client.get_snapshot_ticker(
    market_type=SnapshotMarketType.STOCKS,
    ticker="AAPL",
)

print(f"Price: ${snapshot.last_trade.price}")
print(f"Bid/Ask: ${snapshot.last_quote.bid_price} / ${snapshot.last_quote.ask_price}")
print(f"Day range: ${snapshot.day.low} - ${snapshot.day.high}")
```

### 3. Previous Close

Gets the prior trading day's OHLCV bar. Useful for seeding prices or showing day change from open.

**REST**: `GET /v2/aggs/ticker/{ticker}/prev`

```python
prev = client.get_previous_close_agg(ticker="AAPL")
for agg in prev:
    print(f"Previous close: ${agg.close}")
    print(f"OHLC: O={agg.open} H={agg.high} L={agg.low} C={agg.close}")
    print(f"Volume: {agg.volume}")
```

### 4. Aggregates (Historical Bars)

OHLCV bars over a date range. Not used in the current FinAlly polling loop but useful for historical charts.

**REST**: `GET /v2/aggs/ticker/{ticker}/range/{multiplier}/{timespan}/{from}/{to}`

```python
for agg in client.list_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="day",
    from_="2024-01-01",
    to="2024-01-31",
    limit=50000,
):
    print(f"t={agg.timestamp} O={agg.open} H={agg.high} L={agg.low} C={agg.close} V={agg.volume}")
```

Timestamps in response bars are also Unix milliseconds.

### 5. Last Trade / Last Quote

Individual endpoints for the most recent trade or NBBO quote; less efficient than the snapshot endpoint for multiple tickers.

```python
trade = client.get_last_trade(ticker="AAPL")
print(f"Last trade: ${trade.price} x {trade.size}")

quote = client.get_last_quote(ticker="AAPL")
print(f"Bid: ${quote.bid} x {quote.bid_size}")
print(f"Ask: ${quote.ask} x {quote.ask_size}")
```

## How FinAlly Uses the API

`MassiveDataSource` runs a background asyncio task:

1. Collects all currently-watched tickers
2. Calls `get_snapshot_all()` in a thread pool (synchronous client, non-blocking)
3. Extracts `last_trade.price` and `last_trade.timestamp` from each snapshot
4. Converts timestamp from milliseconds to seconds
5. Writes to the shared `PriceCache`
6. Sleeps `poll_interval` seconds, then repeats

An immediate first poll fires in `start()` so the cache is populated before the first SSE push.

```python
# From backend/app/market/massive_client.py (simplified)
async def _poll_once(self) -> None:
    if not self._tickers or not self._client:
        return
    try:
        snapshots = await asyncio.to_thread(self._fetch_snapshots)
        for snap in snapshots:
            try:
                price = snap.last_trade.price
                timestamp = snap.last_trade.timestamp / 1000.0  # ms → seconds
                self._cache.update(ticker=snap.ticker, price=price, timestamp=timestamp)
            except (AttributeError, TypeError) as e:
                logger.warning("Skipping snapshot for %s: %s", getattr(snap, "ticker", "???"), e)
    except Exception as e:
        logger.error("Massive poll failed: %s", e)
        # Don't re-raise — the loop retries on the next interval

def _fetch_snapshots(self) -> list:
    return self._client.get_snapshot_all(
        market_type=SnapshotMarketType.STOCKS,
        tickers=self._tickers,
    )
```

## Error Handling

The client raises exceptions for HTTP errors. `MassiveDataSource` catches all exceptions in `_poll_once()` and logs them without re-raising, so transient failures (network hiccup, rate limit burst) don't crash the background task — the next poll will retry.

Common exceptions:
- **401** — Invalid API key (`MASSIVE_API_KEY` wrong or missing)
- **403** — Endpoint not available on current plan
- **429** — Rate limit exceeded (free tier: 5 req/min)
- **5xx** — Server errors; the client has built-in retry (3 attempts by default)

Per-snapshot `AttributeError` / `TypeError` (malformed snapshot missing `last_trade`) are caught individually so one bad ticker doesn't abort the entire poll.

## Notes

- The snapshot endpoint returns all requested tickers in **one API call** — critical for staying within the free tier's 5 req/min limit regardless of watchlist size
- Timestamps from the API are Unix **milliseconds**; `PriceCache.update()` takes Unix **seconds**
- During market-closed hours, `last_trade.price` is the last traded price (may include after-hours / pre-market)
- The `day` object resets at market open; during pre-market, change values may reflect the previous session
- When a ticker is added to the watchlist via `add_ticker()`, it will appear in the price cache after the next poll cycle (up to `poll_interval` seconds)
