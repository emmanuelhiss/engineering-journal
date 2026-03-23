# FTIP — Fundamental Trading Intelligence Platform

## One-Liner
> Multi-agent AI trading research desk that ingests real macro-economic data from FRED and Alpha Vantage, exposes it through a FastAPI REST API, and uses Redis Streams as a message bus for AI agents that score currency strength.

## Why I Built This
I actively trade forex. I was manually checking 32+ economic indicators across 8 currencies on different websites — central bank rates, inflation, GDP, unemployment. That's slow and error-prone. FTIP automates the data collection and will eventually use AI agents to score currency strength and flag opportunities.

This isn't a tutorial project. The requirements come from real trading experience.

## Evolution from NemonixCentral
NemonixCentral was my first trading tool — US-only data, monolithic architecture, AWS Bedrock for AI. FTIP is the redesign: 8 currencies instead of 1, microservices instead of monolith, event-driven agents instead of request-response AI, cloud-native deployment from day one.

## Architecture
```
FRED API ──→ Ingestion Service ──→ PostgreSQL
Alpha Vantage ──→ Ingestion Service ──→ PostgreSQL
                  Ingestion Service ──→ Redis Streams
                                         ↓
                                      AI Agents (consume streams)
                                         ↓
                                      Agent Outputs → PostgreSQL
                                         ↓
FastAPI REST API ←──────────────── PostgreSQL
```

## Hard Problems I Solved

### 1. WSL2 + Docker PostgreSQL Auth
**Symptom:** Alembic migrations failed when run from WSL2 terminal
**Root cause:** WSL2 port forwarding + PostgreSQL scram-sha-256 auth don't play well together
**Fix:** Run migrations from inside the Docker container where internal DNS resolves correctly
**Learned:** Docker's internal networking is separate from the host — services should always connect through Docker DNS names, not localhost

```
WSL2 Terminal ──→ localhost:5432 ──→ ??? ──→ PostgreSQL
                  (port forward)     (auth fails)

Docker Container ──→ postgres:5432 ──→ PostgreSQL
                     (internal DNS)    (auth works)
```

### 2. uv Not Found in Runtime Stage
**Symptom:** API container started but crashed immediately with `exec: "uv": executable file not found in $PATH`
**Root cause:** Multi-stage Dockerfile copies uv binary into the builder stage via `COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/`, but the runtime stage starts from a clean `python:3.12-slim` image — no uv binary exists there. The CMD `uv run uvicorn ...` fails because `uv` doesn't exist.
**Fix:** Add `COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/` to the runtime stage of both Dockerfiles. Multi-stage builds start completely fresh — nothing carries over unless explicitly COPY'd.
**Learned:** Each `FROM` in a Dockerfile is a clean slate. If you need a tool in the final image, you must explicitly copy it — even if the builder stage had it.

```
# Builder stage                    # Runtime stage
FROM python:3.12-slim AS builder   FROM python:3.12-slim AS runtime
COPY --from=...uv /uv /bin/       # ← uv is NOT here!
                                   # CMD ["uv", "run", ...] → crash

# Fix: add this to runtime stage
                                   COPY --from=...uv /uv /uvx /bin/  ✓
```

### 3. Permission Denied + Dev Deps at Runtime
**Symptom:** API container entered a crash loop. Logs showed `Permission denied (os error 13)` while trying to install mypy.
**Root cause:** Two stacked issues: (1) CMD used bare `uv run` which triggers `uv sync` at runtime, attempting to install dev dependencies (mypy, ruff, pytest) into the production container. (2) The `.venv` directory was created by root in the builder stage, but the runtime stage runs as `appuser` (uid 1000) who can't write to root-owned directories.
**Fix:** Two changes — add `RUN chown -R appuser:appuser /app` before `USER appuser` so the non-root user owns the application directory. Change CMD from `uv run` to `uv run --no-sync` so uv doesn't attempt to install/sync anything at container startup.
**Learned:** Non-root containers need explicit ownership of copied directories. Files COPY'd from builder stages inherit root ownership. Also, `uv run` without `--no-sync` will sync dependencies on every container start — fine for development, bad for production.

```
Builder (root)                    Runtime (appuser)
┌──────────────────┐             ┌──────────────────┐
│ .venv/ (root:root)│  ──COPY──→ │ .venv/ (root:root)│ ← appuser can't write!
│ app/   (root:root)│            │ app/   (root:root)│
└──────────────────┘             └──────────────────┘

Fix: RUN chown -R appuser:appuser /app  (before USER appuser)
     CMD ["uv", "run", "--no-sync", ...]  (skip dependency sync)
```

### 4. Numeric Overflow on GDP Values
**Symptom:** Ingestion service crashed during FRED persist with `NumericValueOutOfRangeError: numeric field overflow`. Zero economic data was stored — the entire batch failed because of one bad value.
**Root cause:** The `economic_indicators.value` column was defined as `NUMERIC(12, 4)`, which can store at most 99,999,999.9999 (8 digits before the decimal). But GDP values from FRED are raw numbers — Canada's GDP came back as `587,354,750,000.0` (587 billion). That's 12 digits, overflowing the column.
**Fix:** Created an Alembic migration to widen the column from `NUMERIC(12, 4)` to `NUMERIC(20, 4)`, which handles values up to 10^16. Updated the column type in both services' model definitions.
**Learned:** Always check real data ranges before picking column precision. Interest rates are 0-20. CPI is 100-350. But GDP of a single country can be in the trillions. The schema was designed with percentages in mind — nobody checked what raw GDP values look like.

```
NUMERIC(12, 4)  →  max 99,999,999.9999     ← Interest rates: ✓
                                               CPI values:    ✓
                                               GDP (Canada):  ✗  587,354,750,000

NUMERIC(20, 4)  →  max 9,999,999,999,999,999.9999  ← All values: ✓
```

### 5. Six Invalid FRED Series IDs
**Symptom:** 6 of 32 FRED API calls returned HTTP 400 Bad Request. This meant AUD and NZD were missing interest rate data (no cb_state entries for RBA and RBNZ), and several currencies had gaps in GDP and unemployment data.
**Root cause:** The series IDs were sourced from FRED documentation and research but never validated against the actual API. FRED series get deprecated, renamed, or were never accessible via the public API. The IDs looked correct but didn't exist.

| Failed ID | What It Was | Replacement | Source |
|-----------|-------------|-------------|--------|
| `RBATCTR` | RBA Cash Rate | `IRSTCI01AUM156N` | OECD Short-term Rate |
| `NZLOCRS` | RBNZ OCR | `IRSTCI01NZM156N` | OECD Short-term Rate |
| `AUSGDPGDPVOVCPGPQ` | Australia GDP | `NGDPRSAXDCAUQ` | OECD National Accounts |
| `NZLRGDPEXP` | NZ GDP | `NZLGDPNQDSMEI` | OECD MEI |
| `LRHUTTTTNZM156S` | NZ Unemployment | `LRUNTTTTNZQ156S` | OECD Labor Stats |
| `LRHUTTTTCHM156S` | CHF Unemployment | `LMUNRRTTCHM156N` | OECD Registered Unemployment |

**Fix:** Used FRED's search API (`/fred/series/search`) to find valid series for each indicator, then verified each one returns data with a direct API call before hardcoding.
**Learned:** Always validate external API data sources with real calls before committing. Documentation and "should work" aren't the same as "does work." Especially with FRED — series IDs follow loose naming conventions and there's no guarantee a plausible-looking ID exists.

## Tech Choices
See the [Architecture Decisions](../decisions/) folder for detailed reasoning on each choice.

| Choice | Why | What I Considered Instead |
|---|---|---|
| FastAPI | Async-native, Pydantic validation, auto docs | Django (too heavy), Flask (no async) |
| SQLAlchemy 2.0 async | Type-safe ORM with async, Alembic migrations | Raw asyncpg (no ORM), Django ORM (tied to Django) |
| Redis Streams | Persistent event streaming, already running Redis | Celery (task queue, wrong semantics), RabbitMQ (extra service) |
| uv | 10-100x faster than pip, proper lockfiles | pip (slow), poetry (slower, complex) |
| Docker multi-stage | Small images, cached deps, non-root user | Single-stage (bloated, insecure) |

## Phase Progress

| Phase | Status | What It Delivers |
|---|---|---|
| 1 — Data Foundation | ✅ Complete | FRED + AV ingestion, FastAPI, Redis Streams |
| 2 — Ship | ⬜ | Terraform, GitHub Actions, AWS deployment |
| 3 — First Agent | ⬜ | Currency Scoring Agent + Claude API |
| 4 — Weekly Agent Drops | ⬜ | New agent each week |
| 5 — Observability | ⬜ | Prometheus, Grafana |
| 6 — Portfolio + Cert | ⬜ | Polish, blog posts, AWS cert |

## Links
- **Repo:** github.com/emmanuelhiss/ftip
- **Decisions:** [Architecture Decision Records](../decisions/)
