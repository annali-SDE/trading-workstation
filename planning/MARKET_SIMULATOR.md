# Market Simulator Design

Approach and code structure for simulating realistic stock prices when no `MASSIVE_API_KEY` is configured.

## Overview

The simulator uses **Geometric Brownian Motion (GBM)** to generate realistic stock price paths. GBM is the standard model underlying Black-Scholes option pricing — prices evolve continuously with multiplicative random noise, can never go negative, and produce the lognormal distribution observed in real equity markets.

Updates run at 500ms intervals, producing a continuous stream of price changes that feel live and natural.

The simulator consists of two classes:

- **`GBMSimulator`** — Pure math engine. Takes a list of tickers, generates correlated random price steps.
- **`SimulatorDataSource`** — `MarketDataSource` implementation. Wraps `GBMSimulator` in an async loop, writes results to the `PriceCache` every 500ms.

---

## GBM Math

At each time step, a stock price evolves as:

```
S(t + dt) = S(t) * exp((mu - sigma²/2) * dt + sigma * sqrt(dt) * Z)
```

Where:
- `S(t)` = current price
- `mu` = annualized drift (expected return), e.g. `0.05` = 5%
- `sigma` = annualized volatility, e.g. `0.22` = 22%
- `dt` = time step as a fraction of a trading year
- `Z` = standard normal random variable drawn from N(0,1)

The `exp()` formulation keeps prices strictly positive regardless of how extreme `Z` gets.

### Time Step Calibration

For 500ms tick intervals:

```python
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800 seconds
DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR   # ~8.48e-8
```

This tiny `dt` produces sub-cent moves per tick that accumulate naturally. With `sigma=0.22` (AAPL), the expected intraday range over a 6.5-hour session is:

```
sigma * sqrt(dt_per_day) ≈ 0.22 * sqrt(1/252) ≈ 1.4%
```

Which matches typical AAPL daily ranges.

---

## Correlated Moves

Real stocks don't move independently — tech stocks tend to move together when the market rallies or sells off. We use **Cholesky decomposition** of a correlation matrix to generate correlated random draws.

Given a correlation matrix `C`, compute its lower-triangular Cholesky factor `L = cholesky(C)`.  
For `n` independent standard normal draws `Z_ind ~ N(0, I)`:

```
Z_corr = L @ Z_ind
```

`Z_corr` then has the covariance structure defined by `C`.

### Correlation Structure

```python
# From seed_prices.py
CORRELATION_GROUPS = {
    "tech":    {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

INTRA_TECH_CORR    = 0.6   # Tech stocks move together
INTRA_FINANCE_CORR = 0.5   # Finance stocks move together
CROSS_GROUP_CORR   = 0.3   # Between sectors or unknown tickers
TSLA_CORR          = 0.3   # TSLA does its own thing (even within tech)
```

TSLA is technically in the `tech` group but gets the `TSLA_CORR` override in pairwise computation, reflecting its idiosyncratic behavior.

The Cholesky matrix is rebuilt (`_rebuild_cholesky()`) whenever tickers are added or removed. This is O(n²) but n is always small (< 50 tickers).

---

## Seed Prices and Per-Ticker Parameters

```python
# backend/app/market/seed_prices.py

SEED_PRICES: dict[str, float] = {
    "AAPL":  190.00,
    "GOOGL": 175.00,
    "MSFT":  420.00,
    "AMZN":  185.00,
    "TSLA":  250.00,
    "NVDA":  800.00,
    "META":  500.00,
    "JPM":   195.00,
    "V":     280.00,
    "NFLX":  600.00,
}

TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL":  {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT":  {"sigma": 0.20, "mu": 0.05},
    "AMZN":  {"sigma": 0.28, "mu": 0.05},
    "TSLA":  {"sigma": 0.50, "mu": 0.03},  # High vol
    "NVDA":  {"sigma": 0.40, "mu": 0.08},  # High vol, strong drift
    "META":  {"sigma": 0.30, "mu": 0.05},
    "JPM":   {"sigma": 0.18, "mu": 0.04},  # Low vol (bank)
    "V":     {"sigma": 0.17, "mu": 0.04},  # Low vol (payments)
    "NFLX":  {"sigma": 0.35, "mu": 0.05},
}

DEFAULT_PARAMS: dict[str, float] = {"sigma": 0.25, "mu": 0.05}
```

Tickers added dynamically that are not in `SEED_PRICES` start at a random price in `[50.0, 300.0]`. Tickers not in `TICKER_PARAMS` use `DEFAULT_PARAMS`.

---

## Random Shock Events

Every step, each ticker has a small independent chance of a sudden large move:

```python
event_probability = 0.001   # 0.1% per ticker per tick

if random.random() < event_probability:
    shock_magnitude = random.uniform(0.02, 0.05)   # 2–5% move
    shock_sign = random.choice([-1, 1])
    price *= (1 + shock_magnitude * shock_sign)
```

With 10 tickers at 2 ticks/second and `event_probability=0.001`:

```
Expected events per second = 10 tickers × 2 ticks/s × 0.001 = 0.02/s
Expected interval between events = 50 seconds
```

This means roughly one dramatic flash somewhere in the watchlist every ~50 seconds, which is enough to keep the UI visually interesting without feeling chaotic.

---

## GBMSimulator Implementation

```python
# backend/app/market/simulator.py

class GBMSimulator:
    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600   # 5,896,800
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR   # ~8.48e-8

    def __init__(
        self,
        tickers: list[str],
        dt: float = DEFAULT_DT,
        event_probability: float = 0.001,
    ) -> None:
        self._dt = dt
        self._event_prob = event_probability
        self._tickers: list[str] = []
        self._prices: dict[str, float] = {}
        self._params: dict[str, dict[str, float]] = {}
        self._cholesky: np.ndarray | None = None

        for ticker in tickers:
            self._add_ticker_internal(ticker)   # batch — no Cholesky rebuild per ticker
        self._rebuild_cholesky()

    def step(self) -> dict[str, float]:
        """Advance all tickers one time step. Returns {ticker: new_price}."""
        n = len(self._tickers)
        if n == 0:
            return {}

        z_independent = np.random.standard_normal(n)
        z_correlated = self._cholesky @ z_independent if self._cholesky is not None else z_independent

        result = {}
        for i, ticker in enumerate(self._tickers):
            mu = self._params[ticker]["mu"]
            sigma = self._params[ticker]["sigma"]

            drift     = (mu - 0.5 * sigma**2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * z_correlated[i]
            self._prices[ticker] *= math.exp(drift + diffusion)

            if random.random() < self._event_prob:
                shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
                self._prices[ticker] *= (1 + shock)

            result[ticker] = round(self._prices[ticker], 2)
        return result

    def add_ticker(self, ticker: str) -> None:
        """Add a ticker mid-session. Rebuilds the correlation matrix."""
        if ticker in self._prices:
            return
        self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker mid-session. Rebuilds the correlation matrix."""
        if ticker not in self._prices:
            return
        self._tickers.remove(ticker)
        del self._prices[ticker]
        del self._params[ticker]
        self._rebuild_cholesky()

    def get_price(self, ticker: str) -> float | None:
        return self._prices.get(ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    def _add_ticker_internal(self, ticker: str) -> None:
        self._tickers.append(ticker)
        self._prices[ticker] = SEED_PRICES.get(ticker, random.uniform(50.0, 300.0))
        self._params[ticker] = TICKER_PARAMS.get(ticker, dict(DEFAULT_PARAMS))

    def _rebuild_cholesky(self) -> None:
        n = len(self._tickers)
        if n <= 1:
            self._cholesky = None
            return
        corr = np.eye(n)
        for i in range(n):
            for j in range(i + 1, n):
                rho = self._pairwise_correlation(self._tickers[i], self._tickers[j])
                corr[i, j] = rho
                corr[j, i] = rho
        self._cholesky = np.linalg.cholesky(corr)

    @staticmethod
    def _pairwise_correlation(t1: str, t2: str) -> float:
        tech    = CORRELATION_GROUPS["tech"]
        finance = CORRELATION_GROUPS["finance"]
        if t1 == "TSLA" or t2 == "TSLA":
            return TSLA_CORR
        if t1 in tech and t2 in tech:
            return INTRA_TECH_CORR
        if t1 in finance and t2 in finance:
            return INTRA_FINANCE_CORR
        return CROSS_GROUP_CORR
```

---

## SimulatorDataSource Implementation

```python
class SimulatorDataSource(MarketDataSource):
    def __init__(
        self,
        price_cache: PriceCache,
        update_interval: float = 0.5,
        event_probability: float = 0.001,
    ) -> None:
        self._cache = price_cache
        self._interval = update_interval
        self._event_prob = event_probability
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)
        # Seed the cache immediately so SSE has data on first push
        for ticker in tickers:
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None

    async def add_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.add_ticker(ticker)
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)  # Immediate seed

    async def remove_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)

    def get_tickers(self) -> list[str]:
        return self._sim.get_tickers() if self._sim else []

    async def _run_loop(self) -> None:
        while True:
            try:
                if self._sim:
                    prices = self._sim.step()
                    for ticker, price in prices.items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")
            await asyncio.sleep(self._interval)
```

---

## File Structure

```
backend/
  app/
    market/
      seed_prices.py    # SEED_PRICES, TICKER_PARAMS, DEFAULT_PARAMS, correlation constants
      simulator.py      # GBMSimulator + SimulatorDataSource
```

`seed_prices.py` holds only constants (no logic). `simulator.py` imports from it and contains both classes.

---

## Behavioral Properties

| Property | Detail |
|---|---|
| Price floor | None needed — `exp()` is always positive, prices can never reach zero |
| Per-tick move magnitude | Sub-cent at `DEFAULT_DT`; accumulates to ~1–3% intraday range depending on sigma |
| Event frequency | ~1 shock event across the watchlist every 50 seconds (10 tickers × default params) |
| Cholesky rebuild cost | O(n²) where n = number of tracked tickers; n < 50 in practice |
| Dynamic ticker addition | Immediate cache seed + Cholesky rebuild; new ticker visible in SSE on next step |
| Dynamic ticker removal | Immediate cache removal + Cholesky rebuild; stops appearing in SSE immediately |
| Unknown tickers | Seed price: random `[50, 300]`; params: `DEFAULT_PARAMS` (sigma=0.25, mu=0.05) |
