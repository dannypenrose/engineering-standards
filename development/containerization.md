# Containerization Standards

> Authoritative containerization standards for building, securing, and deploying container images across all projects.

## Purpose

Establish consistent patterns for container image creation, security hardening, registry management, and orchestration that ensure reliable, secure, and efficient deployments regardless of the underlying application stack.

## Core Principles

1. **Minimal attack surface** — Use the smallest base image that meets your needs
2. **Reproducible builds** — Pin versions, use lockfiles, deterministic layer ordering
3. **Security by default** — Non-root users, read-only filesystems, no secrets in images
4. **Layer efficiency** — Minimize image size through multi-stage builds and careful layering
5. **Scan everything** — Automated vulnerability scanning in CI before any image reaches a registry
6. **Immutable artefacts** — Once built and tagged, an image is never modified

## Base Image Selection

### Decision Framework

| Requirement | Recommended Base | Approx. Size |
| ----------- | ---------------- | ------------ |
| Smallest possible, static binary | `scratch` or `distroless` | 2-20 MB |
| Node.js production | `node:<version>-alpine` | ~50 MB |
| .NET production | `mcr.microsoft.com/dotnet/aspnet:<version>-alpine` | ~100 MB |
| Python production | `python:<version>-slim` | ~120 MB |
| General-purpose debugging | `alpine:<version>` | ~7 MB |
| Full toolchain (build stage only) | `node:<version>`, `dotnet/sdk:<version>` | 300+ MB |

### Rules

- **Always pin the major.minor version** (e.g., `node:22-alpine`, not `node:latest` or `node:alpine`)
- **Never use `latest` tag** — it introduces non-determinism
- **Prefer Alpine or slim variants** for production images
- **Use distroless for compiled languages** (Go, Rust) where no shell is needed
- **Update base images monthly** to pick up security patches

## Multi-Stage Builds

Every production Dockerfile should use multi-stage builds to separate build dependencies from runtime.

### Node.js / TypeScript

```dockerfile
# Stage 1: Install dependencies
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile

# Stage 2: Build
FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm build
RUN pnpm prune --prod

# Stage 3: Production
FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/package.json ./

USER appuser
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget -qO- http://forge-hub-dev.axiomstudio.io/health || exit 1

ENTRYPOINT ["node", "dist/main.js"]
```

### .NET

```dockerfile
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src
COPY *.sln .
COPY src/*/*.csproj ./
RUN for f in *.csproj; do mkdir -p "src/$(basename $f .csproj)" && mv "$f" "src/$(basename $f .csproj)/"; done
RUN dotnet restore

COPY . .
RUN dotnet publish src/MyApp.Api/MyApp.Api.csproj \
    -c Release -o /app/publish \
    --no-restore \
    /p:UseAppHost=false

# Stage 2: Production
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS runner
WORKDIR /app

RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

COPY --from=build --chown=appuser:appgroup /app/publish .

USER appuser
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:8080/health || exit 1

ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

### Python

```dockerfile
# Stage 1: Build
FROM python:3.12-slim AS builder
WORKDIR /app

RUN pip install --no-cache-dir poetry
COPY pyproject.toml poetry.lock ./
RUN poetry export -f requirements.txt --output requirements.txt --without-hashes
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

COPY . .

# Stage 2: Production
FROM python:3.12-slim AS runner
WORKDIR /app

RUN groupadd -g 1001 appgroup && \
    useradd -u 1001 -g appgroup -s /bin/false appuser

COPY --from=builder /install /usr/local
COPY --from=builder --chown=appuser:appgroup /app .

USER appuser
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

ENTRYPOINT ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

## Container Security

### Non-Root Execution

Every production container must run as a non-root user:

```dockerfile
# Create a dedicated user and group
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

# Switch before CMD/ENTRYPOINT
USER appuser
```

### Read-Only Filesystem

Where possible, run containers with a read-only root filesystem:

```yaml
# docker-compose.yml
services:
  api:
    image: myapp-api:1.0.0
    read_only: true
    tmpfs:
      - /tmp
      - /app/logs
    security_opt:
      - no-new-privileges:true
```

### Secrets Management

```dockerfile
# NEVER do this
ENV DATABASE_URL=postgresql://user:password@host/db
COPY .env /app/.env

# CORRECT: Use runtime injection
# Secrets are passed via environment variables at container start
# or mounted as files from a secret manager
```

**Rules:**
- Never bake secrets into images (ENV, COPY .env, build args)
- Use Docker secrets, Kubernetes secrets, or external secret managers
- Use `.dockerignore` to exclude sensitive files from build context

### .dockerignore

Every project with a Dockerfile must have a `.dockerignore`:

```
.git
.env
.env.*
node_modules
dist
*.md
.vscode
.idea
docker-compose*.yml
Dockerfile*
*.log
coverage
.next
__pycache__
*.pyc
bin/
obj/
```

## Resource Limits

Always define resource constraints for containers:

```yaml
# docker-compose.yml
services:
  api:
    image: myapp-api:1.0.0
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 128M
    restart: unless-stopped
```

### Recommended Defaults

| Service Type | CPU Limit | Memory Limit | Memory Reservation |
| ------------ | --------- | ------------ | ------------------ |
| API server | 1.0 | 512 MB | 128 MB |
| Worker/queue | 0.5 | 256 MB | 64 MB |
| Database | 2.0 | 1 GB | 256 MB |
| Cache (Redis) | 0.5 | 256 MB | 64 MB |

## Health Checks

### Dockerfile Health Checks

```dockerfile
# HTTP health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget -qO- http://forge-hub-dev.axiomstudio.io/health || exit 1

# TCP health check (for databases, caches)
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
  CMD nc -z localhost 5432 || exit 1
```

### Docker Compose Health Checks

```yaml
services:
  api:
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://forge-hub-dev.axiomstudio.io/health"]
      interval: 30s
      timeout: 3s
      start_period: 10s
      retries: 3

  db:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 3s
      retries: 5

  redis:
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
```

## Image Tagging Strategy

### Tag Format

```
<registry>/<namespace>/<image>:<version>

Examples:
  ghcr.io/myorg/api:1.2.3
  ghcr.io/myorg/api:1.2.3-abc1234
  ghcr.io/myorg/api:sha-abc1234
```

### Rules

| Tag Pattern | When to Use | Mutable? |
| ----------- | ----------- | -------- |
| `<semver>` (e.g., `1.2.3`) | Release builds | No |
| `<semver>-<short-sha>` | Release + traceability | No |
| `sha-<short-sha>` | Every CI build | No |
| `latest` | Development convenience only | Yes |
| `main` | Latest from main branch | Yes |

- **Production deployments must use immutable tags** (semver or sha-based)
- **Never deploy `latest` or branch tags to production**
- **Retain images for 90 days minimum** for rollback capability

## Vulnerability Scanning

### CI Integration

```yaml
# GitHub Actions example
- name: Build image
  run: docker build -t myapp:${{ github.sha }} .

- name: Scan for vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    format: table
    exit-code: 1
    severity: CRITICAL,HIGH
    ignore-unfixed: true
```

### Severity Policy

| Severity | Action | Timeline |
| -------- | ------ | -------- |
| Critical | Block deployment, patch immediately | Same day |
| High | Block deployment, schedule patch | Within 7 days |
| Medium | Warning, schedule patch | Within 30 days |
| Low | Informational | Next update cycle |

## Dockerfile Linting

Use hadolint to enforce Dockerfile best practices:

```bash
# Install
brew install hadolint

# Run
hadolint Dockerfile

# CI integration
hadolint --failure-threshold warning Dockerfile
```

### Key Rules

| Rule | Description |
| ---- | ----------- |
| DL3006 | Always tag the version of an image explicitly |
| DL3007 | Using `latest` is prone to errors |
| DL3008 | Pin versions in `apt-get install` |
| DL3009 | Delete apt-get lists after installing |
| DL3018 | Pin versions in `apk add` |
| DL3025 | Use JSON notation for `CMD`/`ENTRYPOINT` |
| DL4006 | Set `SHELL` option `-o pipefail` before `RUN` with pipe |

## Docker Compose Standards

### Development Compose

```yaml
# docker-compose.yml (development)
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: myapp_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dev"]
      interval: 10s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

volumes:
  postgres_data:
```

### Production Compose

```yaml
# docker-compose.prod.yml
services:
  api:
    image: ghcr.io/myorg/api:${APP_VERSION}
    read_only: true
    tmpfs:
      - /tmp
    security_opt:
      - no-new-privileges:true
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
    environment:
      - NODE_ENV=production
    env_file:
      - .env
    ports:
      - "3000:3000"
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://forge-hub-dev.axiomstudio.io/health"]
      interval: 30s
      timeout: 3s
      start_period: 10s
      retries: 3
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

## Layer Optimisation

### Ordering Layers by Change Frequency

```dockerfile
# Least frequently changed → Most frequently changed

# 1. System packages (rarely change)
RUN apk add --no-cache curl

# 2. Dependencies (change when lockfile changes)
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

# 3. Application code (changes every build)
COPY . .
RUN pnpm build
```

### Reducing Layer Size

```dockerfile
# Combine RUN commands to reduce layers
RUN apk add --no-cache \
      curl \
      ca-certificates && \
    rm -rf /var/cache/apk/*

# Use --no-cache-dir for pip
RUN pip install --no-cache-dir -r requirements.txt

# Clean up in the same layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

## Checklist

### Dockerfile

- [ ] Multi-stage build separating build and runtime
- [ ] Base image version pinned (not `latest`)
- [ ] Non-root user configured
- [ ] `HEALTHCHECK` instruction defined
- [ ] `.dockerignore` excludes sensitive files and unnecessary content
- [ ] No secrets in image (ENV, COPY, build args)
- [ ] Layers ordered by change frequency
- [ ] JSON notation for `ENTRYPOINT`/`CMD`
- [ ] hadolint passes with no warnings

### Security

- [ ] Vulnerability scan passes (no critical/high)
- [ ] Read-only filesystem where possible
- [ ] `no-new-privileges` security option set
- [ ] Resource limits defined
- [ ] Logging configured with size limits

### Registry

- [ ] Immutable tags for production deployments
- [ ] Image retention policy configured
- [ ] Automated scanning enabled on push
- [ ] Access controls configured

## References

- [Docker Best Practices](https://docs.docker.com/build/building/best-practices/)
- [OCI Image Specification](https://github.com/opencontainers/image-spec)
- [Hadolint](https://github.com/hadolint/hadolint)
- [Trivy Container Scanner](https://aquasecurity.github.io/trivy/)
- [Distroless Container Images](https://github.com/GoogleContainerTools/distroless)
