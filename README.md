# FinAlly — AI Trading Workstation

A visually stunning AI-powered trading workstation that streams live market data, simulates portfolio trading, and integrates an LLM chat assistant that can analyze positions and execute trades via natural language.

Built entirely by coding agents as a capstone project for an agentic AI coding course. Full specification lives in [`planning/PLAN.md`](planning/PLAN.md).

## Status

- ✅ **Market data** — GBM simulator + Massive API client, shared price cache, SSE streaming. See [`planning/MARKET_DATA_SUMMARY.md`](planning/MARKET_DATA_SUMMARY.md).
- 🚧 **Everything else** — backend API (auth, portfolio, chat), frontend, Docker/deployment — not yet built.

## Architecture

Single Docker container serving everything on port 8000:

- **Frontend**: Next.js (static export) with TypeScript and Tailwind CSS
- **Backend**: FastAPI (Python/`uv`) with SSE streaming
- **Database**: SQLite with lazy initialization
- **AI**: LiteLLM → OpenRouter (Cerebras inference) with structured outputs
- **Market data**: Built-in GBM simulator (default) or Massive API (optional)

## Project Structure

```
finally/
├── backend/     # FastAPI uv project (market data subsystem complete)
├── planning/    # Project documentation and agent contracts
├── frontend/    # Next.js static export (not yet built)
├── test/        # Playwright E2E tests (not yet built)
├── db/          # SQLite volume mount (runtime)
└── scripts/     # Start/stop helpers (not yet built)
```

## Backend Quick Start

```bash
cd backend
uv sync --extra dev
uv run pytest              # run tests
uv run market_data_demo.py # live terminal dashboard with simulated prices
```

See [`backend/README.md`](backend/README.md) and [`backend/CLAUDE.md`](backend/CLAUDE.md) for details.

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `OPENROUTER_API_KEY` | Yes | OpenRouter API key for AI chat |
| `MASSIVE_API_KEY` | No | Massive (Polygon.io) key for real market data; omit to use simulator |
| `LLM_MOCK` | No | Set `true` for deterministic mock LLM responses (testing) |
| `ADMIN_EMAIL` / `ADMIN_PASSWORD` | Yes (production) | Bootstraps the single admin account on first boot |
| `SESSION_SECRET` | Yes (production) | Signs session cookies |

## License

See [`LICENSE`](LICENSE).
