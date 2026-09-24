# Market Data — Summary

**Status: complete.** The market data subsystem lives in `backend/app/market/` and is
covered by unit tests in `backend/tests/market/`. This document is the quick reference;
full design detail and research are in `planning/archive/`.

## What it does

One unified interface, two interchangeable sources, selected automatically by
environment variable — exactly per `PLAN.md` §6:

- `MASSIVE_API_KEY` set → real market data via the Massive (formerly Polygon.io) REST
  API, polled every 15s, one batched call for the whole watchlist.
- `MASSIVE_API_KEY` unset (default) → an in-process GBM simulator, stepping every 500ms,
  with sector-correlated moves and occasional random price events.

Both write into a single thread-safe `PriceCache`, which the `GET /api/stream/prices`
SSE endpoint reads from and streams to the frontend. Downstream code (SSE, and later
portfolio/trade logic) never talks to either source directly — only to the cache and the
`MarketDataSource` abstraction.

```python
from app.market import PriceCache, PriceUpdate, MarketDataSource, create_market_data_source, create_stream_router
```

See `backend/CLAUDE.md` for the day-to-day developer reference (imports, running tests,
the demo script).

## Where to look for more detail

| Document | Covers |
|---|---|
| [`archive/MASSIVE_API.md`](archive/MASSIVE_API.md) | Massive API research: auth, rate limits, the multi-ticker snapshot endpoint used for live prices, and the EOD/historical endpoints (grouped-daily, open-close, range aggregates) available for future use |
| [`archive/MARKET_INTERFACE.md`](archive/MARKET_INTERFACE.md) | The unified interface design — `PriceUpdate`, `PriceCache`, `MarketDataSource`, the factory, and the SSE consumer |
| [`archive/MARKET_SIMULATOR.md`](archive/MARKET_SIMULATOR.md) | The GBM simulator: price model, Cholesky-based sector correlation, seed data, code structure |

## Not yet used, documented for later

Massive's end-of-day endpoints (grouped-daily-for-all-tickers, per-ticker open/close,
and historical range aggregates — see `archive/MASSIVE_API.md` §4) aren't wired into the
current build. Nothing in the shipped feature set needs them: sparklines are accumulated
client-side from the live SSE stream and explicitly not backfilled (`PLAN.md` §10), and
the P&L chart draws from `portfolio_snapshots`, not market history. They're documented
now because they're the natural source if a future feature needs real historical
price data (e.g., a longer-range chart) — that would be a new read path, not a change to
`MarketDataSource` itself (see `archive/MARKET_INTERFACE.md` §6).
