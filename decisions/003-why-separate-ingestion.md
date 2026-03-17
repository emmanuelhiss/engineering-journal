# ADR-003: Why Ingestion Is a Separate Service from API

## Status
Accepted

## Context
FTIP pulls macro data from external APIs (FRED, Alpha Vantage) and serves that data through a REST API. Initially, ingestion logic lived inside the API service. As the system grew, this created problems: long-running data fetches blocked API responsiveness, and deployment of ingestion changes required redeploying the entire API.

## Options Considered

### Ingestion inside the API service
- Simpler deployment — one service to manage
- Data fetching runs on the same event loop as request handling
- A slow or failing external API (FRED rate limits, Alpha Vantage timeouts) degrades API response times
- Can't scale ingestion independently — scaling the API also scales ingestion, wasting resources

### Cron job / scheduled script
- Simple to implement — just a script on a timer
- No connection to the rest of the system — can't publish events when data arrives
- Error handling and retry logic would need to be built from scratch
- No visibility into what's running or what failed

### Separate ingestion service (chosen)
- Runs on its own schedule, independent of API request load
- Publishes events to Redis Streams when new data arrives — agents react without polling
- Can be scaled, restarted, or redeployed without touching the API
- Failures in ingestion don't cascade to API availability

## Decision
Ingestion is its own service. It fetches data on a schedule, writes to the database, and publishes events to Redis Streams. The API service only reads from the database and serves responses. Neither service knows about the other's internals.

## Consequences
- Two services to deploy and monitor instead of one
- Clear ownership: ingestion owns data freshness, API owns data delivery
- Redis Streams acts as the integration point between ingestion and agents
- Adding a new data source means changing ingestion only — API and agents pick it up automatically via events
