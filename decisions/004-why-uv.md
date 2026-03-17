# ADR-004: Why uv over pip or Poetry

## Status
Accepted

## Context
FTIP needs a Python package manager for dependency resolution, virtual environment management, and reproducible installs across development and Docker builds. Speed matters because CI pipelines and Docker builds run dependency installs frequently.

## Options Considered

### pip + requirements.txt
- Universal — works everywhere Python runs
- No lock file by default — `pip freeze` captures the world, not intent
- Dependency resolution is slow and sometimes produces conflicts
- Virtual environment management is manual (`python -m venv`, `source activate`)
- No distinction between direct and transitive dependencies

### Poetry
- Proper lock file with dependency resolution
- Manages virtual environments automatically
- Slow — dependency resolution can take minutes on large trees
- `pyproject.toml` support is good but Poetry adds its own sections
- Docker builds suffer from slow installs — no parallel downloads

### uv (chosen)
- Written in Rust — installs dependencies in seconds, not minutes
- Drop-in replacement for pip commands (`uv pip install`, `uv pip compile`)
- Generates lock files with `uv pip compile`
- Creates and manages virtual environments with `uv venv`
- Docker builds go from ~60s to ~5s for dependency installation
- Supports `pyproject.toml` natively

## Decision
uv. It's dramatically faster than pip or Poetry, supports lock files, and the CLI is familiar to anyone who knows pip. The speed difference is most visible in Docker builds and CI — what took a minute now takes seconds.

## Consequences
- All services use `uv pip compile` to generate lock files from `pyproject.toml`
- Docker builds use `uv pip install` — multi-stage builds are faster
- Developers need uv installed locally (`curl -LsSf https://astral.sh/uv/install.sh | sh`)
- If uv introduces breaking changes (it's pre-1.0), we can fall back to pip since the lock file format is pip-compatible
