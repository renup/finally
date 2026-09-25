# Market Data — Detailed Design & Requirements Review (v2)

**Date:** 2026-09-25
**Scope:** `backend/app/market/`, `backend/tests/market/`, and how the market data
layer connects to the rest of the platform (watchlist, trades, chat, auth, deployment).
**Inputs reviewed:** `PLAN.md` (current version, with production hardening),
`MARKET_DATA_SUMMARY.md`, `archive/MARKET_DATA_DESIGN.md`, `archive/MARKET_INTERFACE.md`,
`archive/MARKET_SIMULATOR.md`, `archive/MASSIVE_API.md`, `archive/MARKET_DATA_REVIEW.md`,
the shipped code, and the installed `massive` SDK (2.x) source.

This document **supersedes `archive/MARKET_DATA_DESIGN.md`** as the implementation
reference. The v1 design was written before `PLAN.md` gained auth, a public-VPS
deployment, and several other requirements. v1 was also only checked with mocks, never
against the real Massive SDK objects. Sections 1–2 cover the review: what v1 meets, what
it misses, and what is broken. Sections 3–10 are the corrected design, with code. Section 11
is the implementation checklist.

---

## Table of Contents

1. [Review Verdict](#1-review-verdict)
2. [Requirements Traceability](#2-requirements-traceability)
3. [Findings (Severity-Ranked)](#3-findings-severity-ranked)
4. [Target Architecture](#4-target-architecture)
5. [Data Model & Cache](#5-data-model--cache)
6. [Interface, Ticker Normalization & Status](#6-interface-ticker-normalization--status)
7. [Simulator](#7-simulator)
8. [Massive Client](#8-massive-client)
9. [Factory & Configuration](#9-factory--configuration)
10. [SSE Stream, Lifecycle & Platform Integration](#10-sse-stream-lifecycle--platform-integration)
11. [Testing](#11-testing)
12. [Implementation Checklist](#12-implementation-checklist)
13. [Open Questions](#13-open-questions)

---

## 1. Review Verdict

**The simulator path is solid and meets its functional requirements. The Massive path
does not work at all against the real API. Several platform-level requirements added
to `PLAN.md` later (auth on `/api/*`, daily change %, trading a ticker that isn't on
the watchlist, keeping held positions priced, health monitoring) are either not
designed yet or only partly designed.**

All 73 unit tests pass (`uv run --extra dev pytest`). They pass only because every
Massive test builds snapshots with `MagicMock`, and `MagicMock` accepts any attribute
name. When the real SDK model is used, the Massive source silently drops every ticker:

```text
$ uv run python scratchpad/ts_check.py      # real massive.rest.models.TickerSnapshot
has .timestamp: False sip_timestamp: 1707580799123456789
Skipping snapshot for AAPL: 'LastTrade' object has no attribute 'timestamp'
cached AAPL: None
```

Summary of findings (details in §3):

| Severity | Count | Headline |
|---|---|---|
| Critical | 2 | Massive parser reads a field that doesn't exist, so no prices ever reach the cache; SSE endpoint is unauthenticated on a public app |
| High | 5 | No daily change %; held positions stop being priced when their ticker leaves the watchlist; Massive has no price yet when chat adds a ticker and trades it; ghost ticker race in Massive remove; SDK retries on 429 spend the rate limit |
| Medium | 7 | No ticker validation or consistent normalization; removals don't bump the cache version; no SSE heartbeat; stream router is module-global and registered inside lifespan; no data-feed health signal; poll interval not configurable; startup can block on a hung Massive call |
| Low | 5 | Stale NVDA seed price; no simulator seed for reproducible tests; `CancelledError` swallowed; SSE JSON re-serialized per client; 2-dp rounding drops sub-penny prices |

---

## 2. Requirements Traceability

Status key: ✅ met · ⚠️ partly met / fragile · ❌ not met · 🔧 met by this v2 design.

### 2.1 `PLAN.md` §6 — Market Data

| # | Requirement | v1 status | v2 resolution |
|---|---|---|---|
| R1 | Two implementations, one abstract interface, downstream code doesn't care which is active | ✅ `MarketDataSource` ABC + factory | Keep; add `status()` (§6) |
| R2 | `MASSIVE_API_KEY` set and non-empty → Massive; else simulator | ✅ `factory.py` strips whitespace | Keep |
| R3 | Simulator uses GBM with per-ticker drift/volatility | ✅ correct log-normal step | Keep |
| R4 | Simulator ticks about every 500ms | ✅ `update_interval=0.5` | Keep |
| R5 | Correlated sector moves via a shared factor | ✅ Cholesky correlation (0.6 tech / 0.5 finance / 0.3 other) | Keep; PSD guard for larger groups (§7) |
| R6 | Occasional 2–5% random events | ✅ 0.1% per ticker per tick | Keep |
| R7 | Realistic seed prices | ⚠️ NVDA at $800 is pre-2024-split (real price is about $100–200) | 🔧 refresh seeds (§7) |
| R8 | In-process, 24/7, no market-hours gating | ✅ | Keep |
| R9 | Massive: REST polling, not WebSocket | ✅ | Keep |
| R10 | One batched call covers every watched ticker | ✅ `get_snapshot_all(tickers=[...])` | Keep; also count SDK retries against the budget (F7) |
| R11 | Poll interval configurable (15s free, 2–15s paid) | ❌ hard-coded 15s default, and the factory never passes a value | 🔧 `MASSIVE_POLL_INTERVAL` env var (§9) |
| R12 | Parse Massive responses into the simulator's format | ❌ **broken**: `last_trade.timestamp` doesn't exist, and the SDK timestamp is in nanoseconds, not milliseconds (F1) | 🔧 `parse_snapshot()` (§8.2) |
| R13 | Shared cache holds latest price, previous price, timestamp | ✅ | Keep; add `reference_price` (R19) |
| R14 | SSE reads from the cache, never from the source | ✅ | Keep |
| R15 | `GET /api/stream/prices`, native `EventSource`, about 500ms cadence | ✅ | Keep; add heartbeat (F12) |
| R16 | Each event carries ticker, price, previous price, timestamp, direction | ✅ via `PriceUpdate.to_dict()` | Keep; add day-change fields |
| R17 | Client reconnects automatically | ✅ `retry: 1000` | Keep |
| R18 | "All tickers known to the system", which today is the user's watchlist | ⚠️ only true if every add/remove path calls the source | 🔧 tracked set = watchlist ∪ open positions (§10.4) |

### 2.2 Requirements from other `PLAN.md` sections that depend on market data

| # | Source | Requirement | v1 status | v2 resolution |
|---|---|---|---|---|
| R19 | §10 Watchlist panel | Show **daily change %** | ❌ `change_percent` is tick-to-tick, so it hovers near 0.00% (simulator) or measures poll-to-poll (Massive) | 🔧 `reference_price` + `day_change_percent` (§5) |
| R20 | §10 Sparklines, main chart | Built client-side from the SSE stream, not backfilled | ✅ full frame every tick | Keep |
| R21 | §8 Auth | Every `/api/*` route except login and health returns 401 without a session | ❌ stream router has no auth dependency (F2) | 🔧 `Depends(require_user)` (§10.1) |
| R22 | §9 step 6 | If chat trades a ticker that isn't watched, add it to the watchlist first "so it has a live price", then trade | ⚠️ works on the simulator (cache is seeded in `add_ticker`); **fails on Massive**, where the price only arrives on the next poll, up to 15s later (F5) | 🔧 early poll + `wait_for_price()` (§5, §8) |
| R23 | §7/§8 Portfolio | Positions are valued at the current price, with 30s snapshots | ⚠️ a ticker removed from the watchlist while still held has no price | 🔧 tracked-set reconciliation (§10.4) |
| R24 | §8 Trade | Instant fill at the current price | ✅ `cache.get_price()` | Keep; clear error if the price is unavailable |
| R25 | §8 Watchlist | Watchlist responses include the latest prices | ✅ cache lookup | Keep |
| R26 | §11 Monitoring | `/api/health` for uptime checks; structured logs | ⚠️ health can't tell whether the price feed is stale or failing | 🔧 `source.status()` in health (§10.5) |
| R27 | §11 Caddy | SSE survives the reverse proxy | ⚠️ no idle heartbeat; Massive can go 15s+ without a frame | 🔧 heartbeat comment every 15s (§10.1) |
| R28 | §3 Future multi-user | "Supports future multi-user scenarios without changes to the data layer" | ✅ cache is global and keyed by ticker | Keep; tracked set = union across users (§10.4) |
| R29 | §12 Tests | Massive response parsing works; both implementations conform to the interface | ❌ parsing tests use `MagicMock`, which hides F1 | 🔧 tests built from real `TickerSnapshot.from_dict` (§11) |
| R30 | §12 E2E | Deterministic, reproducible runs | ⚠️ simulator can't be seeded | 🔧 `MARKET_SIM_SEED` (§9) |

---

## 3. Findings (Severity-Ranked)

### Critical

**F1: The Massive parser never produces a price.** `massive_client.py:108` reads
`snap.last_trade.timestamp`. In SDK 2.x, `massive.rest.models.trades.LastTrade` has
`sip_timestamp`, `participant_timestamp`, and `trf_timestamp`, but no `timestamp`. Every
snapshot raises `AttributeError` inside the per-ticker `try`, gets logged as "Skipping",
and is dropped. The cache stays empty, and SSE sends only the `retry:` line. Even if the
attribute existed, `/1000.0` assumes milliseconds, while `MASSIVE_API.md` §3.1 (correctly)
says `lastTrade.t` is in **nanoseconds**. There is also no fallback when `lastTrade` is
missing (thin tickers, pre-market after the ~3:30 AM ET reset), even though the snapshot
still has `min`, `day`, or `prevDay` closes. *Fix: §8.2.*

**F2: `/api/stream/prices` is unauthenticated.** `PLAN.md` §8 requires a session cookie
on every `/api/*` route except login and health. The stream router has no dependency, so
anyone who finds the URL can hold connections open indefinitely. The cost is one
coroutine each plus JSON serialization every 500ms, which is a cheap DoS vector against a
single-instance app. *Fix: §10.1.*

### High

**F3: No daily change %.** `PriceUpdate.change_percent` compares against the previous
*tick*. The watchlist column required by §10 needs a daily reference price: the previous
close on Massive (`prev_day.close`, or the SDK's `todays_change_percent`), and a session
open on the simulator (the price when the ticker was first seeded). *Fix: §5.1.*

**F4: A ticker removed from the watchlist while still held stops being priced.** v1 §11
proposes a check at the route level, but it's a comment in a snippet: nothing enforces it,
the chat's watchlist changes need the same logic, and a full sell of a ticker that isn't
watched should *stop* tracking it. Startup also loads only watchlist tickers, so a held
position that isn't watched has no price after a restart. Portfolio value, heatmap, P&L
snapshots, and trades then all fail for that ticker. *Fix: §10.4.*

**F5: Chat trade → auto-add → trade fails on Massive.** `MassiveDataSource.add_ticker`
only appends to the list. The price shows up on the next poll, up to 15s later. The chat
flow in §9 step 6 adds the ticker and trades immediately, so the trade hits "no price
available" every time. *Fix: early poll via `_wake` (§8.3) + `PriceCache.wait_for_price` (§5.2).*

**F6: Ghost ticker race in `MassiveDataSource.remove_ticker`.** A poll runs in a worker
thread for up to 10s (the SDK read timeout). If `remove_ticker("X")` runs during that
window, it calls `cache.remove("X")`. The in-flight poll then returns and calls
`cache.update("X", ...)`, putting X back. Nothing ever removes it again, so it stays in
every SSE frame for the rest of the process's life. `add_ticker` also mutates
`self._tickers` in place while the worker thread may be reading it for the request.
*Fix: snapshot the ticker list before the call, and drop results for tickers that are no
longer tracked (§8.3).*

**F7: SDK retries spend the rate limit.** `RESTClient` defaults to `retries=3`, with
`status_forcelist` including **429** and `backoff_factor=0.1`. One 429 on the free tier
(5 requests/min) becomes 4 requests within about 0.7s, which keeps the account
rate-limited. The poll schedule is already the retry. *Fix: `RESTClient(retries=0)`
(§8.3).*

### Medium

**F8: No ticker validation, inconsistent normalization.** `MassiveDataSource` upper-cases
and strips tickers, but `SimulatorDataSource` doesn't, so `"aapl"` and `"AAPL"` become
two simulated tickers, and the cache key differs from the DB key. The simulator also
accepts any string (`"'; DROP"`, `"HELLO WORLD"`) and invents a price for it. That's fine
for a demo, but an LLM that hallucinates a symbol will "successfully" add and trade it.
*Fix: `normalize_ticker()` at the boundary (§6.2), plus a Massive-side check that the
symbol actually returned a snapshot (§8.4).*

**F9: `PriceCache.remove()` doesn't bump `version`.** The SSE loop only sends a frame
when the version changes, so a removal isn't visible to the client until the next
`update()`. That's 500ms on the simulator, but up to 15s on Massive, and forever if the
watchlist becomes empty. *Fix: §5.2.*

**F10: The stream router is module-global and mounted inside `lifespan`.** Calling
`create_stream_router()` twice (tests, reloads) registers `/prices` twice on the same
`APIRouter` (previous review item 3.6, never fixed). v1 §10 also calls
`app.include_router()` *inside* the lifespan, which means the route isn't in the
OpenAPI schema and doesn't exist for a `TestClient` that skips lifespan. *Fix: build the
router at import time and read the cache from `request.app.state` (§10.1).*

**F11: No feed-health signal.** With a bad key (401), `/api/health` still reports
healthy and the frontend shows a green "connected" dot over prices that never change
(v1 §13.3 accepts this). For a production app monitored per `PLAN.md` §11, health should
report the source, the time since the last successful update, and the last error.
*Fix: `SourceStatus` (§6.3) and health (§10.5).*

**F12: No SSE heartbeat.** When nothing changes (Massive between polls, empty
watchlist), the stream sends no bytes. Idle proxies, NATs, and some browsers eventually
drop the connection. `request.is_disconnected()` also only notices a dead client when a
write fails. *Fix: send a `: ping` comment every 15s (§10.1).*

**F13: Poll interval can't be configured (R11).** *Fix: §9.*

**F14: Startup can block on Massive.** `start()` awaits the first poll inline. With a
hung network, each attempt can wait up to 10s connect + 10s read, and with the default
retries that multiplies. That delays FastAPI startup and can fail container health
checks. *Fix: run the first poll inside the background task and bound it with a
timeout (§8.3).*

### Low

**F15:** NVDA seed price of $800 is pre-split. Refresh seeds to plausible 2026 levels
(§7).
**F16:** No way to seed the simulator's RNG, so E2E visual checks aren't reproducible
(§7, §9).
**F17:** `_generate_events` catches `CancelledError` and doesn't re-raise it. Log it,
then re-raise, so structured concurrency behaves (§10.1).
**F18:** Every SSE client re-serializes the same JSON every 500ms. Cache the payload per
cache version (§10.1). This doesn't matter for one user, but it's free.
**F19:** `PriceCache.update` rounds to 2 decimal places. Correct for the default
watchlist, but lossy for sub-$1 tickers from Massive. Round only for display (frontend),
or use 4 dp below $1. Left as-is, but documented.

---

## 4. Target Architecture

```
                      ┌────────────────────────────────────────────────────────┐
  DB (SQLite)         │                 FastAPI process                        │
  watchlist ∪ open ───┼─► sync_tracked_tickers() ──┐                           │
  positions           │   (startup, after watchlist│change, after every trade) │
                      │                            ▼                           │
                      │        MarketDataSource (ABC)  ── status() ──► /api/health
                      │        ├─ SimulatorDataSource  (GBM, 500ms)            │
                      │        └─ MassiveDataSource    (REST snapshot, N sec)  │
                      │                 │ cache.update(ticker, price, ts, ref) │
                      │                 ▼                                      │
                      │        PriceCache (thread-safe, versioned)             │
                      │          │                 │              │            │
                      │          ▼                 ▼              ▼            │
                      │  /api/stream/prices   trade execution  portfolio value │
                      │  (auth, SSE, 500ms,   (+ wait_for_     snapshots (30s) │
                      │   heartbeat)           price on add)   /api/watchlist  │
                      └────────────────────────────────────────────────────────┘
```

Unchanged from v1: the push model (sources write, consumers read the cache), exactly one
active source chosen at startup, and the `app.market` public import surface. Additions:
`normalize_ticker`, `SourceStatus`, the `reference_price` / day-change fields,
`wait_for_price`, and a tracked-ticker reconciler owned by the platform layer (not by
the market package).

### Module layout (v2)

```
backend/app/market/
├── __init__.py         # + normalize_ticker, SourceStatus, InvalidTickerError
├── models.py           # PriceUpdate (+ reference_price, day_change, day_change_percent)
├── cache.py            # PriceCache (+ reference prices, version bump on remove, wait_for_price)
├── interface.py        # MarketDataSource (+ status()), SourceStatus
├── tickers.py          # NEW: normalize_ticker(), InvalidTickerError
├── seed_prices.py      # refreshed seeds
├── simulator.py        # GBMSimulator (+ rng seed), SimulatorDataSource
├── massive_client.py   # parse_snapshot(), MassiveDataSource (fixed)
├── factory.py          # reads MASSIVE_POLL_INTERVAL, MARKET_SIM_SEED
└── stream.py           # module-level router, auth dependency, heartbeat
backend/app/services/
└── tracked_tickers.py  # NEW: sync_tracked_tickers(db, source) (§10.4)
```

---

## 5. Data Model & Cache

### 5.1 `models.py`: add a daily reference price

`reference_price` is the price that "daily change" is measured from. On Massive it's
`prev_day.close`. On the simulator it's the price when the ticker was first seeded,
because the simulator has no real days. It's optional: if unknown, the day-change fields
are `None` and the frontend shows `—`.

```python
from __future__ import annotations

import time
from dataclasses import dataclass, field
from typing import Literal

Direction = Literal["up", "down", "flat"]


@dataclass(frozen=True, slots=True)
class PriceUpdate:
    """Immutable snapshot of a single ticker's price at a point in time."""

    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds
    reference_price: float | None = None  # previous close (Massive) / session open (simulator)

    @property
    def change(self) -> float:
        """Tick-to-tick change. Drives the green/red flash."""
        return round(self.price - self.previous_price, 4)

    @property
    def change_percent(self) -> float:
        if self.previous_price == 0:
            return 0.0
        return round((self.price - self.previous_price) / self.previous_price * 100, 4)

    @property
    def direction(self) -> Direction:
        if self.price > self.previous_price:
            return "up"
        if self.price < self.previous_price:
            return "down"
        return "flat"

    @property
    def day_change(self) -> float | None:
        """Change vs. the daily reference. Drives the watchlist 'daily change %' column."""
        if not self.reference_price:
            return None
        return round(self.price - self.reference_price, 4)

    @property
    def day_change_percent(self) -> float | None:
        if not self.reference_price:
            return None
        return round((self.price - self.reference_price) / self.reference_price * 100, 4)

    def to_dict(self) -> dict:
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "timestamp": self.timestamp,
            "change": self.change,
            "change_percent": self.change_percent,
            "direction": self.direction,
            "reference_price": self.reference_price,
            "day_change": self.day_change,
            "day_change_percent": self.day_change_percent,
        }
```

Adding fields is backward compatible for the SSE wire format. The existing keys are
unchanged, and the new ones are additive.

### 5.2 `cache.py`: reference prices, version bump on remove, `wait_for_price`

```python
from __future__ import annotations

import asyncio
import time
from threading import Lock

from .models import PriceUpdate


class PriceCache:
    """Thread-safe in-memory cache of the latest price per ticker.

    Writers: exactly one MarketDataSource (event loop or a to_thread worker).
    Readers: SSE, trade execution, portfolio valuation, watchlist/health routes.
    """

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()
        self._version = 0

    def update(
        self,
        ticker: str,
        price: float,
        timestamp: float | None = None,
        reference_price: float | None = None,
    ) -> PriceUpdate:
        """Record a new price. `reference_price=None` keeps the existing reference;
        for a brand-new ticker it defaults to the first observed price (session open)."""
        with self._lock:
            prev = self._prices.get(ticker)
            previous_price = prev.price if prev else price
            if reference_price is None:
                reference_price = prev.reference_price if prev else price
            update = PriceUpdate(
                ticker=ticker,
                price=round(price, 2),
                previous_price=round(previous_price, 2),
                timestamp=timestamp if timestamp is not None else time.time(),
                reference_price=round(reference_price, 2) if reference_price else None,
            )
            self._prices[ticker] = update
            self._version += 1
            return update

    def remove(self, ticker: str) -> None:
        with self._lock:
            if self._prices.pop(ticker, None) is not None:
                self._version += 1  # F9: removals must trigger an SSE frame

    def get(self, ticker: str) -> PriceUpdate | None:
        with self._lock:
            return self._prices.get(ticker)

    def get_price(self, ticker: str) -> float | None:
        update = self.get(ticker)
        return update.price if update else None

    def get_all(self) -> dict[str, PriceUpdate]:
        with self._lock:
            return dict(self._prices)

    @property
    def version(self) -> int:
        with self._lock:
            return self._version

    async def wait_for_price(self, ticker: str, timeout: float) -> float | None:
        """Wait up to `timeout` seconds for a first price to appear (F5).

        Polls every 100ms rather than using an asyncio.Event, because writers can be
        in a worker thread, and signalling an Event from another thread requires
        call_soon_threadsafe plumbing in every writer. 100ms is imperceptible here.
        """
        deadline = time.monotonic() + timeout
        while True:
            price = self.get_price(ticker)
            if price is not None or time.monotonic() >= deadline:
                return price
            await asyncio.sleep(0.1)

    def __len__(self) -> int:
        with self._lock:
            return len(self._prices)

    def __contains__(self, ticker: str) -> bool:
        with self._lock:
            return ticker in self._prices
```

Also fixed: `timestamp or time.time()` became an explicit `is not None` check, and
`version` now reads under the lock (previous review item 3.4).

---

## 6. Interface, Ticker Normalization & Status

### 6.1 `interface.py`

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import asdict, dataclass
from typing import Literal


@dataclass(frozen=True, slots=True)
class SourceStatus:
    source: Literal["simulator", "massive"]
    running: bool
    tickers: int
    last_success_at: float | None  # Unix seconds of the last successful cache write batch
    last_error: str | None          # most recent failure message, cleared on success
    consecutive_failures: int

    def to_dict(self) -> dict:
        return asdict(self)


class MarketDataSource(ABC):
    """Contract for market data providers. Sources push into a shared PriceCache;
    consumers read the cache and never call the source for prices.

    All ticker arguments are expected to be normalized (see tickers.normalize_ticker);
    implementations normalize again defensively.
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Seed the cache if possible and start the background task. Must not block
        for longer than a short, bounded time (F14). Call once."""

    @abstractmethod
    async def stop(self) -> None:
        """Cancel the background task. Idempotent."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Begin tracking. No-op if present. Should make a price available as soon
        as the source allows (simulator: immediately; Massive: next permitted poll)."""

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Stop tracking and remove from the cache. No-op if absent. Must guarantee
        that no in-flight update re-inserts the ticker afterwards (F6)."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Currently tracked tickers."""

    @abstractmethod
    def status(self) -> SourceStatus:
        """Health snapshot for /api/health and logs (F11)."""
```

### 6.2 `tickers.py`: one normalization rule for the whole app (F8)

Every entry point (watchlist route, trade route, chat action executor) calls this
*before* touching the DB or the source, so the DB key, cache key, and source key always
match.

```python
from __future__ import annotations

import re

# US equities: 1–5 letters, optional class suffix (BRK.B, BF-B). Deliberately strict:
# the LLM can propose any string, and this is the first line of defense.
_TICKER_RE = re.compile(r"^[A-Z]{1,5}([.\-][A-Z]{1,2})?$")


class InvalidTickerError(ValueError):
    pass


def normalize_ticker(raw: str) -> str:
    ticker = (raw or "").strip().upper()
    if not _TICKER_RE.fullmatch(ticker):
        raise InvalidTickerError(f"Invalid ticker symbol: {raw!r}")
    return ticker
```

Routes map `InvalidTickerError` to HTTP 400. The chat executor puts the error into the
`actions` result so the LLM can tell the user (`PLAN.md` §9, "Auto-Execution").

Format validation doesn't prove the symbol exists. The simulator still accepts any
well-formed symbol by design, since the simulator is a toy. On Massive, a symbol that
returns no snapshot is surfaced through `wait_for_price` timing out (§10.3).

---

## 7. Simulator

The v1 GBM engine is correct and stays as it is (math, Cholesky correlation, events,
hot loop). The changes are small.

### 7.1 Seedable RNG (F16) and normalized tickers (F8)

Replace the module-global `np.random` / `random` calls with instance generators, so a
seed makes a run fully reproducible:

```python
class GBMSimulator:
    def __init__(
        self,
        tickers: list[str],
        dt: float = DEFAULT_DT,
        event_probability: float = 0.001,
        seed: int | None = None,
    ) -> None:
        self._rng = np.random.default_rng(seed)
        ...

    def step(self) -> dict[str, float]:
        n = len(self._tickers)
        if n == 0:
            return {}
        z = self._rng.standard_normal(n)
        if self._cholesky is not None:
            z = self._cholesky @ z
        # Vectorized GBM (same math as v1, one exp() call for all tickers)
        mu, sigma = self._mu_vec, self._sigma_vec          # rebuilt with the Cholesky
        self._price_vec *= np.exp((mu - 0.5 * sigma**2) * self._dt + sigma * np.sqrt(self._dt) * z)
        shocks = self._rng.random(n) < self._event_prob
        if shocks.any():
            mags = self._rng.uniform(0.02, 0.05, n) * self._rng.choice([-1.0, 1.0], n)
            self._price_vec[shocks] *= 1 + mags[shocks]
        return {t: round(float(p), 2) for t, p in zip(self._tickers, self._price_vec)}

    def _add_ticker_internal(self, ticker: str) -> None:
        ...
        seed = SEED_PRICES.get(ticker)
        self._prices[ticker] = seed if seed is not None else float(self._rng.uniform(50.0, 300.0))
```

Vectorization is optional. v1's per-ticker loop is fine at watchlist sizes, but a
vectorized step keeps `step()` O(1) Python work as the tracked set grows with
multi-user. If the loop stays, just swap in `self._rng`.

`SimulatorDataSource.add_ticker` / `remove_ticker` call `normalize_ticker()` first, the
same as the Massive source.

### 7.2 Correlation matrix safety

The 0.6/0.5/0.3 block structure is always positive-definite for the current groups.
But if someone later adds a group with ρ below the cross-group value, or edits constants,
`np.linalg.cholesky` raises `LinAlgError`, which would crash `add_ticker`. Guard it:

```python
try:
    self._cholesky = np.linalg.cholesky(corr)
except np.linalg.LinAlgError:
    # Nudge to the nearest PSD matrix by clipping eigenvalues; never fail an add_ticker.
    w, v = np.linalg.eigh(corr)
    corr = v @ np.diag(np.clip(w, 1e-6, None)) @ v.T
    d = np.sqrt(np.diag(corr))
    self._cholesky = np.linalg.cholesky(corr / np.outer(d, d))
```

### 7.3 Seed data refresh (F15) and constants cleanup

Update `SEED_PRICES` to plausible current levels. The exact numbers don't matter, but
NVDA at $800 is a visible anachronism after its 10:1 split in 2024. Remove the unused
`DEFAULT_CORR` constant from the v1 design doc; the code already uses `CROSS_GROUP_CORR`
for unknown tickers (previous review item 4.3).

### 7.4 Status

```python
def status(self) -> SourceStatus:
    return SourceStatus(
        source="simulator",
        running=self._task is not None and not self._task.done(),
        tickers=len(self.get_tickers()),
        last_success_at=self._last_step_at,   # set in _run_loop after each successful step
        last_error=self._last_error,
        consecutive_failures=self._failures,
    )
```

---

## 8. Massive Client

### 8.1 What the real response looks like (SDK 2.x)

`client.get_snapshot_all(SnapshotMarketType.STOCKS, tickers=[...])` returns
`list[TickerSnapshot]`. The relevant fields, checked in
`.venv/.../massive/rest/models/snapshot.py` and `trades.py`:

| Field | Type | Notes |
|---|---|---|
| `ticker` | `str \| None` | |
| `last_trade` | `LastTrade \| None` | `None` when there's no `lastTrade` key |
| `last_trade.price` | `float \| None` | |
| `last_trade.sip_timestamp` | `int \| None` | **nanoseconds** (JSON `t`); there is no `.timestamp` |
| `last_trade.participant_timestamp` | `int \| None` | nanoseconds (JSON `y`) |
| `min.close`, `day.close` | `float \| None` | current-minute and current-day bars |
| `prev_day.close` | `float \| None` | previous session close, used as the **daily reference** |
| `todays_change_percent` | `float \| None` | computed by Massive against `prev_day.close` |
| `updated` | `int \| None` | nanoseconds |

Tickers the API doesn't recognize are simply **missing** from the list, not errors.

### 8.2 `parse_snapshot()`: a pure function, unit-tested against real SDK objects (F1)

```python
def _positive(x: object) -> float | None:
    return float(x) if isinstance(x, (int, float)) and x > 0 else None


def parse_snapshot(snap) -> tuple[str, float, float, float | None] | None:
    """Map a massive TickerSnapshot → (ticker, price, unix_seconds, reference_price).

    Price precedence: last trade → current-minute close → day close → previous close.
    Returns None if the snapshot carries no usable price (skip, don't fail the batch).
    """
    ticker = snap.ticker
    if not ticker:
        return None
    lt = snap.last_trade
    price = (
        _positive(lt.price if lt else None)
        or _positive(snap.min.close if snap.min else None)
        or _positive(snap.day.close if snap.day else None)
        or _positive(snap.prev_day.close if snap.prev_day else None)
    )
    if price is None:
        return None
    ts_ns = (lt.sip_timestamp or lt.participant_timestamp) if lt else None
    ts_ns = ts_ns or snap.updated
    timestamp = ts_ns / 1e9 if ts_ns else time.time()  # Massive timestamps are nanoseconds
    reference = _positive(snap.prev_day.close if snap.prev_day else None)
    return ticker, price, timestamp, reference
```

I checked this against real `TickerSnapshot.from_dict(...)` objects (scratch run):

```text
{"lastTrade":{"p":190.33,"t":1707580799123456789},"prevDay":{"c":189.10}} → ('AAPL', 190.33, 1707580799.1234567, 189.1)
{"day":{"c":12.5},"updated":1707580799000000000}                          → ('NEW', 12.5, 1707580799.0, None)
{}                                                                        → None
{"prevDay":{"c":50.0}}   (pre-market, after the daily reset)             → ('PRE', 50.0, <now>, 50.0)
```

### 8.3 `MassiveDataSource` (fixes F5, F6, F7, F13, F14)

```python
from __future__ import annotations

import asyncio
import logging
import time

from massive import RESTClient
from massive.rest.models import SnapshotMarketType

from .cache import PriceCache
from .interface import MarketDataSource, SourceStatus
from .tickers import normalize_ticker

logger = logging.getLogger(__name__)


class MassiveDataSource(MarketDataSource):
    """Polls the Massive full-market snapshot endpoint for all tracked tickers in ONE
    call per cycle, and writes the results to the PriceCache.

    Rate budget: at most one request every `min_gap` seconds (free tier: 5 req/min, so
    12s). The regular schedule is `poll_interval` (default 15s). add_ticker() requests an
    early poll, which still respects min_gap, so a newly added ticker gets a price in
    ≤ min_gap seconds instead of ≤ poll_interval.
    """

    def __init__(
        self,
        api_key: str,
        price_cache: PriceCache,
        poll_interval: float = 15.0,
        min_gap: float = 12.0,
        request_timeout: float = 8.0,
    ) -> None:
        self._api_key = api_key
        self._cache = price_cache
        self._interval = max(poll_interval, min_gap)
        self._min_gap = min_gap
        self._timeout = request_timeout
        self._tickers: list[str] = []
        self._client: RESTClient | None = None
        self._task: asyncio.Task | None = None
        self._wake = asyncio.Event()
        self._last_request_at = 0.0
        # status
        self._last_success_at: float | None = None
        self._last_error: str | None = None
        self._failures = 0

    # --- lifecycle ---------------------------------------------------------------

    async def start(self, tickers: list[str]) -> None:
        # F7: the poll schedule IS the retry. SDK-level retries on 429 would burn the
        # 5 req/min budget in under a second.
        self._client = RESTClient(
            api_key=self._api_key,
            retries=0,
            connect_timeout=self._timeout,
            read_timeout=self._timeout,
        )
        self._tickers = [normalize_ticker(t) for t in tickers]
        # F14: don't await the network here. The loop polls immediately on its first pass.
        self._wake.set()
        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")
        logger.info("Massive poller started: %d tickers, %.1fs interval",
                    len(self._tickers), self._interval)

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        self._client = None

    # --- ticker set --------------------------------------------------------------

    async def add_ticker(self, ticker: str) -> None:
        ticker = normalize_ticker(ticker)
        if ticker not in self._tickers:
            self._tickers = [*self._tickers, ticker]  # rebind, never mutate in place (F6)
            self._wake.set()                          # F5: request an early poll

    async def remove_ticker(self, ticker: str) -> None:
        ticker = normalize_ticker(ticker)
        self._tickers = [t for t in self._tickers if t != ticker]
        self._cache.remove(ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    # --- polling -----------------------------------------------------------------

    async def _poll_loop(self) -> None:
        while True:
            try:
                await asyncio.wait_for(self._wake.wait(), timeout=self._interval)
            except TimeoutError:
                pass
            self._wake.clear()
            gap = self._last_request_at + self._min_gap - time.monotonic()
            if gap > 0:
                await asyncio.sleep(gap)
            await self._poll_once()

    async def _poll_once(self) -> None:
        requested = self._tickers  # immutable snapshot (the list is only ever rebound)
        if not requested or not self._client:
            return
        self._last_request_at = time.monotonic()
        try:
            snapshots = await asyncio.wait_for(
                asyncio.to_thread(self._fetch_snapshots, requested),
                timeout=self._timeout * 2 + 1,
            )
        except Exception as e:  # 401, 403 (plan), 429, timeout, network
            self._failures += 1
            self._last_error = f"{type(e).__name__}: {e}"
            log = logger.error if self._failures in (1, 10) or self._failures % 40 == 0 else logger.debug
            log("Massive poll failed (%d in a row): %s", self._failures, self._last_error)
            return

        tracked = set(self._tickers)  # re-read AFTER the await (F6)
        written = 0
        for snap in snapshots:
            try:
                parsed = parse_snapshot(snap)
            except Exception:  # defensive: never let one ticker kill the batch
                logger.warning("Unparseable snapshot for %s", getattr(snap, "ticker", "?"))
                continue
            if parsed is None:
                continue
            ticker, price, ts, ref = parsed
            if ticker not in tracked:
                continue  # removed while the request was in flight: don't resurrect it
            self._cache.update(ticker, price, timestamp=ts, reference_price=ref)
            written += 1

        self._failures = 0
        self._last_error = None
        self._last_success_at = time.time()
        missing = tracked - {getattr(s, "ticker", None) for s in snapshots}
        if missing:
            logger.info("Massive returned no snapshot for: %s", ", ".join(sorted(missing)))
        logger.debug("Massive poll: %d/%d tickers updated", written, len(requested))

    def _fetch_snapshots(self, tickers: list[str]) -> list:
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=tickers,
        )

    def status(self) -> SourceStatus:
        return SourceStatus(
            source="massive",
            running=self._task is not None and not self._task.done(),
            tickers=len(self._tickers),
            last_success_at=self._last_success_at,
            last_error=self._last_error,
            consecutive_failures=self._failures,
        )
```

Notes:
- **Log throttling.** A bad key otherwise logs an error every 15s forever. The pattern
  above logs the 1st, 10th, and every 40th failure (about every 10 minutes) at ERROR.
- **`asyncio.wait_for` around `to_thread`** stops *awaiting* after the timeout, but it
  can't kill the worker thread. The SDK's own connect/read timeouts are what actually
  bound the thread. Both are set.
- The **v1 lazy-import vs. top-level-import** debate is settled: commit `6a2b36e` made
  `massive` a hard dependency with top-level imports. Keep that; it's in
  `pyproject.toml` anyway.

### 8.4 Unknown symbols on Massive

A symbol that passes `normalize_ticker` but doesn't exist (e.g. `ZZZZ`) never appears in
the snapshot list. The watchlist route handles this with `wait_for_price` (§10.3). If no
price arrives within the timeout, it keeps the ticker, returns it with `price: null`,
and logs it. It doesn't reject, because a real ticker legitimately has no snapshot for a
short window after the daily reset. The chat flow instead *reports* "no price available
for ZZZZ yet" and skips the trade.

---

## 9. Factory & Configuration

```python
def _float_env(name: str, default: float, minimum: float) -> float:
    raw = os.environ.get(name, "").strip()
    if not raw:
        return default
    try:
        return max(float(raw), minimum)
    except ValueError:
        logger.warning("Ignoring invalid %s=%r; using %s", name, raw, default)
        return default


def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        interval = _float_env("MASSIVE_POLL_INTERVAL", 15.0, minimum=1.0)
        # Free tier needs 12s between calls; paid tiers can lower MASSIVE_MIN_GAP.
        min_gap = _float_env("MASSIVE_MIN_GAP", 12.0, minimum=0.5)
        logger.info("Market data source: Massive (poll %.1fs, min gap %.1fs)", interval, min_gap)
        return MassiveDataSource(api_key, price_cache, poll_interval=interval, min_gap=min_gap)

    seed_raw = os.environ.get("MARKET_SIM_SEED", "").strip()
    seed = int(seed_raw) if seed_raw.isdigit() else None
    logger.info("Market data source: GBM simulator (seed=%s)", seed)
    return SimulatorDataSource(price_cache, seed=seed)
```

### Configuration summary

| Variable / param | Default | Where | Purpose |
|---|---|---|---|
| `MASSIVE_API_KEY` | empty | env | Non-empty selects Massive |
| `MASSIVE_POLL_INTERVAL` | `15` | env | Regular poll cadence (R11). Paid tiers: 2–5 |
| `MASSIVE_MIN_GAP` | `12` | env | Hard floor between requests. Free tier must be ≥ 12 |
| `MARKET_SIM_SEED` | unset | env | Reproducible simulator runs (E2E, demos) |
| `update_interval` | `0.5`s | `SimulatorDataSource` | Simulator tick |
| `event_probability` | `0.001` | `GBMSimulator` | Shock chance per ticker per tick |
| SSE push interval | `0.5`s | `stream.py` | Frame cadence |
| SSE heartbeat | `15`s | `stream.py` | Idle keep-alive comment |
| SSE `retry` | `1000`ms | `stream.py` | EventSource reconnect delay |
| New-ticker price wait | `3`s sim / `min_gap + 3`s Massive | routes | `wait_for_price` timeout |

Add the three new variables to `.env.example` as commented-out optional entries, in
line with `PLAN.md` §5.

---

## 10. SSE Stream, Lifecycle & Platform Integration

### 10.1 `stream.py` (fixes F2, F10, F12, F17, F18)

```python
from __future__ import annotations

import asyncio
import json
import logging
import time
from collections.abc import AsyncGenerator

from fastapi import APIRouter, Depends, Request
from fastapi.responses import StreamingResponse

from app.auth import require_user  # 401 unless a valid signed session cookie (PLAN §8)

from .cache import PriceCache

logger = logging.getLogger(__name__)

PUSH_INTERVAL = 0.5
HEARTBEAT_INTERVAL = 15.0

router = APIRouter(prefix="/api/stream", tags=["streaming"])


class _FrameCache:
    """Serialize each cache version once, however many clients are connected (F18)."""

    def __init__(self, cache: PriceCache) -> None:
        self._cache = cache
        self._version = -1
        self._frame = ""

    def frame(self) -> tuple[int, str]:
        v = self._cache.version
        if v != self._version:
            data = {t: u.to_dict() for t, u in self._cache.get_all().items()}
            self._frame = f"data: {json.dumps(data, separators=(',', ':'))}\n\n"
            self._version = v
        return self._version, self._frame


@router.get("/prices", dependencies=[Depends(require_user)])
async def stream_prices(request: Request) -> StreamingResponse:
    """SSE: one `data:` frame with ALL tracked tickers whenever the cache changes
    (checked every 500ms), plus a `: ping` comment every 15s when idle."""
    state = request.app.state
    if not hasattr(state, "sse_frames"):
        state.sse_frames = _FrameCache(state.price_cache)
    return StreamingResponse(
        _generate_events(state.sse_frames, request),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "Connection": "keep-alive", "X-Accel-Buffering": "no"},
    )


async def _generate_events(frames: _FrameCache, request: Request) -> AsyncGenerator[str, None]:
    yield "retry: 1000\n\n"
    last_version = -1
    last_sent = time.monotonic()
    try:
        while not await request.is_disconnected():
            version, frame = frames.frame()
            if version != last_version:
                last_version = version
                yield frame            # an empty watchlist sends `data: {}` so the UI clears
                last_sent = time.monotonic()
            elif time.monotonic() - last_sent >= HEARTBEAT_INTERVAL:
                yield ": ping\n\n"     # SSE comment: ignored by EventSource, keeps proxies alive
                last_sent = time.monotonic()
            await asyncio.sleep(PUSH_INTERVAL)
    except asyncio.CancelledError:
        logger.debug("SSE stream cancelled")
        raise                          # F17
```

Behavior changes from v1:
- Auth is required. `EventSource` sends same-origin cookies automatically, so the
  frontend needs no changes. On a 401, `EventSource` fires `onerror` and stops
  reconnecting (non-200), and the frontend then routes to the login screen.
- An empty cache sends `data: {}` instead of nothing, so removing the last watched
  ticker clears the UI.
- `create_stream_router(cache)` stays as a thin compatibility shim that returns the
  module-level `router` (idempotent). It can be removed once `main.py` includes
  `router` directly.

**Caddy.** `reverse_proxy` flushes `text/event-stream` responses immediately by default,
so no `flush_interval` is needed. The heartbeat covers idle-timeout middleboxes.

### 10.2 Lifespan wiring (`app/main.py`)

Routers are included at import time. Only the *state* is created in the lifespan (F10).

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    await init_db()                                   # lazy schema + seed + admin bootstrap (PLAN §7)
    cache = PriceCache()
    source = create_market_data_source(cache)
    app.state.price_cache = cache
    app.state.market_source = source

    await source.start(await tracked_tickers_from_db())   # watchlist ∪ open positions (§10.4)

    snapshot_task = asyncio.create_task(portfolio_snapshot_loop(app), name="portfolio-snapshots")
    try:
        yield
    finally:
        snapshot_task.cancel()
        await asyncio.gather(snapshot_task, return_exceptions=True)
        await source.stop()


app = FastAPI(title="FinAlly", lifespan=lifespan)
app.include_router(auth_router)
app.include_router(stream_router)          # from app.market.stream import router as stream_router
app.include_router(portfolio_router)
app.include_router(watchlist_router)
app.include_router(chat_router)
app.include_router(health_router)
# static export mounted last, so /api/* routes take precedence


def get_price_cache(request: Request) -> PriceCache:
    return request.app.state.price_cache


def get_market_source(request: Request) -> MarketDataSource:
    return request.app.state.market_source
```

The dependencies take `Request` instead of closing over the module-level `app` (v1),
so tests can build their own app instance.

### 10.3 Watchlist & trade routes: the market-data contract

```python
@router.post("/api/watchlist", status_code=201)
async def add_to_watchlist(
    body: WatchlistAdd,
    user: User = Depends(require_user),
    db: DB = Depends(get_db),
    source: MarketDataSource = Depends(get_market_source),
    cache: PriceCache = Depends(get_price_cache),
):
    try:
        ticker = normalize_ticker(body.ticker)
    except InvalidTickerError as e:
        raise HTTPException(400, str(e))
    await db.add_watchlist(user.id, ticker)                 # idempotent on UNIQUE(user_id, ticker)
    await sync_tracked_tickers(db, source)                  # §10.4
    price = await cache.wait_for_price(ticker, timeout=new_ticker_wait(source))
    return {"ticker": ticker, "price": price}               # price may be null on Massive


@router.post("/api/portfolio/trade")
async def trade(body: TradeRequest, user=Depends(require_user), db=Depends(get_db),
                source=Depends(get_market_source), cache=Depends(get_price_cache)):
    ticker = normalize_ticker(body.ticker)
    price = cache.get_price(ticker)
    if price is None:
        raise HTTPException(400, f"No live price for {ticker} yet. Add it to the watchlist and retry.")
    result = await execute_trade(db, user.id, ticker, body.side, body.quantity, price)
    await sync_tracked_tickers(db, source)   # a full sell of a ticker that isn't watched stops tracking it
    await record_portfolio_snapshot(db, user.id, cache)     # PLAN §7: snapshot right after each trade
    return result
```

**Chat (`PLAN.md` §9 step 6), per LLM trade:** `normalize_ticker` → if not watched,
`db.add_watchlist` + `sync_tracked_tickers` → `await cache.wait_for_price(ticker,
new_ticker_wait(source))` → execute, or record `"No price available for X"` as that
action's error. With the §8.3 early poll, this succeeds on Massive within about 12s at
worst on the free tier (typically immediately, if the minimum gap has already passed), and
immediately on the simulator.

```python
def new_ticker_wait(source: MarketDataSource) -> float:
    return 3.0 if source.status().source == "simulator" else getattr(source, "_min_gap", 12.0) + 3.0
```

### 10.4 Tracked-ticker reconciliation (fixes F4, keeps R18/R28)

The **set of tickers the source tracks = ∪ over users of (watchlist ∪ tickers with
quantity > 0)**. Rather than sprinkling conditional `add_ticker`/`remove_ticker` calls
through the routes (v1 §11), one idempotent function derives the desired set from the DB
and diffs it against the source. Call it at startup, after any watchlist change (REST or
chat), and after every trade.

```python
# backend/app/services/tracked_tickers.py
async def desired_tickers(db) -> set[str]:
    rows = await db.fetch_all(
        """
        SELECT ticker FROM watchlist
        UNION
        SELECT ticker FROM positions WHERE quantity > 0
        """
    )
    return {r["ticker"] for r in rows}


_sync_lock = asyncio.Lock()


async def sync_tracked_tickers(db, source: MarketDataSource) -> None:
    async with _sync_lock:  # serialize concurrent route calls so diffs don't interleave
        desired = await desired_tickers(db)
        current = set(source.get_tickers())
        for t in sorted(desired - current):
            await source.add_ticker(t)
        for t in sorted(current - desired):
            await source.remove_ticker(t)
```

Consequences:
- Removing a watched ticker that's still held keeps it priced, so the positions table,
  heatmap, and snapshots stay correct. The SSE stream still carries it, so the frontend
  should render the **watchlist from `/api/watchlist`**, not from the SSE keys, and use
  SSE only to look up prices.
- The query has no `user_id` filter on purpose. That's the multi-user union (R28). Per-user
  filtering happens in the routes, not in the price feed.

### 10.5 Health (F11)

```python
@router.get("/api/health")
async def health(request: Request):
    st = request.app.state.market_source.status()
    stale_after = 5.0 if st.source == "simulator" else 4 * _poll_interval(request)
    stale = st.last_success_at is None or time.time() - st.last_success_at > stale_after
    body = {"status": "ok" if st.running and not stale else "degraded", "market_data": st.to_dict()}
    return JSONResponse(body, status_code=200 if body["status"] == "ok" else 503)
```

Returning 503 when degraded makes the external uptime check from `PLAN.md` §11 page on a
dead feed or a bad key. This endpoint stays unauthenticated (§8), and it exposes no
secrets: `last_error` comes from exception text, and the Massive SDK doesn't echo the key
in its errors. Truncate it to 200 characters anyway. Decide whether Docker's own
`HEALTHCHECK` should use a separate liveness path that ignores feed staleness, so a
Massive outage doesn't restart-loop the container (see §13).

### 10.6 Frontend consumption (reference)

```ts
type PriceTick = {
  ticker: string; price: number; previous_price: number; timestamp: number;
  change: number; change_percent: number; direction: "up" | "down" | "flat";
  reference_price: number | null; day_change: number | null; day_change_percent: number | null;
};

const es = new EventSource("/api/stream/prices");          // same-origin cookie auth
es.onopen = () => setStatus("connected");
es.onmessage = (e) => {
  const frame: Record<string, PriceTick> = JSON.parse(e.data);
  for (const t of Object.values(frame)) {
    if (t.timestamp !== lastSeen[t.ticker]) {               // only real updates extend sparklines
      lastSeen[t.ticker] = t.timestamp;
      pushSparklinePoint(t.ticker, t.timestamp, t.price);
      if (t.direction !== "flat") flash(t.ticker, t.direction);
    }
  }
  setPrices(frame);
};
es.onerror = () => setStatus(es.readyState === EventSource.CLOSED ? "disconnected" : "reconnecting");
```

The `timestamp` dedupe matters on Massive. A frame is sent whenever *any* ticker
changes, so an unchanged ticker must not get a duplicate sparkline point or a spurious
flash.

---

## 11. Testing

### 11.1 Regression tests that would have caught F1 (use real SDK models, never `MagicMock`)

```python
# tests/market/test_massive_parsing.py
from massive.rest.models import TickerSnapshot
from app.market.massive_client import parse_snapshot


def snap(d: dict) -> TickerSnapshot:
    return TickerSnapshot.from_dict(d)


def test_last_trade_price_and_nanosecond_timestamp():
    s = snap({"ticker": "AAPL", "lastTrade": {"p": 190.33, "t": 1707580799123456789},
              "prevDay": {"c": 189.10}})
    ticker, price, ts, ref = parse_snapshot(s)
    assert (ticker, price, ref) == ("AAPL", 190.33, 189.10)
    assert ts == pytest.approx(1707580799.123, abs=1e-3)   # seconds, not ms/ns


def test_falls_back_to_minute_then_day_then_prev_close():
    assert parse_snapshot(snap({"ticker": "X", "min": {"c": 11.0}, "day": {"c": 12.0}}))[1] == 11.0
    assert parse_snapshot(snap({"ticker": "X", "day": {"c": 12.0}}))[1] == 12.0
    assert parse_snapshot(snap({"ticker": "X", "prevDay": {"c": 13.0}}))[1] == 13.0


def test_no_price_returns_none():
    assert parse_snapshot(snap({"ticker": "X"})) is None
    assert parse_snapshot(snap({"ticker": "X", "lastTrade": {"p": 0}})) is None
```

### 11.2 Behavior tests for the fixes

```python
async def test_removed_ticker_not_resurrected_by_inflight_poll():          # F6
    cache = PriceCache()
    src = MassiveDataSource("k", cache)
    src._client, src._tickers = object(), ["AAPL", "TSLA"]
    started = threading.Event(); release = threading.Event()

    def slow_fetch(tickers):
        started.set(); release.wait(2)
        return [snap({"ticker": t, "lastTrade": {"p": 100.0, "t": 1}}) for t in tickers]

    src._fetch_snapshots = slow_fetch
    poll = asyncio.create_task(src._poll_once())
    await asyncio.to_thread(started.wait, 2)
    await src.remove_ticker("TSLA")
    release.set(); await poll
    assert "TSLA" not in cache and "AAPL" in cache


async def test_add_ticker_triggers_early_poll():                           # F5
    ...  # poll_interval=60, min_gap=0: after add_ticker, price appears in < 1s


def test_rest_client_has_no_sdk_retries():                                 # F7
    ...  # after start(): src._client.retries == 0


def test_remove_bumps_version():                                           # F9
    c = PriceCache(); c.update("A", 1.0); v = c.version
    c.remove("A"); assert c.version == v + 1


def test_day_change_uses_reference_price():                                # F3
    c = PriceCache()
    c.update("A", 100.0, reference_price=90.0)
    u = c.update("A", 99.0)            # reference carried forward
    assert u.day_change_percent == pytest.approx(10.0)


def test_normalize_ticker():                                               # F8
    assert normalize_ticker(" brk.b ") == "BRK.B"
    for bad in ["", "TOOLONG", "A B", "1234", "AAPL;"]:
        with pytest.raises(InvalidTickerError):
            normalize_ticker(bad)


def test_simulator_seed_is_reproducible():                                 # F16
    a, b = GBMSimulator(["AAPL", "MSFT"], seed=7), GBMSimulator(["AAPL", "MSFT"], seed=7)
    assert [a.step() for _ in range(50)] == [b.step() for _ in range(50)]


def test_full_default_watchlist_cholesky():                                # previous review 4.2
    GBMSimulator(list(SEED_PRICES))  # must not raise
```

### 11.3 SSE tests (previous review 4.2: `stream.py` was at 31% coverage)

Test the generator directly with a fake request. That avoids streaming through an ASGI
client, which can hang on infinite responses:

```python
class FakeRequest:
    def __init__(self, disconnect_after: int):
        self._n = disconnect_after
    async def is_disconnected(self):
        self._n -= 1
        return self._n < 0


async def test_stream_emits_retry_then_frame(monkeypatch):
    monkeypatch.setattr(stream, "PUSH_INTERVAL", 0)
    cache = PriceCache(); cache.update("AAPL", 190.0)
    out = [c async for c in stream._generate_events(stream._FrameCache(cache), FakeRequest(1))]
    assert out[0] == "retry: 1000\n\n"
    assert json.loads(out[1].removeprefix("data: "))["AAPL"]["price"] == 190.0


def test_stream_requires_auth(client_without_session):
    assert client_without_session.get("/api/stream/prices").status_code == 401  # F2
```

### 11.4 E2E (`test/`)
Run the app with `MARKET_SIM_SEED=42` and `LLM_MOCK=true`. Add a test for "add a
ticker, then trade it through chat", and a test for "remove a held ticker from the
watchlist: the position is still priced".

---

## 12. Implementation Checklist

In dependency order. Each step is one small PR with its tests.

1. **F1** Add `parse_snapshot()` to `massive_client.py`, plus the real-SDK tests (§11.1). Replace the `MagicMock` parsing tests.
2. **F6, F7, F13, F14** `MassiveDataSource` rewrite per §8.3; `factory.py` env config per §9; `.env.example`.
3. **F3, F9** `reference_price` and day-change fields in `PriceUpdate`; `PriceCache` changes and `wait_for_price` (§5).
4. **F8** Add `tickers.py`; normalize in both sources and export from `app.market`.
5. **F11** `SourceStatus` + `status()` on both sources (§6.1, §7.4).
6. **F15, F16** Simulator seedable RNG, PSD guard, refreshed seeds (§7).
7. **F2, F10, F12, F17, F18** `stream.py` rewrite (§10.1). Depends on `app.auth.require_user` from the auth work.
8. **F4** `sync_tracked_tickers` + lifespan wiring (§10.2, §10.4). Depends on the DB layer.
9. **R22** Chat executor uses `wait_for_price` (§10.3). Depends on the chat work.
10. **R26** `/api/health` reports feed status (§10.5).
11. Update `MARKET_DATA_SUMMARY.md` and `backend/CLAUDE.md` to point at this document and the new public API.

Items 1–6 are self-contained in `backend/app/market/` and can start now. Items 7–10
land together with the auth, DB, and chat work they plug into.

---

## 13. Open Questions

1. **Massive plan entitlement.** It's unclear whether the free **Basic** stocks plan can
   call `/v2/snapshot/...`. Historically (as Polygon.io) snapshots required a paid stocks
   plan, and the Basic tier returned `NOT_AUTHORIZED` (HTTP 403) for them. I couldn't
   verify this for the current Massive plans, because massive.com is blocked from this
   environment. `MASSIVE_API.md` §2 assumes the free tier works at 15s. **Someone with a
   key should run one `get_snapshot_all` call on a free key.** If it's refused, the
   options are: (a) document "Massive requires Starter+" in `PLAN.md` §5/§6, or (b) add a
   free-tier fallback that polls `/v2/aggs/ticker/{t}/prev` per ticker, which gives
   end-of-day prices only and costs one call per ticker. Either way, the 403 now surfaces
   through `/api/health` instead of failing silently.
2. **Should `/api/health` affect container restarts?** Recommendation: Docker
   `HEALTHCHECK` hits a liveness-only path (`/api/health?probe=live`, always 200 while
   the process serves requests), and the external uptime monitor hits the full check.
   Otherwise a Massive outage restart-loops the app for no benefit.
3. **Daily reference on the simulator.** This design uses "price when first seeded in
   this process", so it resets on restart. An alternative is to persist a daily open per
   ticker, which needs a table. Recommendation: keep it in memory. It's a simulator, and
   `PLAN.md` already accepts ephemeral sparklines.
4. **Watchlist size cap.** The Massive snapshot URL takes a comma-separated query string,
   and the Cholesky rebuild is O(n³). Suggest a cap of 50 tickers per user, enforced in
   the watchlist route and the chat executor.
