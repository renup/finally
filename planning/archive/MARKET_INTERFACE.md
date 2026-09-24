# Market Data Interface — Design

Design and reference for FinAlly's unified market-data layer: one Python interface that
both the GBM simulator and the Massive API implement, so every downstream consumer
(SSE stream, portfolio valuation, trade execution) is written once against an
abstraction and never has to know or care which source is live.

**Status: implemented.** This describes the design as built in `backend/app/market/`.
See `MARKET_SIMULATOR.md` for the simulator's internals and `MASSIVE_API.md` for the
Massive API research this design is built on.

---

## 1. Goals

- One interface, two implementations, selected purely by environment variable
  (`MASSIVE_API_KEY` set → Massive; unset → simulator) — per `PLAN.md` §6.
- Downstream code (SSE, trades, portfolio math) reads from a shared cache, never calls
  a data source directly. Swapping the source never touches downstream code.
- A single batched call per poll cycle covers the *entire* watchlist, regardless of
  size — this is what keeps the Massive free-tier rate limit (5 req/min, see
  `MASSIVE_API.md` §2) independent of how many tickers the user is watching.
- Ticker set is dynamic — the watchlist changes at runtime (`add_ticker` /
  `remove_ticker`) without restarting the background task.

## 2. Module Layout

```
backend/app/market/
├── __init__.py        # Public exports
├── models.py           # PriceUpdate — immutable price snapshot
├── cache.py            # PriceCache — thread-safe shared store
├── interface.py         # MarketDataSource — abstract base class
├── simulator.py        # GBMSimulator + SimulatorDataSource
├── massive_client.py     # MassiveDataSource (Massive/Polygon REST client)
├── seed_prices.py       # Seed prices, per-ticker vol/drift, correlation groups
└── stream.py            # SSE router, reads from PriceCache
```

Public surface (`app/market/__init__.py`):

```python
from app.market import PriceCache, PriceUpdate, MarketDataSource, create_market_data_source, create_stream_router
```

## 3. Core Types

### 3.1 `PriceUpdate` (`models.py`)

An immutable, frozen dataclass — one price observation for one ticker at one instant.
Both implementations produce these; nothing downstream constructs one directly except
via `PriceCache.update()`.

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds

    @property
    def change(self) -> float: ...          # price - previous_price, rounded
    @property
    def change_percent(self) -> float: ...  # % change, rounded
    @property
    def direction(self) -> str: ...          # "up" | "down" | "flat"
    def to_dict(self) -> dict: ...            # JSON-serializable for SSE
```

Deriving `change`/`change_percent`/`direction` as properties (rather than storing them)
means the cache is the single source of truth for "previous" — a data source only ever
needs to hand over the latest price; the delta math happens in one place.

### 3.2 `PriceCache` (`cache.py`)

Thread-safe (`threading.Lock`) in-memory dict, keyed by ticker. This is the seam between
producers and consumers:

- **Writers**: exactly one active `MarketDataSource` at a time (simulator or Massive —
  never both).
- **Readers**: the SSE endpoint, portfolio valuation, trade execution — all read-only.

```python
cache.update(ticker, price, timestamp=None) -> PriceUpdate   # computes previous/change/direction
cache.get(ticker) -> PriceUpdate | None
cache.get_price(ticker) -> float | None
cache.get_all() -> dict[str, PriceUpdate]                     # shallow copy snapshot
cache.remove(ticker) -> None
cache.version -> int                                           # bumped on every update()
```

`version` is a monotonic counter used purely so the SSE loop (§5) can cheaply detect
"has anything changed since I last sent a frame" without diffing dictionaries.

A plain `Lock` (not asyncio-aware) is correct here: writes happen either on FastAPI's
event loop thread (simulator's `asyncio.create_task` loop) or from a `asyncio.to_thread`
worker (Massive's synchronous SDK call) — both need the same conventional thread lock,
not an `asyncio.Lock`, because the Massive path genuinely crosses a thread boundary.

### 3.3 `MarketDataSource` (`interface.py`)

The abstract contract both implementations satisfy:

```python
class MarketDataSource(ABC):
    @abstractmethod
    async def start(self, tickers: list[str]) -> None: ...
    @abstractmethod
    async def stop(self) -> None: ...
    @abstractmethod
    async def add_ticker(self, ticker: str) -> None: ...
    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None: ...
    @abstractmethod
    def get_tickers(self) -> list[str]: ...
```

Lifecycle contract:

```python
source = create_market_data_source(cache)
await source.start(["AAPL", "GOOGL", ...])   # exactly once
await source.add_ticker("TSLA")               # any time after start
await source.remove_ticker("GOOGL")
await source.stop()                            # safe to call more than once
```

`start()` is expected to seed the cache immediately (so the very first SSE frame has
data) and then hand off to a background `asyncio.Task` that keeps writing on its own
schedule. `stop()` cancels that task and is idempotent.

### 3.4 `create_market_data_source()` (`factory.py`)

The entire source-selection decision, in one function:

```python
def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    return SimulatorDataSource(price_cache=price_cache)
```

Everything above this line — FastAPI startup, the SSE router, trade execution — imports
`MarketDataSource` and `create_market_data_source`, never `SimulatorDataSource` or
`MassiveDataSource` directly. That's the whole point of the abstraction: adding a third
source later (a different vendor, a replay-from-file source for tests) means writing one
new class and one new branch here.

## 4. The Two Implementations

Both live behind the same interface but have very different internal shapes, because
their upstream data has very different shapes:

| | `SimulatorDataSource` | `MassiveDataSource` |
|---|---|---|
| Update cadence | 500ms, in-process | 15s, polled REST |
| Update source | `GBMSimulator.step()` (pure computation) | `client.get_snapshot_all(...)` (network I/O) |
| Threading | Runs entirely on the event loop | Synchronous SDK call wrapped in `asyncio.to_thread` |
| Failure mode | None (can't fail) | Network/API errors caught per-poll, logged, retried next cycle |
| Ticker add/remove | Immediate — recomputes correlation matrix (see `MARKET_SIMULATOR.md`) | Immediate — takes effect on the *next* poll |

Both convert their native price representation into `cache.update(ticker, price,
timestamp)` calls — that call is the entire adapter. See `MARKET_SIMULATOR.md` for the
simulator side; see `MASSIVE_API.md` §3.1 for the exact Massive endpoint and response
shape the `MassiveDataSource` adapter parses (`snap.last_trade.price`,
`snap.last_trade.timestamp / 1000.0` — Massive nanosecond→millisecond, then converted to
the cache's Unix-seconds float).

## 5. SSE Streaming (`stream.py`)

The only consumer of `PriceCache` that ships in the current build. `create_stream_router(cache)`
returns a FastAPI `APIRouter` exposing `GET /api/stream/prices`:

- Long-lived `StreamingResponse`, `text/event-stream`.
- Polls `cache.version` every 500ms; only serializes and sends a frame when the version
  has changed since the last frame — this decouples the SSE tick rate from however often
  the underlying source actually produces new prices (500ms for the simulator, 15s for
  Massive — the endpoint doesn't need to know which).
- Sends `retry: 1000` up front so `EventSource`'s built-in reconnect logic retries after
  1s on disconnect.
- Detects client disconnect via `request.is_disconnected()` and exits cleanly.
- One frame contains *all* current prices, keyed by ticker:
  `data: {"AAPL": {...PriceUpdate.to_dict()...}, "MSFT": {...}}\n\n`

## 6. What's Deliberately Out of Scope

- **No EOD/historical endpoints wired up.** `MASSIVE_API.md` §4 documents Massive's
  grouped-daily and aggregates-range endpoints, but nothing in the current watchlist/SSE
  flow needs them — sparklines are accumulated client-side from the live stream and
  explicitly not backfilled (`PLAN.md` §10). If a future feature needs real historical
  data (e.g., a longer main chart than "since page load"), it plugs in as a *separate*
  read path against the Massive REST client — it doesn't belong in `MarketDataSource`,
  whose whole contract is "push the current price into the cache."
- **No push/WebSocket.** Both sources are poll-based by design (`PLAN.md` §3) — even
  Massive's real-time-capable plans are consumed via REST polling, not their WebSocket
  feed, to keep one transport model (SSE) between backend and frontend regardless of
  upstream source.
- **Single source active at a time.** The interface doesn't support blending simulator
  and Massive data, or multiple Massive polls at different intervals — one
  `MarketDataSource` instance, chosen once at process startup.
