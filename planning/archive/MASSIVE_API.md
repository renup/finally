# Massive API — Research Notes

Research reference for the Massive market data API (formerly Polygon.io — Polygon.io
rebranded to Massive.com on 2025-10-30). Existing Polygon.io API keys, accounts, and
integrations continued to work unchanged after the rebrand; `api.polygon.io` still
resolves, but `api.massive.com` is the current base URL and the one this project uses.

This document covers only what FinAlly needs: **pulling real-time-ish prices and
end-of-day prices for a *set* of tickers efficiently**, in a way that maps cleanly onto
the project's own `MarketDataSource` interface (see `MARKET_INTERFACE.md`). It is not a
full API reference — see https://massive.com/docs for that.

---

## 1. Client Library

The official Python SDK is the `massive` package on PyPI (the successor to
`polygon-api-client`). It's already a dependency of this project (`backend/pyproject.toml`,
`"massive>=1.0.0"`, currently resolving to `2.2.0`).

```bash
pip install -U massive
# or, in this project:
uv add massive
```

```python
from massive import RESTClient

client = RESTClient(api_key="YOUR_API_KEY")   # also reads POLYGON_API_KEY / MASSIVE_API_KEY env vars
```

The client is **synchronous**. In an asyncio codebase (this project's FastAPI backend),
calls must be wrapped in `asyncio.to_thread(...)` to avoid blocking the event loop —
see `backend/app/market/massive_client.py` for the pattern already in use.

### Raw REST alternative

Every SDK method is a thin wrapper over a plain HTTPS GET. If a future need falls
outside the SDK's coverage, the raw request shape is:

```bash
curl "https://api.massive.com/v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,MSFT&apiKey=YOUR_API_KEY"
```

Authentication accepts either form:
- Query parameter: `?apiKey=YOUR_API_KEY`
- Header: `Authorization: Bearer YOUR_API_KEY`

Base URL: `https://api.massive.com` (the SDK defaults to this; `api.polygon.io` remains
a working alias).

---

## 2. Rate Limits

Massive doesn't have named "free tier" request caps per se — it's per **asset class**:

- Any asset class you have **not** subscribed to (i.e., no paid plan for it) sits on the
  free **Basic** tier: **5 requests/minute**.
- Any asset class you **do** pay for has no documented request-rate cap.
- Limits are counted per asset class, not per account or per API key — creating more
  keys doesn't raise the ceiling.

For FinAlly (stocks only, likely on the free Basic tier or the $29/mo Starter plan),
this means: **poll on an interval no tighter than every 15 seconds** to stay safely
under 5 req/min with a single call per poll (5 calls/min → one call every 12s is the
hard floor; 15s matches this project's existing default and leaves headroom). This is
exactly why the project design batches all watched tickers into **one** snapshot call
per poll instead of one call per ticker — the call count is independent of watchlist
size.

| Plan | Stocks REST limit | Data recency |
|---|---|---|
| Basic (no stocks subscription) | 5 req/min | 15-minute delayed |
| Starter ($29/mo) | Unlimited | 15-minute delayed, 5yr history |
| Developer | Unlimited | 15-minute delayed |
| Advanced / Business | Unlimited | Real-time | 

Even on a real-time-capable plan, the project should keep polling (not push/WebSocket)
per the architecture decision in `PLAN.md` §6 — polling is simpler and the SSE layer
in front of the price cache is what gives the frontend its "live" feel regardless of
how fresh the upstream data actually is.

---

## 3. Real-Time(-ish) Prices for Multiple Tickers

### 3.1 Full Market Snapshot — what this project uses

```
GET /v2/snapshot/locale/us/markets/stocks/tickers
```

| Param | Required | Notes |
|---|---|---|
| `tickers` | No | Comma-separated, case-sensitive list (e.g. `AAPL,MSFT,TSLA`). Omit for the entire market (10,000+ tickers) — always pass it explicitly here. |
| `include_otc` | No | Default `false`. |

One call returns a snapshot **per requested ticker**, each containing the last trade,
last quote, the current minute's OHLCV bar, and the previous day's OHLCV bar:

```json
{
  "status": "OK",
  "count": 2,
  "tickers": [
    {
      "ticker": "AAPL",
      "todaysChange": 1.23,
      "todaysChangePerc": 0.65,
      "updated": 1707580800000000000,
      "day":     { "o": 189.10, "h": 191.05, "l": 188.90, "c": 190.33, "v": 41234567, "vw": 190.02 },
      "prevDay": { "o": 187.50, "h": 190.00, "l": 187.00, "c": 189.10, "v": 52345678, "vw": 188.75 },
      "min":     { "o": 190.30, "h": 190.35, "l": 190.28, "c": 190.33, "v": 12345, "t": 1707580740000 },
      "lastTrade": { "p": 190.33, "s": 100, "t": 1707580799123456789, "x": 4 },
      "lastQuote": { "p": 190.32, "P": 190.34, "s": 2, "S": 3, "t": 1707580799000000000 }
    }
  ]
}
```

Notes:
- `lastTrade.t` and `.updated` are **nanoseconds** since epoch, not milliseconds — a
  common off-by-1000x bug source. Divide by `1e9` for a Unix-seconds float.
- Fields are omitted rather than null when unavailable (e.g., a ticker with no trades
  yet today) — code consuming this must not assume every key is present.
- Snapshot data is cleared daily at ~3:30 AM ET and repopulates as exchanges report,
  starting as early as 4:00 AM ET — an empty/missing snapshot right after that window
  is expected, not an error.

**Python (SDK):**

```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient(api_key="YOUR_API_KEY")

snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "MSFT", "TSLA"],
)
for snap in snapshots:
    print(snap.ticker, snap.last_trade.price, snap.last_trade.timestamp)
```

This is precisely what `MassiveDataSource._fetch_snapshots()` in
`backend/app/market/massive_client.py` calls, on a 15-second `asyncio` loop, for the
full watchlist in one request.

### 3.2 Unified Snapshot — the newer, multi-asset-class alternative

```
GET /v3/snapshot?ticker.any_of=AAPL,MSFT,TSLA
```

Functionally similar for stocks, but designed to span stocks/options/forex/crypto in
one call and cap at 250 tickers per request (`ticker.any_of`, comma-separated, max 250).
Response shape differs slightly (`last_minute` instead of `min`, a `session` block with
`change`/`change_percent` pre-computed). Not currently used by this project — the v2
full-market-snapshot endpoint is simpler for a stocks-only watchlist and is what the
installed SDK version's `get_snapshot_all` targets — but worth knowing about if the
watchlist ever needs to include options or crypto.

---

## 4. End-of-Day Prices for Multiple Tickers

Three endpoints cover this, at different granularities. None of these are wired into
the current `MarketDataSource` implementation (the live SSE stream only needs
snapshots) — they're documented here because they're the natural source for any future
feature needing historical/EOD data (e.g., backfilling a longer-range chart, or an
end-of-day close-price reconciliation job).

### 4.1 Daily Market Summary ("grouped daily") — all tickers, one date, one call

The efficient way to get EOD prices for *many or all* tickers at once — exactly the
multi-ticker EOD case this research was asked to cover.

```
GET /v2/aggs/grouped/locale/us/market/stocks/{date}
```

| Param | Notes |
|---|---|
| `date` (path) | `YYYY-MM-DD` |
| `adjusted` | Default `true` (split-adjusted) |
| `include_otc` | Default `false` |

```json
{
  "status": "OK",
  "resultsCount": 4123,
  "results": [
    { "T": "AAPL", "o": 189.10, "h": 191.05, "l": 188.90, "c": 190.33, "v": 41234567, "vw": 190.02, "t": 1707595200000, "n": 312456 }
  ]
}
```

`T` = ticker, `t` = end-of-window timestamp in **milliseconds**. This single call
returns the whole market's closing prices for that date — filter client-side for the
tickers you care about rather than calling per-ticker.

```python
results = client.get_grouped_daily_aggs(date="2026-09-23", adjusted=True)
closes = {r.ticker: r.close for r in results}
```

### 4.2 Daily Ticker Summary ("open/close") — one ticker, one date

```
GET /v1/open-close/{ticker}/{date}
```

Returns `open`, `close`, `high`, `low`, `volume`, plus `preMarket` and `afterHours`
prices for that single ticker/date. Useful for a per-ticker daily reconciliation but
requires one call per ticker per date — for a watchlist-sized set, prefer 4.1 and filter.

```python
oc = client.get_daily_open_close_agg("AAPL", "2026-09-23", adjusted=True)
print(oc.close, oc.after_hours)
```

### 4.3 Custom Bars ("aggregates"/"range") — a time series for one ticker

```
GET /v2/aggs/ticker/{ticker}/range/{multiplier}/{timespan}/{from}/{to}
```

`timespan` ∈ `minute|hour|day|week|month|quarter|year`. For daily closes over a range
(e.g., a 30-day EOD history for a chart), set `multiplier=1, timespan=day`:

```python
aggs = list(client.list_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="day",
    from_="2026-08-01",
    to="2026-09-23",
    adjusted=True,
    limit=50000,
))
# each aggs[i] has .open .high .low .close .volume .vwap .timestamp (ms)
```

Again, this is per-ticker — for a whole-watchlist historical backfill, looping this
per ticker is the only option (there's no "grouped range" endpoint); the grouped-daily
endpoint (4.1) only gives one date per call.

---

## 5. Error Handling Notes

Carried into `MassiveDataSource` and worth keeping in mind for any future EOD code:

- **401** — bad/missing API key.
- **429** — rate limit exceeded (see §2); back off and retry on the next scheduled poll
  rather than retrying immediately.
- Malformed or partial snapshot entries (e.g., a ticker with no `last_trade` yet) should
  be skipped per-ticker, not treated as a fatal batch failure — one bad ticker shouldn't
  take down updates for the rest of the watchlist.
- Network/timeout errors should be logged and swallowed at the poll-cycle level; the
  next scheduled poll is the retry, matching the existing `_poll_once()` try/except in
  `massive_client.py`.

---

## 6. Summary — What Maps to What

| Need | Endpoint | SDK call | Used today? |
|---|---|---|---|
| Live-ish price per watchlist ticker, batched | `/v2/snapshot/locale/us/markets/stocks/tickers` | `client.get_snapshot_all(...)` | ✅ `MassiveDataSource` |
| EOD close for every ticker on one date | `/v2/aggs/grouped/locale/us/market/stocks/{date}` | `client.get_grouped_daily_aggs(...)` | Not yet — candidate for a future historical-backfill feature |
| EOD open/close for one ticker on one date | `/v1/open-close/{ticker}/{date}` | `client.get_daily_open_close_agg(...)` | Not yet |
| Daily/minute/etc. time series for one ticker | `/v2/aggs/ticker/{ticker}/range/...` | `client.list_aggs(...)` | Not yet |
