---
description: "Use when working with containers. Covers multi-stage builds, non-root users, layer caching, secrets hygiene, HEALTHCHECK, signal handling, and docker-compose conventions."
applyTo: "**/{Dockerfile,Containerfile,docker-compose*,docker-compose.*,compose*,compose.*,.dockerignore}"
---

# Container Conventions

## Multi-Stage Builds — Mandatory

Every container image MUST use multi-stage builds. Build dependencies stay in the builder stage; the final image contains only runtime artifacts.

```dockerfile
# Builder stage — compiles, bundles, generates
FROM golang:1.22-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /app ./cmd/server

# Runtime stage — minimal, no build tools
FROM alpine:3.20
RUN apk add --no-cache ca-certificates tzdata
COPY --from=builder /app /app
USER appuser
ENTRYPOINT ["/app"]
```

- **Builder stage** has compilers, package managers, dev headers
- **Runtime stage** has only the binary and runtime deps (ca-certificates, tzdata)
- Use `--from=builder` to copy artifacts across stages
- `docker build --target builder` for debugging without bloating the final image

## Non-Root User — Mandatory

Never run containers as root in production. Create a dedicated user.

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

- Use `-S` (system user, no login shell) — not a human account
- The `USER` directive must come AFTER any commands that need root (package installs, file copies to system dirs)
- Test with `docker run --rm --user nobody <image>` — if it works, you're root-free
- Kubernetes: set `securityContext.runAsNonRoot: true` and `runAsUser: 1000`

## Layer Ordering for Cache Hits

Order `COPY` and `RUN` commands from least-frequently-changing to most-frequently-changing.

```dockerfile
# 1. Dependencies first (changes rarely)
COPY package.json package-lock.json ./
RUN npm ci --production

# 2. Source code last (changes every commit)
COPY . .
```

- Package manager files (`package.json`, `go.mod`, `requirements.txt`, `Cargo.toml`) go BEFORE source
- Docker caches layers — if a layer changes, all subsequent layers rebuild
- Put expensive operations (compilation, downloads) early, trivial operations (file copies) late
- Combine shell commands with `&&` to avoid unnecessary layers:

```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends ca-certificates curl && \
    rm -rf /var/lib/apt/lists/*
```

## .dockerignore — Mandatory

Every project with containers MUST have a `.dockerignore`. It prevents leaking secrets, bloating context, and invalidating cache.

```dockerignore
# Secrets — never in the build context
.env
.env.*
*.key
*.pem
secrets/

# Version control
.git
.gitignore
.gitattributes

# Dependencies (installed inside the build)
node_modules/
vendor/
__pycache__/

# Build artifacts
dist/
build/
target/

# Docs & config (non-runtime)
*.md
.dockerignore
docker-compose*.yml
.editorconfig
```

- `.dockerignore` is a denylist, not an allowlist. Be explicit about what to exclude.
- Secrets in the build context end up in image layers — even if you `rm` them later.
- Test context size: `docker build --no-cache . 2>&1 | grep "sending build context"`

## Pin Base Image Digests

Never use floating tags in production. Pin to a specific digest for reproducibility and security.

```dockerfile
# Bad — moves under your feet
FROM node:20-alpine

# Good — pinned to exact digest
FROM node:20-alpine@sha256:a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2
```

- Digest pinning guarantees the same bytes every build
- Use `docker pull node:20-alpine && docker inspect node:20-alpine | jq -r '.[0].RepoDigests[0]'` to find the digest
- Update digests periodically (Renovate/Dependabot can do this)

## HEALTHCHECK — Mandatory

Every long-running container MUST have a `HEALTHCHECK`. It tells the orchestrator whether the container is actually working, not just running.

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 --start-period=10s \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1
```

- `--interval`: how often to check (default 30s)
- `--timeout`: how long to wait for a response (default 5s)
- `--retries`: consecutive failures before marking unhealthy (default 3)
- `--start-period`: grace period after container starts (default 0s)
- The health check endpoint should be lightweight — no database queries, no external calls
- For non-HTTP services, use `pg_isready` (Postgres), `redis-cli ping` (Redis), `nc -z localhost PORT` (TCP)

## Signal Handling

Containers receive `SIGTERM` on stop. Your process must handle it gracefully.

- **Use `exec` form, not shell form:**

```dockerfile
# Bad — /bin/sh -c "node server.js" is PID 1 and doesn't forward signals
CMD node server.js

# Good — node is PID 1 and receives signals directly
CMD ["node", "server.js"]
```

- **PID 1 problem**: the Linux kernel treats PID 1 specially — it won't receive signals unless you register handlers. Use `tini` or `dumb-init` if your app doesn't handle signals:

```dockerfile
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]
```

- **Graceful shutdown**: catch `SIGTERM`, stop accepting new requests, finish in-flight requests, close connections, exit cleanly
- Kubernetes: set `terminationGracePeriodSeconds` to longer than your app's shutdown time

## No Secrets in Image Layers

Secrets baked into layers are permanent. Anyone with image access can extract them.

```dockerfile
# Bad — secret is in layer history forever
ENV DATABASE_URL=postgres://user:password@host/db

# Good — passed at runtime
# docker run -e DATABASE_URL=... <image>
# Or via Kubernetes secrets / external secrets manager
```

- Never `COPY` or `ENV` secrets into the image
- Use Docker BuildKit secrets for build-time secrets (`--secret` flag, `RUN --mount=type=secret`)
- For runtime secrets: environment variables, mounted secret files, or external secrets manager
- Scan images for secrets: `docker scan`, `trufflehog`, `git-secrets` on the repo (not just the image)

## Image Size Hygiene

Small images pull faster, start faster, and have a smaller attack surface.

- **Use slim/alpine base images**: `python:3.12-slim` not `python:3.12` (700MB saved)
- **Clean package manager caches** in the same `RUN` layer:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends pkg && \
    rm -rf /var/lib/apt/lists/*
```

- **`--no-install-recommends`**: avoids pulling suggested-but-unnecessary packages
- **Strip debug symbols**: `-ldflags="-s -w"` for Go, `strip` for C/C++ binaries
- **`.dockerignore` aggressively**: `node_modules/`, `target/`, `.git/` — rebuild inside the container
- Check image size: `docker image ls <image>` and `docker history <image>` to find layers to slim

## docker-compose Conventions

- **Use profiles for optional services**: `profiles: ["debug"]` for tools like Adminer, phpMyAdmin — not started by default
- **Named volumes for persistent data**: `postgres_data:/var/lib/postgresql/data` — not bind mounts for databases
- **Health checks in compose**: `depends_on` with `condition: service_healthy` (requires `HEALTHCHECK` in the Dockerfile)
- **Environment file separation**: `.env` for Compose variable substitution, not app secrets. App secrets go in `.env.secrets` (gitignored) or external secrets manager.
- **Never expose ports to `0.0.0.0` unless necessary**: `127.0.0.1:8080:8080` for dev-only services
- **Resource limits on all services**:

```yaml
services:
  api:
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
```
