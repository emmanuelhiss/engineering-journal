# ADR-006: Why Multi-Stage Docker Builds

## Status
Accepted

## Context
FTIP and NexOps services are containerized. Initial Dockerfiles used a single stage — install everything in one image. This produced images over 1GB that included build tools, compilers, and cached package files that aren't needed at runtime.

## Options Considered

### Single-stage build
- Simple — one `FROM`, install everything, done
- Image includes gcc, python headers, pip cache, and other build-only dependencies
- 1GB+ images are slow to push, pull, and deploy
- Larger attack surface — build tools in production containers

### Docker layer caching only
- Reorder commands to maximize cache hits
- Helps build speed but doesn't reduce final image size
- Still ships build tools in production

### Multi-stage builds (chosen)
- Stage 1 (builder): install build tools, compile dependencies, create virtual environment
- Stage 2 (runtime): copy only the virtual environment and application code from builder
- Final image contains only Python runtime + application — no compilers, no pip cache
- Images drop from ~1.1GB to ~250MB

## Decision
Multi-stage builds for all services. The builder stage handles compilation and dependency installation. The runtime stage copies the finished virtual environment and runs the app. Nothing else.

## Consequences
- Final images are ~250MB instead of ~1.1GB
- Faster CI/CD — smaller images push and pull in seconds
- Smaller attack surface — no gcc, no make, no pip in production
- Dockerfiles are slightly more complex (two `FROM` statements) but the pattern is consistent across all services
- Build cache still works — Docker caches the builder stage between builds if dependencies haven't changed
