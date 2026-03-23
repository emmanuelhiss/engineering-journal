# ADR-007: Why SQLAlchemy Core in Ingestion vs Full ORM in API

## Status
Accepted

## Context
FTIP has two services that talk to the same PostgreSQL database: the API reads data and serves it through endpoints, the ingestion service writes data from external APIs. Both use SQLAlchemy, but they use it differently — the API uses the full ORM (mapped classes, relationships, `Mapped[]` type hints), while ingestion uses SQLAlchemy Core (raw `Table` objects and `insert()` statements). This was a deliberate choice.

## Options Considered

### Full ORM in both services
- Consistent pattern across the codebase — one way to do things
- ORM features like `.query()`, lazy loading, and relationship navigation make reads expressive
- But ingestion never reads — it only writes. ORM overhead (identity map, change tracking, object instantiation) adds no value for bulk inserts
- Both services would need identical model classes — tight coupling through shared code, or duplicated 100+ line model files that must stay in sync

### SQLAlchemy Core in both services
- Lightweight everywhere — no ORM overhead
- But the API benefits from ORM: `session.query(EconomicIndicator).filter_by(currency="AUD")` is more readable than Core's `select(economic_indicators).where(economic_indicators.c.currency == "AUD")`
- Pydantic's `from_attributes = True` works beautifully with ORM objects — with Core you'd need manual dict mapping

### Core for ingestion, ORM for API (chosen)
- Ingestion uses `Table` objects with `insert()` and `on_conflict_do_update()` — perfect for write-only workloads
- API uses ORM classes with `Mapped[]` types — Pydantic serializes them directly via `from_attributes`
- Each service defines only what it needs: ingestion's models are 60 lines of column definitions, API's models are full mapped classes with type hints
- No shared code between services — each owns its own model layer

## Decision
SQLAlchemy Core for ingestion, full ORM for API. The ingestion service only does bulk writes — it doesn't need object identity, change tracking, or relationship navigation. The API does reads with filtering and joins — the ORM makes those expressive and integrates cleanly with Pydantic serialization.

## Consequences
- Column definitions exist in two places (ingestion `Table` objects, API ORM classes) — they must match the same database schema
- Alembic lives in the API service since it owns the ORM models and `Base.metadata`
- Ingestion's `Table` objects don't need to exactly mirror every ORM feature — they just need matching column names and types for the SQL they generate
- If a third service needs database access, it chooses Core or ORM based on whether it reads or writes

```
Ingestion (Core)                    API (ORM)
┌─────────────────────┐     ┌──────────────────────────┐
│ Table("forex_prices" │     │ class ForexPrice(Base):   │
│   Column("pair",...) │     │   pair: Mapped[str]       │
│   Column("price",..) │     │   price: Mapped[Decimal]  │
│ )                    │     │                          │
│                      │     │ → Pydantic serialization  │
│ insert().values(...)  │     │ → session.query(...)      │
│ on_conflict(...)     │     │ → relationship loading    │
└─────────┬───────────┘     └──────────┬───────────────┘
          │                            │
          └──────────┬─────────────────┘
                     │
              ┌──────┴──────┐
              │  PostgreSQL  │
              │  (same DB)   │
              └─────────────┘
```
