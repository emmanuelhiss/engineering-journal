# ADR-005: Why Async Across the Entire Stack

## Status
Accepted

## Context
FTIP is I/O-heavy: it fetches data from external APIs (FRED, Alpha Vantage), reads/writes to PostgreSQL, publishes to Redis Streams, and serves HTTP responses. Most time is spent waiting for network responses, not computing. The question was whether to use sync code with threading or go fully async.

## Options Considered

### Synchronous with threading
- Familiar to most Python developers
- Each blocking call ties up a thread — thread pools have limits
- Thread safety requires locks, which adds complexity
- Mixed sync/async code creates "function coloring" problems — sync functions can't call async ones without wrappers

### Async in API only, sync elsewhere
- API endpoints are async (FastAPI encourages this)
- Ingestion and agents use sync code with `requests`, `psycopg2`, etc.
- Two different patterns in the same codebase — developers context-switch between styles
- Can't share database or Redis connection code between services without adapters

### Async everywhere (chosen)
- One concurrency model across all services: `asyncio`
- `httpx` for external API calls (async HTTP client)
- `asyncpg` + SQLAlchemy 2.0 async for database access
- `redis.asyncio` for Redis Streams and caching
- FastAPI runs on Uvicorn (ASGI) — async from request to response
- Single event loop handles hundreds of concurrent I/O operations without threads

## Decision
Async everywhere. Every service — API, ingestion, agents — uses `async def` functions and async libraries. This keeps one consistent model across the codebase and maximizes I/O concurrency without thread overhead.

## Consequences
- All database queries use `async with session` and `await`
- External API calls use `httpx.AsyncClient` with connection pooling
- Developers must understand `asyncio` — no hiding behind sync wrappers
- CPU-bound work (if any emerges) would need `run_in_executor` to avoid blocking the event loop
- Testing uses `pytest-asyncio` for async test functions
