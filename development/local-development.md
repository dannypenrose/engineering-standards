# Local Development Standards

> Authoritative local development standards for consistent, reproducible developer workflows.

## Purpose

Establish consistent, reproducible local development environments that enable developers to be productive from day one while maintaining parity with production systems.

## Core Principles

1. **Environment parity** - Local should mirror production as closely as possible
2. **Single-command setup** - New developers productive within minutes
3. **Isolated dependencies** - Projects don't interfere with each other
4. **Reproducible builds** - Same inputs produce same outputs
5. **Fast feedback loops** - Changes reflected immediately
6. **Offline-capable** - Core development works without internet

## Development Environment Setup

### Prerequisites

| Tool | Purpose | Installation |
|------|---------|--------------|
| Node.js 18+ | JavaScript runtime | `nvm install 18` |
| PNPM 8+ | Package manager | `npm install -g pnpm` |
| Docker | Containerization | Docker Desktop |
| Git | Version control | `brew install git` |
| Database client | Local databases | MySQL Workbench, pgAdmin |

### Project Setup Flow

```bash
# 1. Clone repository
git clone <repository-url>
cd <project>

# 2. Install dependencies
pnpm install

# 3. Configure environment
# Create .env.local with local credentials (see project README)
# Backend: .env.local (local dev secrets)
# Frontend: .env.local (local dev overrides)

# 4. Initialize database
pnpm db:setup  # or: pnpm prisma db push

# 5. Start development server
pnpm dev
```

### Environment Variables

#### File Convention

##### Backend
| File | Purpose | Git Status |
|------|---------|------------|
| `.env.local` | Local development secrets and config | **Gitignored** |
| `.env` | Production secrets (copied to server `/etc` dir) | **Gitignored** |

##### Frontend (Next.js)
| File | Purpose | Git Status |
|------|---------|------------|
| `.env.local` | Local development overrides | **Gitignored** |
| `.env.production` | Public `NEXT_PUBLIC_*` vars (loaded at build time) | **Committed** |
| `.env` | Production secrets (copied to server `/etc` dir) | **Gitignored** |

> **Warning**: `.env.local` overrides `.env.production` during local development. Always verify `.env.production` values independently.

#### Required Variables Documentation

```bash
# Backend .env.local - document all required variables
# ============================================
# Database
# ============================================
DATABASE_URL="mysql://user:password@localhost:3306/dbname"

# ============================================
# Authentication
# ============================================
JWT_SECRET="generate-with-openssl-rand-base64-32"

# ============================================
# External Services (optional for development)
# ============================================
# STRIPE_SECRET_KEY="sk_test_..."
# OPENROUTER_API_KEY="sk-or-..."
```

## Port Management

### Port Assignment Strategy

Assign ports by technology stack to avoid conflicts and provide predictable assignments:

| Range | Technology | Use Case |
|-------|------------|----------|
| **3000-3999** | Node.js / Next.js | Frontend and backend JavaScript/TypeScript services |
| **4000-4999** | Python / Ruby | Flask, FastAPI, Django, Rails services |
| **5000-5999** | .NET / ASP.NET Core | C# backend services and APIs |
| **7000-7999** | Go / Rust | High-performance compiled services |
| **8000-8999** | Alternate Web/Frontend | Secondary web servers, admin panels |

> **Full port assignments**: Document port allocations in each project's CLAUDE.md or README.

### Port Documentation Requirements

Every project must document its port assignments:

```markdown
## Development Server

| Service | Port | URL |
|---------|------|-----|
| Frontend | 3000 | http://localhost:3000 |
| Backend API | 3002 | http://localhost:3002/api/v1 |
| API Docs | 3002 | http://localhost:3002/api/docs |
| Prisma Studio | 5555 | http://localhost:5555 |
```

### Conflict Resolution

```bash
# Check if port is in use
lsof -i :3000

# Kill process on port
kill -9 $(lsof -t -i :3000)

# Or use alternative port
PORT=3001 pnpm dev
```

## Database Management

### Local Database Options

| Option | Use Case | Pros | Cons |
|--------|----------|------|------|
| Docker container | Team consistency | Reproducible, isolated | Requires Docker |
| Local install | Simple setup | Fast, native | Version conflicts |
| Cloud dev DB | Shared data | Real data, backups | Network dependency |

### Docker Database Setup

```yaml
# docker-compose.yml
services:
  mysql:
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: development
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
```

### Database Workflow

```bash
# Generate Prisma client
pnpm prisma generate

# Push schema changes (development)
pnpm prisma db push

# Create migration (before commit)
pnpm prisma migrate dev --name descriptive_name

# Reset database
pnpm prisma migrate reset

# Open database UI
pnpm prisma studio
```

## Development Servers

### Hot Reload Configuration

#### Frontend (Next.js)

```javascript
// next.config.js
module.exports = {
  reactStrictMode: true,
  // Fast Refresh enabled by default
};
```

#### Backend (NestJS)

```json
// nest-cli.json
{
  "watchAssets": true,
  "compilerOptions": {
    "webpack": true,
    "deleteOutDir": true
  }
}
```

### Concurrent Development

```json
// package.json
{
  "scripts": {
    "dev": "turbo run dev",
    "dev:frontend": "pnpm --filter frontend dev",
    "dev:backend": "pnpm --filter backend dev"
  }
}
```

## Debugging

### VS Code Configuration

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Backend",
      "type": "node",
      "request": "attach",
      "port": 9229,
      "restart": true,
      "sourceMaps": true
    },
    {
      "name": "Debug Frontend",
      "type": "chrome",
      "request": "launch",
      "url": "http://localhost:3000",
      "webRoot": "${workspaceFolder}/apps/frontend"
    }
  ]
}
```

### Debug Mode Startup

```bash
# Backend with debugging
NODE_OPTIONS='--inspect' pnpm dev:backend

# Or in package.json
"dev:debug": "nest start --debug --watch"
```

## Performance Optimization

### Local Build Performance

```bash
# Enable Turborepo caching
TURBO_CACHE=1 pnpm build

# Skip type checking during development
SKIP_TYPE_CHECK=1 pnpm dev

# Use SWC for faster compilation
# (configured in next.config.js)
```

### Memory Management

```bash
# Increase Node.js memory for large projects
NODE_OPTIONS="--max-old-space-size=4096" pnpm dev

# Monitor memory usage
node --expose-gc --trace-gc app.js
```

## Common Issues

### Issue: Port Already in Use

```bash
# Find and kill process
lsof -ti :3000 | xargs kill -9
```

### Issue: Database Connection Failed

```bash
# Check if database is running
docker ps | grep mysql

# Check connection
mysql -u root -p -h localhost -P 3306
```

### Issue: Dependencies Out of Sync

```bash
# Clean install
rm -rf node_modules pnpm-lock.yaml
pnpm install
```

### Issue: Environment Variables Not Loading

```bash
# Check file exists and has correct name
ls -la .env*

# Verify no syntax errors
cat .env.local

# Restart development server
# (env vars loaded at startup)
```

## Checklist

### New Project Setup

- [ ] README includes setup instructions
- [ ] Environment files follow convention (`.env.local`, `.env.production`, `.env`)
- [ ] Port assignments documented
- [ ] Database setup automated
- [ ] Development scripts defined
- [ ] VS Code settings configured

### New Developer Onboarding

- [ ] Clone repository
- [ ] Install dependencies
- [ ] Configure environment variables
- [ ] Initialize database
- [ ] Start development server
- [ ] Run tests successfully
- [ ] Make a test change (verify hot reload)

### Before Committing

- [ ] Tests pass locally
- [ ] Linting passes
- [ ] Type checking passes
- [ ] Database migrations created (if schema changed)
- [ ] Environment variables documented (if added)

## References

- [12-Factor App - Dev/Prod Parity](https://12factor.net/dev-prod-parity)
- [Docker Development Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [VS Code Debugging](https://code.visualstudio.com/docs/editor/debugging)
- [Turborepo Caching](https://turbo.build/repo/docs/core-concepts/caching)
