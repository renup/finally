# Market Simulator — Design

Design reference for the GBM-based market simulator: the default market data source
(used whenever `MASSIVE_API_KEY` is unset). Implements the same `MarketDataSource`
interface as the Massive-backed source (see `MARKET_INTERFACE.md`), so this document
covers only what's specific to the simulator: the price model, correlation structure,
and the code that generates a continuous stream of plausible, correlated stock prices
with zero external dependencies.

**Status: implemented.** `backend/app/market/simulator.py` +
`backend/app/market/seed_prices.py`. A terminal demo of it is runnable via
`uv run market_data_demo.py` (`backend/market_data_demo.py`).

---

## 1. Goals

- Plausible-looking price action with no network calls and no API key — the default,
  zero-config path (`PLAN.md` §6).
- Tickers in the same sector move together, the way real stocks do, rather than as
  independent random walks.
- Occasional dramatic moves ("events") for a livelier demo.
- Cheap enough to step at 500ms for an arbitrary number of tickers, since it's the hot
  path of the whole live-updating UI.
- Dynamic ticker set: adding/removing a ticker from the watchlist must be reflected
  without restarting the simulation.

## 2. The Price Model: Geometric Brownian Motion

Each ticker's price follows GBM, the standard log-normal random walk used for
short-horizon equity price simulation:

```
S(t+dt) = S(t) * exp((μ - σ²/2) * dt + σ * √dt * Z)
```

| Symbol | Meaning |
|---|---|
| `S(t)` | current price |
| `μ` (mu) | annualized drift (expected return) |
| `σ` (sigma) | annualized volatility |
| `dt` | time step, as a fraction of a trading year |
| `Z` | a (correlated — see §3) standard normal random draw |

### Why this model

- It's the textbook choice for equity price simulation over short horizons — prices stay
  positive (log-normal), and returns are the natural i.i.d.-normal quantity to add
  correlation and volatility structure to.
- `μ` and `σ` are exactly "expected annual return" and "annualized volatility," so the
  seed parameters in `seed_prices.py` (§4) can be picked to reflect a ticker's real-world
  character (TSLA/NVDA volatile with high `σ`; JPM/V calmer) using intuitive numbers.

### The tick size

```python
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800
DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR   # ~8.48e-8, for a 500ms tick
```

252 trading days × 6.5 trading hours/day × 3600s. A 500ms step is therefore a tiny
fraction of a trading year, which is the point: at this `dt`, each individual tick moves
the price by a sub-cent amount, and those tiny moves *compound* into realistic-looking
minute-to-minute and hour-to-hour volatility — the simulator never needs to reason about
"today's move" directly, only about the next 500ms, and realistic-scale movement emerges
from compounding.

### Random events

Independent of the GBM step, each ticker has a small per-tick chance
(`event_probability`, default `0.001` = 0.1%) of an additional sudden 2–5% shock, sign
chosen at random:

```python
if random.random() < self._event_prob:
    shock_magnitude = random.uniform(0.02, 0.05)
    shock_sign = random.choice([-1, 1])
    self._prices[ticker] *= 1 + shock_magnitude * shock_sign
```

At 10 tickers × 2 ticks/sec, this produces roughly one dramatic move somewhere in the
watchlist every ~50 seconds — enough to give the demo "news event" drama without
dominating normal price action.

## 3. Correlation Structure

Real stocks in the same sector move together (tech stocks rally or sell off as a group;
a bank's move is correlated with other banks). The simulator reproduces this with a
Cholesky-decomposition approach rather than treating each ticker as an independent
random walk:

1. Draw `n` independent standard-normal values, one per ticker: `Z_independent`.
2. Multiply by the Cholesky factor `L` of the ticker correlation matrix:
   `Z_correlated = L @ Z_independent`.
3. Feed each ticker's entry in `Z_correlated` — now correlated with the others per the
   matrix — into its own GBM step.

```python
z_independent = np.random.standard_normal(n)
z_correlated = self._cholesky @ z_independent if self._cholesky is not None else z_independent
```

This is standard technique for simulating correlated multivariate normals: if `Σ = L
Lᵀ` (Cholesky factorization of the target correlation matrix `Σ`), then `L @
Z_independent` has exactly covariance `Σ`. Rebuilding `L` is `O(n²)`-ish (Cholesky of an
`n×n` matrix) but `n` here is watchlist-sized (< 50), so it's cheap enough to redo
on every `add_ticker`/`remove_ticker` rather than maintaining it incrementally.

### Pairwise correlation rule (`_pairwise_correlation`)

```python
if t1 == "TSLA" or t2 == "TSLA":
    return TSLA_CORR            # 0.3 — TSLA does its own thing
if t1 in tech and t2 in tech:
    return INTRA_TECH_CORR      # 0.6 — tech stocks move together
if t1 in finance and t2 in finance:
    return INTRA_FINANCE_CORR   # 0.5 — finance stocks move together
return CROSS_GROUP_CORR         # 0.3 — cross-sector / unknown tickers
```

Sector membership (`seed_prices.py`):

```python
CORRELATION_GROUPS = {
    "tech":    {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}
```

TSLA is deliberately excluded from the "tech" correlation bucket even though it's
grouped there for the watchlist — its price action in this simulator is intentionally
more idiosyncratic than the rest of the tech cluster. Tickers outside both named sets
(e.g., one the user adds manually via the watchlist) default to `CROSS_GROUP_CORR` (0.3)
against everything, and to `DEFAULT_PARAMS` (`σ=0.25, μ=0.05`) for their own GBM
parameters.

## 4. Seed Data (`seed_prices.py`)

Starting prices and per-ticker GBM parameters for the default watchlist:

```python
SEED_PRICES = {
    "AAPL": 190.00, "GOOGL": 175.00, "MSFT": 420.00, "AMZN": 185.00, "TSLA": 250.00,
    "NVDA": 800.00, "META": 500.00, "JPM": 195.00, "V": 280.00, "NFLX": 600.00,
}

TICKER_PARAMS = {
    "AAPL": {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT": {"sigma": 0.20, "mu": 0.05},
    "AMZN": {"sigma": 0.28, "mu": 0.05},
    "TSLA": {"sigma": 0.50, "mu": 0.03},   # high volatility
    "NVDA": {"sigma": 0.40, "mu": 0.08},   # high volatility, strong drift
    "META": {"sigma": 0.30, "mu": 0.05},
    "JPM":  {"sigma": 0.18, "mu": 0.04},   # low volatility (bank)
    "V":    {"sigma": 0.17, "mu": 0.04},   # low volatility (payments)
    "NFLX": {"sigma": 0.35, "mu": 0.05},
}

DEFAULT_PARAMS = {"sigma": 0.25, "mu": 0.05}  # for any ticker not listed above
```

A ticker added at runtime that isn't in `TICKER_PARAMS`/`SEED_PRICES` gets
`DEFAULT_PARAMS` and a random seed price in `[50, 300)` — the simulator never rejects an
unknown ticker, it just gives it generic, moderate behavior.

## 5. Code Structure

```
GBMSimulator            # Pure computation: correlated GBM price stepping
├── step() -> dict[str, float]              # advance all tickers by one dt, hot path
├── add_ticker(ticker) / remove_ticker(t)    # mutate tracked set, rebuild Cholesky
├── get_price(ticker) / get_tickers()
└── _pairwise_correlation(t1, t2)            # sector-based correlation lookup

SimulatorDataSource(MarketDataSource)  # Adapter: wraps GBMSimulator in the interface
├── start(tickers)   # construct GBMSimulator, seed cache, spawn asyncio.Task loop
├── stop()            # cancel the task, idempotent
├── add_ticker(t) / remove_ticker(t)   # delegate to GBMSimulator, keep cache in sync
├── get_tickers()
└── _run_loop()        # while True: step() -> cache.update(...) each ticker; sleep(interval)
```

The split matters: `GBMSimulator` is pure, synchronous, dependency-free computation
(easy to unit-test deterministically by seeding `numpy`/`random`), while
`SimulatorDataSource` is the thin `asyncio` + `PriceCache` glue that makes it conform to
`MarketDataSource`. `market_data_demo.py` (the Rich-based terminal dashboard) exercises
`SimulatorDataSource` directly and is a good manual sanity check when touching this code
— it also confirms visually that sector correlation and event shocks look right.

### The hot loop

```python
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

Wrapping the step in `try/except Exception` (logged, not re-raised) means a bug in one
tick's math can't silently kill the background task and freeze the entire watchlist's
prices — the loop keeps running on the next interval regardless.

## 6. Adding/Removing Tickers at Runtime

Both `GBMSimulator.add_ticker`/`remove_ticker` and `SimulatorDataSource`'s wrappers are
designed to be called while the loop is running (e.g., when the user adds a ticker to
their watchlist via the UI or the AI chat, per `PLAN.md` §6):

- `add_ticker`: appends to `_tickers`, seeds a starting price/params, **rebuilds the
  Cholesky matrix** (§3) to include the new ticker's correlations, and immediately seeds
  the shared `PriceCache` so the new ticker has *some* price before the next scheduled
  tick.
- `remove_ticker`: removes from `_tickers`/`_prices`/`_params`, rebuilds Cholesky for the
  now-smaller set, and removes the entry from `PriceCache` (`SimulatorDataSource.remove_ticker`
  calls `cache.remove(ticker)` explicitly — `GBMSimulator` itself has no cache reference).

Rebuilding the full correlation matrix on every add/remove is `O(n²)` but the watchlist
is expected to stay in the tens of tickers, so this is not a performance concern in
practice.
