# NYX Phoenix Migration

Migration of the NYX Decision Intelligence Simulator from a single Streamlit
application to a decoupled architecture: a Next.js frontend, a FastAPI
backend, and the original simulation kernel preserved unchanged.

## Problem

The original simulator was a monolithic Streamlit app. The simulation logic,
the user interface, and the LLM provider handling were coupled in one process,
which made the system hard to deploy, hard to test, and hard to extend. The
goal of this migration was to separate those concerns without changing the
simulation itself: same kernel, same results, deployable parts.

## Approach

- The original `nyx_kernel.py` was copied into `backend/app/core/` without
  modification. It is byte-identical to the legacy copy at the repository
  root (same blob in the repository tree).
- The backend is FastAPI. It exposes the simulation, analysis, and LLM
  fallback functions as HTTP endpoints, plus a WebSocket for live runs.
- The frontend is Next.js 14. It is a dashboard over the API; it contains no
  simulation logic of its own.
- LLM provider fallback was kept and hardened with a circuit breaker, instead
  of being rewritten.

## Layout

```
backend/                  FastAPI backend
  app/main.py             Application entry point; routes, health, WebSocket
  app/core/nyx_kernel.py  Simulation kernel, copied unmodified
  app/services/fallback.py   KeyRotator: multi-provider LLM fallback
  app/services/debate.py     Debate orchestration (preserved)
  app/services/retriever.py  RAG retriever: SQL + vector search
  app/services/adapters.py   Provider adapters
  app/db/                 Database models (placeholder)
  requirements.txt
frontend/                 Next.js 14 app
  src/app/page.tsx        Main dashboard
  src/app/dashboard/      Dashboard route
  src/components/TimeScrubber.tsx
  src/lib/api.ts          React Query hooks for the API
  package.json, next.config.js, tailwind.config.js
streamlit_app.py          Legacy Streamlit app (kept for reference)
nyx_kernel.py             Legacy kernel copy (kept for reference)
ARCHITECTURE.md           Architecture diagrams and data flow
PHOENIX_DISCOVERY_REPORT.md  Discovery report for the migration
vercel.json               Frontend rewrites
render.yaml               Backend service definition
.env.example              Environment variable template
```

## The kernel

The kernel is deterministic: a seeded PRNG (`SeededRandom`) drives a
population of `CognitiveAgent` instances, each carrying ten psychological
variables. It provides:

- `run_simulation()` — the round loop
- `detect_black_swan()` — fragility analysis
- `run_counterfactual()` — what-if scenarios
- `run_multi_trial()` — statistical aggregation over repeated runs
- `game_theory_insights()` — Nash equilibrium analysis

Nothing in the kernel was changed in this migration. The legacy copy at the
repository root is retained so the preservation claim can be checked directly.

## KeyRotator

`backend/app/services/fallback.py` keeps the original
`generate_with_fallback()` behaviour and adds:

- A circuit breaker, so a failing provider is skipped instead of retried
  repeatedly.
- Weighted round-robin: provider weights adapt to observed success rates.
- Rate-limit tracking: 429 responses trigger a cooldown for that provider.
- Hot reload of API keys from the environment without a restart.

## Retriever

`backend/app/services/retriever.py` is new in this migration. It provides
retrieval for RAG: SQLAlchemy adapters for SQL databases (PostgreSQL,
SQLite), vector database adapters (Qdrant, Pinecone), hybrid search across
both, and async I/O.

## API

Base URL `http://localhost:8000` in development; interactive docs at `/docs`.
Verified against `backend/app/main.py` on 21 Nov 2026: 14 HTTP endpoints and
one WebSocket.

| Endpoint | Method | Description |
|---|---|---|
| `/health` | GET | Health check |
| `/health/ready` | GET | Readiness check with dependency verification |
| `/api/simulate` | POST | Run a cognitive-social simulation |
| `/api/simulate/{id}` | GET | Retrieve a simulation by ID |
| `/api/debate` | POST | Run a debate round |
| `/api/keys/status` | GET | API key health dashboard data |
| `/api/keys/reset/{name}` | POST | Reset a provider circuit breaker |
| `/api/generate` | POST | Generate text with LLM fallback |
| `/api/retrieve` | POST | Retrieve context from a database (RAG) |
| `/api/analyze/black-swan` | POST | Detect fragile assumptions |
| `/api/analyze/counterfactual` | POST | Run a what-if scenario |
| `/api/analyze/multi-trial` | POST | Run multiple trials |
| `/api/analyze/game-theory` | POST | Compute Nash equilibria |
| `/api/config/reload` | POST | Hot-reload API keys from the environment |
| `/ws/simulation/{id}` | WebSocket | Live simulation streaming |

## Run it

Backend:

```bash
cd backend
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp ../.env.example .env   # add API keys
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

Frontend:

```bash
cd frontend
npm install
cp .env.example .env.local
npm run dev
```

The frontend runs on `http://localhost:3000`.

## Deployment

- Frontend: Vercel. `vercel.json` defines rewrites to the backend; set
  `NEXT_PUBLIC_BACKEND_URL` to the backend URL.
- Backend: Render. `render.yaml` defines the web service; build with
  `pip install -r backend/requirements.txt`, start with
  `uvicorn app.main:app --host 0.0.0.0 --port $PORT`.

No live deployment is configured in this repository.

## Security

- API keys live in environment variables only and are never committed.
- CORS is configured for specific origins in production.
- Inputs are validated with Pydantic models.
- Invalid configuration returns 503 rather than crashing the service.

## Status

The migration is complete: kernel preserved, backend and frontend built,
deployment configs in place. Work in progress:

- No unit tests for the backend services yet.
- No end-to-end tests for critical flows.
- No load testing or monitoring setup.

## License

No license file is present in this repository.
