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

*(Add more as they come up during build)*

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
| 1 — Data Foundation | 🟡 Building | FRED + AV ingestion, FastAPI, Redis Streams |
| 2 — Ship | ⬜ | Terraform, GitHub Actions, AWS deployment |
| 3 — First Agent | ⬜ | Currency Scoring Agent + Claude API |
| 4 — Weekly Agent Drops | ⬜ | New agent each week |
| 5 — Observability | ⬜ | Prometheus, Grafana |
| 6 — Portfolio + Cert | ⬜ | Polish, blog posts, AWS cert |

## Links
- **Repo:** github.com/emmanuelhiss/ftip
- **Decisions:** [Architecture Decision Records](../decisions/)
