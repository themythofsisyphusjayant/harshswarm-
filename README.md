# NYX Phoenix Migration

[![Python 3.12](https://img.shields.io/badge/Python-3.12-blue)](backend/requirements.txt)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.109-009688)](backend/requirements.txt)
[![Next.js 14](https://img.shields.io/badge/Next.js-14-black)](frontend/package.json)
[![React 18](https://img.shields.io/badge/React-18-61dafb)](frontend/package.json)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.3-3178c6)](frontend/package.json)

Migration of the NYX Decision Intelligence Simulator from a single Streamlit
application to a decoupled architecture: a Next.js 14 frontend, a FastAPI
backend, and the original simulation kernel preserved unchanged.

> **What this repo is.** The simulator itself lives in
> [NYX-CRYSTAL-AI](https://github.com/themythofsisyphusjayant/NYX-CRYSTAL-AI)
> (live app:
> [nyx-crystal-ai.lovable.app](https://nyx-crystal-ai.lovable.app/)).
> This repository is the migration of that simulator off Streamlit onto a
> service architecture — same kernel, same results, deployable parts.

## Problem

The original simulator was a monolithic Streamlit app. The simulation logic,
the user interface, and the LLM provider handling were coupled in one process,
which made the system hard to deploy, hard to test, and hard to extend. The
goal of this migration was to separate those concerns without changing the
simulation itself: same kernel, same results, deployable parts.

## Approach

- The original `nyx_kernel.py` was copied into `backend/app/core/` without
  modification. It is byte-identical to the legacy copy at the repository
  root (verified with `diff`: the two files are identical).
- The backend is FastAPI (`backend/app/main.py`, 546 lines, API version
  2.0.0). It exposes the simulation, analysis, and LLM fallback functions
  as HTTP endpoints, plus a WebSocket for live runs.
- The frontend is Next.js 14. It is a dashboard over the API; it contains no
  simulation logic of its own. Data fetching goes through React Query hooks
  in `frontend/src/lib/api.ts`.
- LLM provider fallback was kept and hardened with a circuit breaker, instead
  of being rewritten. The debate feature was preserved as an async service
  rather than dropped.
- Retrieval (RAG) and caching/queue adapters were added as new services
  around the preserved core, not inside it.

## Architecture

```
                        ┌─────────────────────────┐
                        │  Next.js 14 frontend    │
                        │  Vercel · React Query   │
                        │  page.tsx · dashboard/  │
                        │  TimeScrubber           │
                        └────────────┬────────────┘
                                     │ REST + WebSocket
                        ┌────────────▼────────────┐
                        │  FastAPI backend        │
                        │  app/main.py (v2.0.0)   │
                        │  14 HTTP + 1 WebSocket  │
                        └────────────┬────────────┘
              ┌──────────────────────┼──────────────────────┐
              ▼                      ▼                      ▼
   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
   │ nyx_kernel.py    │  │ KeyRotator       │  │ DatabaseRetriever│
   │ (preserved,      │  │ 8 LLM providers  │  │ SQL + vector RAG │
   │ byte-identical)  │  │ circuit breaker  │  │ Qdrant · Pinecone│
   └──────────────────┘  └──────────────────┘  └──────────────────┘
```

The four tiers, as stated in `backend/app/main.py`:

| Tier | Component | Notes |
|---|---|---|
| 1 | Next.js 14 frontend (Vercel) | Dashboard only; no simulation logic |
| 2 | FastAPI backend (Render/Fly.io) | Async endpoints, Pydantic validation |
| 3 | Core logic (`nyx_kernel.py`) | Preserved intact, byte-identical |
| 4 | Database layer | SQLite/PostgreSQL + vector DB ready |

Full diagrams, data-flow sequences, component responsibilities, deployment
topology, and security boundaries are in [`ARCHITECTURE.md`](ARCHITECTURE.md);
the migration discovery notes are in
[`PHOENIX_DISCOVERY_REPORT.md`](PHOENIX_DISCOVERY_REPORT.md).

## Layout

```
backend/                     FastAPI backend (Python 3.12+)
  app/main.py                Application entry point; routes, health, WebSocket
  app/core/nyx_kernel.py     Simulation kernel, copied unmodified (727 lines)
  app/services/fallback.py   KeyRotator: multi-provider LLM fallback (411 lines)
  app/services/debate.py     Debate orchestration, preserved from Streamlit
  app/services/retriever.py  RAG retriever: SQL + vector search (354 lines)
  app/services/adapters.py   SemanticCache (Redis) + Celery task queue (328 lines)
  app/db/                    Database models (placeholder)
  requirements.txt           fastapi 0.109, uvicorn 0.27, httpx 0.26,
                             sqlalchemy 2.0.25, pydantic 2.5.3, websockets 12
frontend/                    Next.js 14 app
  src/app/page.tsx           Main dashboard
  src/app/dashboard/         Dashboard route (WebSocket + TimeScrubber)
  src/components/TimeScrubber.tsx   Scrub through simulation rounds
  src/lib/api.ts             React Query hooks for the API (285 lines)
  package.json, next.config.js, tailwind.config.js
streamlit_app.py             Legacy Streamlit app (kept for reference)
nyx_kernel.py                Legacy kernel copy (kept for reference)
ARCHITECTURE.md              Architecture diagrams and data flow
PHOENIX_DISCOVERY_REPORT.md  Discovery report for the migration
vercel.json                  Frontend rewrites to the backend
render.yaml                  Backend service definition (Render, Python 3.12)
.env.example                 Environment variable template
```

## The swarm: agents and roles

Two kinds of agent operate in this system. They are different things and
should not be confused.

### 1. Simulation agents (the kernel)

The kernel runs a small society of `CognitiveAgent` instances, each carrying
ten psychological variables (self-worth, anxiety, consistency, momentum,
reputation, opportunity access, fragility index, lock-in, learning rate,
energy). Each round runs perception → cognition → intent emission → world
resolution; agents influence each other, converge, and occasionally cascade
(emotional crash). The kernel provides:

| Function | What it does |
|---|---|
| `run_simulation()` | The round loop; returns full `state_history` + `outcome_vector` |
| `detect_black_swan()` | Fragility analysis — the most breakable assumption |
| `run_counterfactual()` | What-if scenarios ("what if anxiety was 20% higher?") |
| `run_multi_trial()` | Statistical aggregation over repeated runs |
| `game_theory_insights()` | Nash equilibrium analysis |

Nothing in the kernel was changed in this migration. The legacy copy at the
repository root is retained so the preservation claim can be checked
directly (`diff nyx_kernel.py backend/app/core/nyx_kernel.py` prints
nothing).

### 2. Debate agents (the LLM layer)

`backend/app/services/debate.py` preserves the Streamlit `DebateAgent`
class, wrapped for async FastAPI use:

| Class | Role |
|---|---|
| `DebateAgent` | One LLM-powered participant: a name, a stance, and a history of its arguments. Each round it speaks 1–3 sentences in character, replying to the previous message. |
| `DebateOrchestrator` | Runs the debate: takes a topic, a list of agents (`[{name, stance}, ...]`), and a round count (default 3), and returns the full log plus the winner. |

Debate agents argue through the LLM fallback (below), so a debate keeps
going even if one provider fails mid-round.

## KeyRotator: LLM fallback with a circuit breaker

`backend/app/services/fallback.py` keeps the original
`generate_with_fallback()` behaviour and adds:

- **Circuit breaker**, so a failing provider is skipped instead of retried
  repeatedly.
- **Weighted round-robin**: provider weights adapt to observed success
  rates, so reliable providers get more traffic.
- **Rate-limit tracking**: 429 responses trigger a cooldown for that
  provider.
- **Hot reload** of API keys from the environment without a restart
  (`POST /api/config/reload`).

Eight providers are configured; only the ones with a key set in the
environment are used:

| Provider | Model (as configured) |
|---|---|
| Groq | llama-3.3-70b-versatile |
| SambaNova | Meta-Llama-3.3-70B-Instruct |
| Cerebras | llama-3.3-70b |
| Google | gemini-2.5-flash |
| Mistral | mistral-small-4 |
| Cohere | command-a-03-2025 |
| OpenRouter | openrouter/free |
| HuggingFace | meta-llama/Llama-3.3-70B-Instruct |

Key health (success rate, circuit state, cooldown) is visible at
`GET /api/keys/status`, and a tripped breaker can be reset per provider at
`POST /api/keys/reset/{name}`.

## Retriever: RAG over SQL and vector stores

`backend/app/services/retriever.py` is new in this migration. It powers the
"Oracle" functionality — retrieving context for generation:

| Capability | Detail |
|---|---|
| SQL retrieval | SQLAlchemy adapters (PostgreSQL, SQLite, MySQL) |
| Vector retrieval | Qdrant, Pinecone, Chroma adapters |
| Hybrid search | SQL + vector combined, deduplicated, ranked by score (default `top_k` 5) |
| Embeddings | Generated per query for vector search |
| Source selection | `source` = `"sql"`, `"vector"`, or omitted for hybrid |

`backend/app/services/adapters.py` adds two optional layers around it:
a `SemanticCache` (Redis-backed, SHA-256 cache keys, LRU) and a Celery
task queue for background jobs (including large simulations). Both are
optional imports — if `redis-py` or `celery` is not installed, that layer
is disabled with a warning rather than crashing the service.

## API

Base URL `http://localhost:8000` in development; interactive docs at `/docs`.
Verified against `backend/app/main.py` on 21 Nov 2026: **14 HTTP endpoints
and one WebSocket**. (An earlier count of 13 missed `/api/debate`.)

| Endpoint | Method | Description |
|---|---|---|
| `/health` | GET | Health check (`{"status": "healthy", "version": "2.0.0"}`) |
| `/health/ready` | GET | Readiness check with dependency verification; 503 if a critical service is down |
| `/api/simulate` | POST | Run a cognitive-social simulation |
| `/api/simulate/{id}` | GET | Retrieve a simulation by ID (in-memory store) |
| `/api/debate` | POST | Run a debate round — topic, agents `[{name, stance}]`, rounds (default 3); returns log + winner |
| `/api/keys/status` | GET | API key health dashboard data |
| `/api/keys/reset/{name}` | POST | Reset a provider circuit breaker |
| `/api/generate` | POST | Generate text with LLM fallback |
| `/api/retrieve` | POST | Retrieve context from a database (RAG) |
| `/api/analyze/black-swan` | POST | Detect fragile assumptions |
| `/api/analyze/counterfactual` | POST | Run a what-if scenario |
| `/api/analyze/multi-trial` | POST | Run multiple trials |
| `/api/analyze/game-theory` | POST | Compute Nash equilibria |
| `/api/config/reload` | POST | Hot-reload API keys from the environment |
| `/ws/simulation/{id}` | WebSocket | Live simulation streaming, with ping/pong keepalive |

Example — health check and a debate:

```bash
curl http://localhost:8000/health

curl -X POST http://localhost:8000/api/debate \
  -H "Content-Type: application/json" \
  -d '{"topic": "Should we ban smartphones in schools?",
       "agents": [{"name": "Maya", "stance": "For the ban"},
                  {"name": "Dev", "stance": "Against the ban"}],
       "rounds": 3}'
```

### Frontend hooks

`frontend/src/lib/api.ts` wraps the API in React Query hooks, so the
dashboard never calls `fetch` directly:

| Hook | Endpoint used | Notes |
|---|---|---|
| `useHealthCheck()` | `GET /health` | Refetches every 30 s |
| `useKeyStatus()` | `GET /api/keys/status` | Refetches every 10 s |
| `useRunSimulation()` | `POST /api/simulate` | Mutation |
| `useSimulation(id)` | `GET /api/simulate/{id}` | Query by ID |
| `useBlackSwanAnalysis()` | `POST /api/analyze/black-swan` | Mutation |
| `useCounterfactual()` | `POST /api/analyze/counterfactual` | Mutation |
| `useMultiTrial()` | `POST /api/analyze/multi-trial` | Mutation |
| `useGenerate()` | `POST /api/generate` | Mutation |
| `useResetProvider()` | `POST /api/keys/reset/{name}` | Mutation |
| `useReloadConfig()` | `POST /api/config/reload` | Mutation |
| `useSimulationWebSocket(id)` | `/ws/simulation/{id}` | Live ticks on the dashboard route |

The dashboard (`src/app/dashboard/page.tsx`) pairs the WebSocket hook with
the `TimeScrubber` component to scrub through simulation rounds as they
stream in.

## Run it

Backend (Python 3.12+):

```bash
cd backend
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp ../.env.example .env   # add API keys (Groq, Gemini, etc.)
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

API docs at `http://localhost:8000/docs`. The backend runs without any LLM
keys set — simulation and analysis endpoints work on the kernel alone; only
`/api/generate` and `/api/debate` need a provider key.

Frontend:

```bash
cd frontend
npm install
cp .env.example .env.local   # NEXT_PUBLIC_BACKEND_URL=http://localhost:8000
npm run dev
```

The frontend runs on `http://localhost:3000`. Other scripts:
`npm run build`, `npm run start`, `npm run lint`.

## Deployment

- **Frontend: Vercel.** `vercel.json` defines rewrites: `/api/:path*` and
  `/ws/:path*` proxy to the backend host. Set `NEXT_PUBLIC_BACKEND_URL`
  to the backend URL (the example production value in
  `frontend/.env.example` is the Render host).
- **Backend: Render.** `render.yaml` defines the web service (Python 3.12,
  Oregon, starter plan): build with
  `pip install -r backend/requirements.txt`, start with
  `uvicorn app.main:app --host 0.0.0.0 --port $PORT`, health check on
  `/health`, 1 GB persistent disk for simulation logs and the SQLite
  database. API keys are injected via the Render dashboard, not committed.

**No live deployment has been verified for this repository.** The configs
above describe where each part deploys, not a running instance.

## Security

- API keys live in environment variables only and are never committed
  (`.env.example` lists the eight provider keys plus database and vector
  DB settings).
- Inputs are validated with Pydantic models.
- Invalid configuration returns 503 rather than crashing the service.
- Known hardening item: CORS in `backend/app/main.py` currently allows
  `"*"` (all origins), with a code comment to remove the wildcard in
  production. Restrict `ALLOWED_ORIGINS` to the real frontend domain
  before deploying publicly.
- Simulation results are held in an in-memory dict (`simulation_store`);
  the code notes this is to be replaced with Redis in production, so
  results do not survive a restart and are not shared across workers.

## Status

The migration is complete: kernel preserved byte-identical, backend and
frontend built, deployment configs in place. Work in progress:

- No unit tests for the backend services yet (pytest is listed as
  optional in `requirements.txt`, not wired up).
- No end-to-end tests for critical flows.
- No load testing or monitoring setup.
- The in-memory simulation store needs Redis before multi-worker
  deployment; CORS needs tightening (see above).

## License

No license file is present in this repository.
