# FinAlly — AI Trading Workstation

## Project Specification

## 1. Vision

FinAlly (Finance Ally) is a visually stunning AI-powered trading workstation that streams live market data, lets users trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades on the user's behalf. It looks and feels like a modern Bloomberg terminal with an AI copilot.

This is the capstone project for an agentic AI coding course. It is built entirely by Coding Agents demonstrating how orchestrated AI agents can produce a production-quality full-stack application. Agents interact through files in `planning/`.

Beyond the course exercise, this app is intended to run as a real, internet-facing personal app on a self-managed VPS — not just a local demo. The portfolio remains simulated/paper-trading money indefinitely (no real brokerage integration is planned), but because it's reachable over the internet and the AI can execute trades autonomously, it needs a login gate, HTTPS, backups, and basic operational hardening from the start. See §7, §8, §9, §11 for what that adds to the base design.

## 2. User Experience

### First Launch

The user runs a single Docker command (or a provided start script). A browser opens to the app's URL. On first boot, a single admin account is bootstrapped from environment variables (`ADMIN_EMAIL` / `ADMIN_PASSWORD`, see §5) — the deployer logs in once with that email/password. There is no public self-signup; this stays a single-user app for now, but the schema and auth layer (§7, §8) are built so multi-user signup can be enabled later without a rewrite. After logging in, they immediately see:

- A watchlist of 10 default tickers with live-updating prices in a grid
- $10,000 in virtual cash
- A dark, data-rich trading terminal aesthetic
- An AI chat panel ready to assist

### What the User Can Do

- **Watch prices stream** — prices flash green (uptick) or red (downtick) with subtle CSS animations that fade
- **View sparkline mini-charts** — price action beside each ticker in the watchlist, accumulated on the frontend from the SSE stream since page load (sparklines fill in progressively)
- **Click a ticker** to see a larger detailed chart in the main chart area
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog
- **Monitor their portfolio** — a heatmap (treemap) showing positions sized by weight and colored by P&L, plus a P&L chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change
- **Chat with the AI assistant** — ask about their portfolio, get analysis, and have the AI execute trades and manage the watchlist through natural language
- **Manage the watchlist** — add/remove tickers manually or via the AI chat

### Visual Design

- **Dark theme**: backgrounds around `#0d1117` or `#1a1a2e`, muted gray borders, no pure black
- **Price flash animations**: brief green/red background highlight on price change, fading over ~500ms via CSS transitions
- **Connection status indicator**: a small colored dot (green = connected, yellow = reconnecting, red = disconnected) visible in the header
- **Professional, data-dense layout**: inspired by Bloomberg/trading terminals — every pixel earns its place
- **Responsive but desktop-first**: optimized for wide screens, functional on tablet

### Color Scheme
- Accent Yellow: `#ecad0a`
- Blue Primary: `#209dd7`
- Purple Secondary: `#753991` (submit buttons)

## 3. Architecture Overview

### Single Container, Single Port

```
┌─────────────────────────────────────────────────┐
│  Docker Container (port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST endpoints             │
│  ├── /api/stream/*   SSE streaming              │
│  └── /*              Static file serving         │
│                      (Next.js export)            │
│                                                 │
│  SQLite database (volume-mounted)               │
│  Background task: market data polling/sim        │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file at `db/finally.db`, volume-mounted for persistence
- **Real-time data**: Server-Sent Events (SSE) — simpler than WebSockets, one-way server→client push, works everywhere
- **AI integration**: LiteLLM → OpenRouter (Cerebras for fast inference), with structured outputs for trade execution
- **Market data**: Environment-variable driven — simulator by default, real data via Massive API if key provided

### Why These Choices

| Decision | Rationale |
|---|---|
| SSE over WebSockets | One-way push is all we need; simpler, no bidirectional complexity, universal browser support |
| Static Next.js export | Single origin, no CORS issues, one port, one container, simple deployment |
| SQLite over Postgres | No auth = no multi-user = no need for a database server; self-contained, zero config |
| Single Docker container | Students run one command; no docker-compose for production, no service orchestration |
| uv for Python | Fast, modern Python project management; reproducible lockfile; what students should learn |
| Market orders only | Eliminates order book, limit order logic, partial fills — dramatically simpler portfolio math |
| Single bootstrapped user + session auth (no public signup yet) | The app runs on a public VPS, so it needs a real login gate — but full multi-user signup isn't needed yet. The `users` table and `user_id` foreign keys (§7) are built now so multi-user support is a signup flow, not a schema rewrite |
| Caddy reverse proxy for TLS | Automatic HTTPS via Let's Encrypt with minimal config; appropriate for a single-VPS deployment (§11) |

---

## 4. Directory Structure

```
finally/
├── frontend/                 # Next.js TypeScript project (static export)
├── backend/                  # FastAPI uv project (Python)
│   └── db/                   # Schema definitions, seed data, migration logic
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document
│   └── ...                   # Additional agent reference docs
├── scripts/
│   ├── start_mac.sh          # Launch Docker container (macOS/Linux)
│   ├── stop_mac.sh           # Stop Docker container (macOS/Linux)
│   ├── start_windows.ps1     # Launch Docker container (Windows PowerShell)
│   └── stop_windows.ps1      # Stop Docker container (Windows PowerShell)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # Volume mount target (SQLite file lives here at runtime)
│   └── .gitkeep              # Directory exists in repo; finally.db is gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── docker-compose.yml        # Runs the app + Caddy (reverse proxy/TLS) together for production
├── Caddyfile                 # Caddy reverse proxy config (domain, automatic HTTPS, proxy to app)
├── .env                      # Environment variables (gitignored, .env.example committed)
└── .gitignore
```

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents.
- **`backend/db/`** contains the schema as versioned `.sql` files (the `CREATE TABLE` statements, run in order) plus the seed-data logic in Python. The backend lazily initializes the database on first request — running these `.sql` files and seeding default data if the SQLite file doesn't exist or is empty. Once schema changes need to land against a live database (post-launch, per §7), these same `.sql` files become the input to a small migration runner rather than being replaced by one.
- **`db/`** at the top level is the runtime volume mount point. The SQLite file (`db/finally.db`) is created here by the backend and persists across container restarts via Docker volume.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains start/stop scripts that wrap Docker commands.

---

## 5. Environment Variables

```bash
# Required: OpenRouter API key for LLM chat functionality
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: Massive (Polygon.io) API key for real market data
# If not set, the built-in market simulator is used (recommended for most users)
MASSIVE_API_KEY=

# Optional: Set to "true" for deterministic mock LLM responses (testing)
LLM_MOCK=false

# Required in production: bootstraps the single admin account on first boot
ADMIN_EMAIL=you@example.com
ADMIN_PASSWORD=change-me-before-deploying

# Required in production: signs session cookies. Generate with `openssl rand -hex 32`
SESSION_SECRET=
```

### Behavior

- If `MASSIVE_API_KEY` is set and non-empty → backend uses Massive REST API for market data
- If `MASSIVE_API_KEY` is absent or empty → backend uses the built-in market simulator
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (for E2E tests)
- The backend reads `.env` from the project root (mounted into the container or read via docker `--env-file`)
- On first boot, if no row exists in `users`, the backend creates one from `ADMIN_EMAIL` / `ADMIN_PASSWORD` (password is hashed before storage, plaintext is never persisted). This only happens once — changing `ADMIN_EMAIL`/`ADMIN_PASSWORD` later does **not** retroactively update the existing `users` row (so a future in-app "change password" feature isn't silently overwritten on restart). To reset credentials manually before that feature exists, delete the `users` row via `sqlite3 db/finally.db` and restart the container to re-bootstrap
- `ADMIN_EMAIL`, `ADMIN_PASSWORD`, and `SESSION_SECRET` have no built-in defaults — if any is missing or empty, the backend fails fast at startup with a clear error instead of booting with a guessable default or a disabled login. This is the same locally and in production: first launch always means copying `.env.example` to `.env` and filling in real values before `docker run` / `docker compose up`
- Sessions are stateless, signed cookies (signed with `SESSION_SECRET`, e.g. via `itsdangerous` or a JWT) — there is no server-side `sessions` table to manage or expire. The cookie is set `HttpOnly`, `Secure`, and `SameSite=Lax` (Lax rather than Strict so a normal top-level navigation to the app still carries it), with a fixed expiry (e.g. 7 days) encoded in the signed payload so the server doesn't need to track it. `SameSite=Lax` combined with cookie-only auth means state-changing endpoints (trade, chat, watchlist) aren't reachable via a simple cross-site form or image request, so no separate CSRF token is needed. Logout just clears the cookie client-side — as with any stateless-cookie scheme, a copied cookie stays valid until it expires or `SESSION_SECRET` is rotated (which invalidates all sessions at once)

### Production Secrets Handling

- `.env` lives only on the VPS — it is never committed, never baked into the Docker image (`COPY`'d), and is passed at run time via `docker run --env-file .env`
- File permissions on the VPS are locked down (`chmod 600 .env`)
- `SESSION_SECRET` and `ADMIN_PASSWORD` must be changed from any default/example value before the first production deploy

---

## 6. Market Data

### Two Implementations, One Interface

Both the simulator and the Massive client implement the same abstract interface. The backend selects which to use based on the environment variable. All downstream code (SSE streaming, price cache, frontend) is agnostic to the source.

### Simulator (Default)

- Generates prices using geometric Brownian motion (GBM) with configurable drift and volatility per ticker
- Updates at ~500ms intervals
- Correlated moves across tickers via simple sector grouping — tickers in the same group (e.g., a "tech" cluster: AAPL, GOOGL, MSFT, AMZN, NVDA, META) share a common noise factor each tick, so they tend to move together
- Occasional random "events" — sudden 2-5% moves on a ticker for drama
- Starts from realistic seed prices (e.g., AAPL ~$190, GOOGL ~$175, etc.)
- Runs as an in-process background task — no external dependencies
- Runs continuously, 24/7 — no market-hours gating (this is a simulator for demo purposes, always-on by design)

### Massive API (Optional)

- REST API polling (not WebSocket) — simpler, works on all tiers
- Polls for the union of all watched tickers on a configurable interval, using a single batched/snapshot call that covers all watched tickers at once (not one call per ticker) — this is what keeps polling within free-tier rate limits regardless of watchlist size
- Free tier (5 calls/min): poll every 15 seconds
- Paid tiers: poll every 2-15 seconds depending on tier
- Parses REST response into the same format as the simulator

### Shared Price Cache

- A single background task (simulator or Massive poller) writes to an in-memory price cache
- The cache holds the latest price, previous price, and timestamp for each ticker
- SSE streams read from this cache and push updates to connected clients
- This architecture supports future multi-user scenarios without changes to the data layer

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- Server pushes price updates for all tickers known to the system at a regular cadence (~500ms) — in the single-user model this is equivalent to the user's watchlist
- Each SSE event contains ticker, price, previous price, timestamp, and change direction
- Client handles reconnection automatically (EventSource has built-in retry)

---

## 7. Database

### SQLite with Lazy Initialization

The backend checks for the SQLite database on startup (or first request). If the file doesn't exist or tables are missing, it creates the schema and seeds default data (including bootstrapping the admin `users` row from `ADMIN_EMAIL`/`ADMIN_PASSWORD`). This means:

- No separate migration step for a fresh install
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically
- WAL mode is enabled (`PRAGMA journal_mode=WAL`) for better concurrent read/write behavior under real traffic
- **Once the app is live with real data**, "create if missing" is no longer sufficient for future schema changes — any change to an existing production database needs a proper versioned migration step (e.g., a small SQL migration runner, or Alembic), not a lazy-init rewrite. This only kicks in after the first production deploy; the initial build can ship without a migration framework.

### Backups (production)

Docker volumes protect against container removal, not disk failure or accidental data loss — they are not a backup by themselves. A cron job on the VPS runs a nightly `sqlite3 finally.db ".backup /backups/finally-$(date +%F).db"` and syncs the result to off-box storage (e.g., an S3-compatible bucket or a second machine). Restoring means stopping the container, replacing `db/finally.db` with a backup file, and restarting.

### Schema

All tables include a `user_id` column, a foreign key to `users.id`. Today there's exactly one row in `users` (the bootstrapped admin), so in practice every other table has one `user_id` value — but the schema is already shaped for multi-user support without a migration when that's needed.

**users** — Account credentials (currently a single bootstrapped admin account)
- `id` TEXT PRIMARY KEY (UUID)
- `email` TEXT UNIQUE
- `password_hash` TEXT (bcrypt or argon2 — plaintext password is never stored)
- `created_at` TEXT (ISO timestamp)

**user_profiles** — User state (cash balance)
- `id` TEXT PRIMARY KEY (references `users.id`)
- `cash_balance` REAL (default: `10000.0`)
- `created_at` TEXT (ISO timestamp)

**watchlist** — Tickers the user is watching
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (references `users.id`)
- `ticker` TEXT
- `added_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**positions** — Current holdings (one row per ticker per user)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (references `users.id`)
- `ticker` TEXT
- `quantity` REAL (fractional shares supported, rounded to 6 decimal places; no minimum trade size)
- `avg_cost` REAL
- `updated_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**trades** — Trade history (append-only log)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (references `users.id`)
- `ticker` TEXT
- `side` TEXT (`"buy"` or `"sell"`)
- `quantity` REAL (fractional shares supported, rounded to 6 decimal places; no minimum trade size)
- `price` REAL
- `executed_at` TEXT (ISO timestamp)

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Recorded every 30 seconds by a background task, and immediately after each trade execution. On a fresh start (no trades yet), the chart simply shows a single point at $10,000 — no synthetic history is backfilled. A background task prunes snapshots older than 7 days to keep the table bounded — this is a deliberate limit matching the P&L chart's intended use as a short-term activity view; a longer-range view later should down-sample into a separate rollup table rather than extending raw retention.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (references `users.id`)
- `total_value` REAL
- `recorded_at` TEXT (ISO timestamp)

**chat_messages** — Conversation history with LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (references `users.id`)
- `role` TEXT (`"user"` or `"assistant"`)
- `content` TEXT
- `actions` TEXT (JSON — trades executed, watchlist changes made; null for user messages)
- `created_at` TEXT (ISO timestamp)

### Default Seed Data

- One `users` row bootstrapped from `ADMIN_EMAIL` / `ADMIN_PASSWORD`
- One user profile for that user: `cash_balance=10000.0`
- Ten watchlist entries for that user: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

---

## 8. API Endpoints

All `/api/*` endpoints except `/api/auth/login` and `/api/health` require a valid session cookie; unauthenticated requests get a 401.

### Auth
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/auth/login` | Log in with `{email, password}`; sets a session cookie |
| POST | `/api/auth/logout` | Clears the session cookie |
| GET | `/api/auth/me` | Returns the current logged-in user, or 401 |

`/api/auth/login` is rate-limited per source IP (e.g. 5 attempts/minute, independent of the per-session chat limiter in §9) to blunt credential brute-forcing against the single admin account — same in-process limiter pattern as §9, no external service required.

### Market Data
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates |

### Portfolio
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio` | Current positions, cash balance, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a trade: `{ticker, quantity, side}` |
| GET | `/api/portfolio/history` | Portfolio value snapshots over time (for P&L chart) |

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist tickers with latest prices |
| POST | `/api/watchlist` | Add a ticker: `{ticker}` |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker |

Adding a ticker the active market data source can't resolve (a typo, or a symbol the simulator/Massive don't know) returns `400` with an error message rather than silently accepting it — the frontend surfaces this inline, and the same check applies to tickers the LLM chat flow tries to add (see §9).

### Chat
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/chat` | Send a message, receive complete JSON response (message + executed actions) |
| GET | `/api/chat` | Retrieve chat message history (for restoring the conversation on page load) |

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check (for Docker/deployment) |

---

## 9. LLM Integration

When writing code to make calls to LLMs, use cerebras-inference skill to use LiteLLM via OpenRouter to the `openrouter/openai/gpt-oss-120b` model with Cerebras as the inference provider. Structured Outputs should be used to interpret the results.

There is an OPENROUTER_API_KEY in the .env file in the project root.

### How It Works

When the user sends a chat message, the backend:

1. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
2. Loads the last 10 messages of conversation history from the `chat_messages` table
3. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
4. Calls the LLM via LiteLLM → OpenRouter, requesting structured output, using the cerebras-inference skill
5. Parses the complete structured JSON response
6. Auto-executes any trades or watchlist changes specified in the response — if a trade targets a ticker not currently on the watchlist, the ticker is added to the watchlist first (so it has a live price from the shared cache), then the trade executes. If that ticker can't be resolved by the market data source (see §8 Watchlist), the add and the trade both fail and the error is returned to the LLM the same way a failed cash/share validation is (see **Auto-Execution** below), so it can explain the failure instead of silently skipping it
7. Stores the message and executed actions in `chat_messages`
8. Returns the complete JSON response to the frontend (no token-by-token streaming — Cerebras inference is fast enough that a loading indicator is sufficient)

### Structured Output Schema

The LLM is instructed to respond with JSON matching this schema:

```json
{
  "message": "Your conversational response to the user",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add"}
  ]
}
```

- `message` (required): The conversational text shown to the user
- `trades` (optional): Array of trades to auto-execute. Each trade goes through the same validation as manual trades (sufficient cash for buys, sufficient shares for sells)
- `watchlist_changes` (optional): Array of watchlist modifications

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

If a trade fails validation (e.g., insufficient cash), the error is included in the chat response so the LLM can inform the user.

### Rate Limiting & Cost Control

`/api/chat` is rate-limited per session (e.g., 20 messages/minute) since each call is a paid LLM request against a publicly reachable app — without this, a bug or a bad actor with the login could run up API costs quickly. This is simple in-process rate limiting (no Redis or external service needed — it's a single instance). Trade and watchlist endpoints are intentionally *not* rate-limited: they're free, login-gated, and simulated, so there's nothing costly to protect against.

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- State clearly, when relevant, that this is a simulated portfolio with fake money and not real investment advice
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true`, the backend returns deterministic mock responses instead of calling OpenRouter. This enables:
- Fast, free, reproducible E2E tests
- Development without an API key
- CI/CD pipelines

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Login screen** — shown instead of the terminal UI when there's no valid session; a simple email/password form posting to `/api/auth/login`. On success, redirects to the terminal.
- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), daily change %, and a sparkline mini-chart (accumulated from SSE since page load; sparklines are ephemeral and reset on reload — not backfilled from history)
- **Main chart area** — larger chart for the currently selected ticker, with at minimum price over time. Clicking a ticker in the watchlist selects it here.
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by P&L (green = profit, red = loss)
- **P&L chart** — line chart showing total portfolio value over time, using data from `portfolio_snapshots`
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, % change
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill.
- **AI chat panel** — docked/collapsible sidebar. Message input, scrolling conversation history, loading indicator while waiting for LLM response. Trade executions and watchlist changes shown inline as confirmations.
- **Header** — portfolio total value (updating live), connection status indicator, cash balance

### Technical Notes

- Use `EventSource` for SSE connection to `/api/stream/prices`
- Use Lightweight Charts (canvas-based) for all charts — built for financial/price data and performs well under frequent streaming updates
- Price flash effect: on receiving a new price, briefly apply a CSS class with background color transition, then remove it
- All API calls go to the same origin (`/api/*`) — no CORS configuration needed
- Tailwind CSS for styling with a custom dark theme
- A persistent, small disclaimer ("Simulated portfolio — not real money, not investment advice") is shown in the header or footer, since the AI actively suggests and auto-executes trades

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node 20 slim
  - Copy frontend/
  - npm install && npm run build (produces static export)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/
  - uv sync (install Python dependencies from lockfile)
  - Copy frontend build output into a static/ directory
  - Expose port 8000
  - CMD: uvicorn serving FastAPI app
```

FastAPI serves the static frontend files and all API routes on port 8000.

### Docker Volume

The SQLite database persists via a named Docker volume. For local/dev use, the app container can be run directly:

```bash
docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

The `db/` directory in the project root maps to `/app/db` in the container. The backend writes `finally.db` to this path.

For production, `docker-compose.yml` runs this same app container alongside a Caddy container (see below) rather than exposing port 8000 directly — `docker-compose up -d` is the standard way to start (or restart) the whole stack on the VPS.

### Start/Stop Scripts

**`scripts/start_mac.sh`** (macOS/Linux):
- Builds the Docker image if not already built (or if `--build` flag passed)
- Runs the container with the volume mount, port mapping, and `.env` file
- Prints the URL to access the app
- Opens the browser automatically by default

**`scripts/stop_mac.sh`** (macOS/Linux):
- Stops and removes the running container
- Does NOT remove the volume (data persists)

**`scripts/start_windows.ps1`** / **`scripts/stop_windows.ps1`**: PowerShell equivalents for Windows.

All scripts should be idempotent — safe to run multiple times.

### Production Deployment (VPS)

The intended production target is a self-managed VPS (e.g., a Droplet or EC2 instance), not a managed container platform — this matters because managed platforms like App Runner often run ephemeral, stateless instances that don't support a persistent local volume the way a VPS does.

- **Reverse proxy + TLS**: `docker-compose.yml` runs two services — `app` (this container) and `caddy` (official Caddy image, config from the repo's `Caddyfile`) — on a shared Docker network. Caddy is the only service that publishes ports 80/443 to the host; it terminates HTTPS via automatic Let's Encrypt and reverse-proxies to `app` by its Compose service name (e.g., `reverse_proxy app:8000`). The `app` service does not publish any port to the host at all.
- **Firewall**: only 80/443 and SSH are open to the internet on the VPS itself; the app container is unreachable except through Caddy.
- **Process resilience**: the container runs with `--restart unless-stopped` (or the docker-compose equivalent) so it recovers automatically from crashes or VPS reboots.
- **Backups**: see §7 — nightly SQLite backup synced off-box. Test the restore procedure at least once before relying on it.
- **Monitoring**: an external uptime check (e.g., UptimeRobot, or a cron + curl) polls `/api/health`. Backend logs are structured JSON written to stdout, captured via `docker logs`; a log shipper or error tracker (e.g., Sentry) can be added later without changing this plan.

### Optional Alternative: Managed Container Platforms

The container can still deploy to AWS App Runner, Render, or a similar platform if preferred later, but on those platforms `db/` needs a genuinely persistent, network-attached volume (or the app needs to move off SQLite to a managed database) — a local Docker volume mount, as described above, does not carry over. A Terraform configuration for App Runner may be provided in a `deploy/` directory as a stretch goal, but is not part of the core build.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Auth: password hashing/verification, session creation/expiry, protected routes reject requests without a valid session
- Market data: simulator generates valid prices, GBM math is correct, Massive API response parsing works, both implementations conform to the abstract interface
- Portfolio: trade execution logic, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, selling at a loss)
- LLM: structured output parsing handles all valid schemas, graceful handling of malformed responses, trade validation within chat flow
- API routes: correct status codes, response shapes, error handling

**Frontend (React Testing Library or similar)**:
- Component rendering with mock data
- Price flash animation triggers correctly on price changes
- Watchlist CRUD operations
- Portfolio display calculations
- Chat message rendering and loading state

### E2E Tests (in `test/`)

**Infrastructure**: A separate `docker-compose.test.yml` in `test/` that spins up the app container plus a Playwright container. This keeps browser dependencies out of the production image.

**Environment**: Tests run with `LLM_MOCK=true` by default for speed and determinism.

**Key Scenarios**:
- Auth: logging in with correct/incorrect credentials, session persists across reload, protected routes redirect to the login screen when logged out
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: heatmap renders with correct colors, P&L chart has data points
- AI chat (mocked): send a message, receive a response, trade execution appears inline
- SSE resilience: disconnect and verify reconnection
