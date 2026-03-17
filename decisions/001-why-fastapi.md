# ADR-001: Why FastAPI over Django or Flask

## Status
Accepted

## Context
FTIP needs a Python REST API that serves trading data to future frontends and agents. The API must handle concurrent requests efficiently since multiple agents will query it simultaneously.

## Options Considered

### Django REST Framework
- Full-featured, batteries-included
- Synchronous by default — would need Django Channels or ASGI adapter for async
- ORM is tightly coupled to Django — we want SQLAlchemy for async support
- Overkill for an API-only service with no admin panel or templating

### Flask
- Lightweight and flexible
- No native async support (needs Quart or similar)
- No built-in request validation or API documentation
- Would need Flask-SQLAlchemy, Flask-Marshmallow, etc. for basic features

### FastAPI (chosen)
- Async-native from the ground up — built on Starlette ASGI
- Pydantic integration for request/response validation with type hints
- Auto-generates OpenAPI documentation at /docs
- Works naturally with SQLAlchemy 2.0 async and asyncpg
- Lightweight but not bare — right level of abstraction for an API service

## Decision
FastAPI. It's async-native, has built-in validation via Pydantic, and generates API docs automatically. Every dependency in our stack (asyncpg, httpx, redis.asyncio) is async — FastAPI is the natural fit.

## Consequences
- All endpoint handlers use `async def`
- Pydantic models serve as both validation AND documentation
- OpenAPI docs available at `/docs` with zero extra work
- Team members unfamiliar with FastAPI have a small learning curve, but the documentation is excellent
