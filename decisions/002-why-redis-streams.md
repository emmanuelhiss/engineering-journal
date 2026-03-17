# ADR-002: Why Redis Streams over Celery or RabbitMQ

## Status
Accepted

## Context
FTIP agents need to receive events when new data arrives. The ingestion service fetches data from FRED and Alpha Vantage, and agents need to react to that data. We need a message bus that decouples producers (ingestion) from consumers (agents).

## Options Considered

### Celery + Redis/RabbitMQ
- Designed for task queues — "do this job" semantics
- Workers pull tasks and execute them
- Great for background jobs, but agents aren't "jobs" — they're continuous processes that react to data
- Adds Celery as a dependency with its own configuration complexity
- Task queue = one consumer per message. We need multiple agents reading the same data.

### RabbitMQ
- Full-featured message broker with routing, exchanges, queues
- Supports pub/sub patterns well
- Adds another infrastructure component to manage
- Overkill for our use case — we don't need complex routing

### Redis Streams (chosen)
- Built into Redis, which we already run for caching
- Persistent — messages survive restarts (unlike Redis pub/sub)
- Multiple consumers can read the same stream independently
- Consumer groups let multiple instances of the same agent share work
- XADD with maxlen keeps streams from growing unbounded
- Simple API — just XADD to publish, XREAD to consume

## Decision
Redis Streams. We already run Redis. Streams give us persistent, multi-consumer event streaming without adding another service. The semantics match our use case: "data arrived, any interested agent can read it."

## Consequences
- No additional infrastructure — Redis handles both caching and messaging
- Agents use consumer groups for scalability
- Messages persist until trimmed (maxlen=10000)
- If we outgrow Redis Streams (millions of events/sec), we'd migrate to Kafka — but that's far beyond our current scale
