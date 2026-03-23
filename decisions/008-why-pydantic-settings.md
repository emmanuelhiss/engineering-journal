# ADR-008: Why Pydantic Settings over os.getenv

## Status
Accepted

## Context
FTIP services need configuration: database URLs, Redis URLs, API keys, debug flags. This configuration comes from environment variables (and `.env` files in development). The question was how to read and validate these values.

## Options Considered

### Raw os.getenv
- Built-in, zero dependencies
- No validation — `os.getenv("DATABASE_URL")` returns `str | None`, so every access needs a None check or a default
- No type conversion — port numbers come back as strings, booleans as `"true"` strings
- Scattered throughout the codebase — config values read wherever they're needed, no single source of truth
- Typos in env var names are silent bugs — `os.getenv("DATABSE_URL")` returns `None` without error
- No `.env` file support without adding `python-dotenv`

### python-decouple
- Reads from `.env` files
- Type casting built in: `config("DEBUG", cast=bool)`
- But each value is still read individually — no config object, no centralized validation
- Missing required values fail at first use, not at startup

### Pydantic Settings (chosen)
- Extends Pydantic `BaseModel` — all fields are typed and validated
- Missing required fields raise `ValidationError` at import time, not at first use
- Reads from environment variables AND `.env` files automatically
- Single `Settings` class is the source of truth for all config
- `@lru_cache` on the factory function creates a singleton — one parse, used everywhere
- Type coercion built in: `debug: bool = False` converts `"true"`, `"1"`, `"yes"` automatically
- Testable via dependency injection — pass a test `Settings` object with overrides

## Decision
Pydantic Settings. It validates all configuration at startup, fails fast if required values are missing, and provides a typed, cached singleton that any module can import. The `.env` file is read automatically with zero extra code.

## Consequences
- Each service has a `config.py` with a `Settings` class — every config value is defined there
- App crashes immediately on startup if `DATABASE_URL` or `FRED_API_KEY` is missing — no "works fine until the first database call" surprises
- IDE autocomplete works on settings: `settings.database_url` instead of `os.getenv("DATABASE_URL")`
- Adding a new config value means adding one line to the `Settings` class — it's automatically read from the environment
- `extra = "ignore"` in model config means services don't crash when other services' env vars are present (e.g., ingestion doesn't care about API-only settings)

```python
# What we have (Pydantic Settings):
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")
    database_url: str              # Required — crashes at startup if missing
    redis_url: str = "redis://redis:6379"  # Optional with default
    fred_api_key: str              # Required
    debug: bool = False            # Auto-converts "true" → True

# What we avoided (os.getenv scattered everywhere):
DATABASE_URL = os.getenv("DATABASE_URL")  # Returns None silently
if DATABASE_URL is None:
    raise ValueError("DATABASE_URL not set")  # Manual check in every file
```
