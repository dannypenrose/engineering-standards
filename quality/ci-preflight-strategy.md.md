# CI/CD & Preflight Strategy

> Authoritative quality gate standards for pre-commit and pre-push validation across all projects.

## Core Philosophy

**Local-first quality gate system with deploy-only CI:**

1. **Preflight (Local Pre-Push)** - Comprehensive gate that catches all quality issues before code reaches GitHub
2. **Deploy (GitHub Actions)** - Build, bundle, and deploy only. No quality gates.

**Key principle:** The pre-push hook is the single source of truth for code quality. If code passes the local gate, it is trusted for deployment. Deploy workflows do not duplicate quality checks — they exist solely to build artifacts and ship them to the server. This eliminates wasted CI minutes and removes the false safety net of "CI will catch it."

---

## Division of Responsibility

```
┌─────────────────────────────────────────────────────────────────┐
│           LOCAL (Pre-Push)         │   DEPLOY (GitHub Actions)  │
├─────────────────────────────────────────────────────────────────┤
│ ✅ Type checking                   │ ✅ Build                   │
│ ✅ Linting                         │ ✅ Bundle preparation      │
│ ✅ Build verification              │ ✅ rsync to server         │
│ ✅ Standalone smoke test           │ ✅ Database migrations     │
│ ✅ Dependency verification         │ ✅ Service restart          │
│ ✅ Security scanning (quick)       │ ❌ NO quality gates        │
│ ❌ NO tests (too slow)             │ ❌ NO smoke tests          │
└─────────────────────────────────────────────────────────────────┘
```

> **Why no quality gates in CI?** The pre-push hook catches all quality issues before code ever reaches GitHub. Deploy workflows trust that the code has already been validated. This avoids paying for redundant CI minutes and keeps deploy pipelines fast and focused.

### Why This Split?

| Concern | Preflight (Local) | Deploy (GitHub Actions) |
|---------|-------------------|-------------------------|
| **Speed** | Must complete in <60 seconds | As fast as possible — build + ship |
| **Environment** | Developer's machine | Clean runner |
| **Purpose** | Catch all quality issues | Build and deploy only |
| **Tests** | Skip (too slow for pre-push) | Skip (not CI's job) |
| **Quality gates** | Type-check, lint, build, smoke test | None |
| **Turbo cache** | Local `.turbo/` cache | Restored via `actions/cache` for `.turbo/` |

---

## Preflight Check Matrix (All Tech Stacks)

This matrix shows which checks apply to each technology stack, enabling consistent quality gates regardless of project type.

### Core Checks

| Check | TypeScript/Node.js | .NET Core | Python | Purpose |
|-------|-------------------|-----------|--------|---------|
| **Type checking** | `tsc --noEmit` or `turbo type-check` | Built into `dotnet build` | `mypy` or `pyright` | Compile-time errors |
| **Linting** | `eslint` | `dotnet format --verify-no-changes` | `ruff` or `flake8` | Code style enforcement |
| **Build** | `turbo build` or `npm run build` | `dotnet build -c Release` | `python -m build` | Compilation verification |
| **Dependency lock** | `--frozen-lockfile` | `dotnet restore --locked-mode` | `pip install --require-hashes` | Reproducible builds |
| **Security scan** | `npm audit` / `pnpm audit` | `dotnet list package --vulnerable` | `pip-audit` / `safety` | Vulnerability detection |

### Platform-Specific Checks

| Check | TypeScript/Node.js | .NET Core | Purpose |
|-------|-------------------|-----------|---------|
| **Prisma postinstall** | ✅ Verify `postinstall` script | N/A | Vercel/serverless deployment |
| **Build-time client init** | ✅ Check for module-level SDK init | N/A | Serverless compatibility |
| **EF migrations** | N/A | ✅ Check for pending migrations | Database sync |
| **Environment validation** | ✅ Check `.env` files exist | ✅ Check `appsettings.json` | Configuration completeness |
| **Serverless compatibility** | ✅ No `fs` in API routes | ✅ No file system in Azure Functions | Edge/serverless runtime |

### ORM-Specific Checks

| Check | Prisma (Node.js) | Entity Framework (.NET) | Purpose |
|-------|-----------------|------------------------|---------|
| **Client generation** | `prisma generate` | EF generates at build | ORM client ready |
| **Migration status** | `prisma migrate status` | `dotnet ef migrations list` | Pending migrations |
| **Schema validation** | `prisma validate` | Build validates model | Schema correctness |

### Command Quick Reference

```bash
# TypeScript/Node.js Preflight
pnpm install --frozen-lockfile
pnpm type-check          # or: tsc --noEmit
pnpm lint                # or: eslint .
pnpm build               # or: turbo build
pnpm audit               # Security scan

# .NET Core Preflight
dotnet restore --locked-mode
dotnet build --configuration Release
dotnet format --verify-no-changes
dotnet list package --vulnerable --include-transitive

# Python Preflight
pip install -r requirements.txt --require-hashes
mypy src/
ruff check src/
python -m build
pip-audit
```

---

## Preflight (Local Pre-Push Hook)

### What It Should Check

1. **Type checking** - Catches compile-time errors
2. **Linting** - Enforces code style
3. **Build** - Verifies the app compiles
4. **Standalone smoke test** - Boots the standalone bundle to catch missing dependencies
5. **Dependencies** - Ensures lockfile is in sync and backend deps are declared
6. **Platform-specific** - Prisma setup, serverless compatibility
7. **Security scanning** - Quick vulnerability scan

### What It Should NOT Check

1. **Tests** - Too slow for pre-push
2. **Coverage** - Requires running tests
3. **Code duplication analysis** - Expensive, informational only
4. **Full security audits** - Time-consuming

### Implementation (Bash Script)

```bash
#!/bin/bash
set -e

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m'

echo "🚀 Running preflight checks..."

# 1. Dependencies
echo "📦 Verifying dependencies..."
pnpm install --frozen-lockfile

# 2. Type check
echo "🔍 Running type check..."
pnpm type-check

# 3. Lint
echo "🧹 Running lint..."
pnpm lint

# 4. Build (optional - skip with SKIP_BUILD=true)
if [ "$SKIP_BUILD" != "true" ]; then
    echo "🏗️ Running build..."
    pnpm build
else
    echo "⏭️ Skipping build (SKIP_BUILD=true)"
fi

echo -e "${GREEN}✅ Preflight passed${NC}"
```

### Git Hook Setup (Husky)

```bash
# .husky/pre-push
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

pnpm preflight
```

---

## Deploy Workflows (GitHub Actions)

### Key Principles

1. **Build and deploy only** - No quality gates, no tests, no linting
2. **Trust the pre-push hook** - Code that reaches GitHub has already been validated locally
3. **Turbo cache** - Restore `.turbo/` directory via `actions/cache` to speed up builds
4. **Parallel execution** - Build independent apps concurrently
5. **Fast deploys** - Install, build, bundle, rsync. Nothing else.

### Workflow Structure

```yaml
name: Deploy

on:
  push:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  PUPPETEER_SKIP_DOWNLOAD: 'true'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true

      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'

      # Restore Turbo cache for faster builds
      - name: Restore Turbo cache
        uses: actions/cache@v4
        with:
          path: .turbo
          key: turbo-${{ runner.os }}-${{ github.sha }}
          restore-keys: turbo-${{ runner.os }}-

      - run: pnpm install --frozen-lockfile
      - run: pnpm build

      # Prepare and upload artifacts per app...
      # (See Deployment Strategy section for full examples)

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      # Download artifacts, rsync to server, run migrations, restart services
```

> **No validate job, no test job, no lint job.** These are handled entirely by the pre-push hook. The deploy workflow's only job is to produce build artifacts and ship them.

---

## Monorepo: Affected-Only Pattern

For monorepos, the critical enhancement is **affected-only** checking in the pre-push hook — only run checks for code that actually changed.

### Why Affected-Only?

Without it:
- Push to `app-a` is blocked because `app-b` has failing type checks
- Developer frustration and wasted time
- Encourages people to bypass hooks

With it:
- Changes to `app-a` only run checks for `app-a`
- Other apps' issues do not block you
- Same pattern used by Google (Bazel), Meta (Buck), Vercel

### Turborepo Implementation (Pre-Push)

```bash
# In the pre-push hook / preflight script
# Turbo automatically caches — unchanged packages are instant

pnpm turbo run type-check
pnpm turbo run lint
pnpm turbo run build
```

Turbo's content-aware caching means unchanged packages hit the cache and complete in milliseconds. There is no need for manual `--filter` logic in most cases — Turbo handles this automatically.

### Deploy Workflows (No Affected Detection Needed)

Deploy workflows build all affected apps via Turbo's cache and deploy them. There is no need for change detection in CI because:

1. Turbo's cache makes unchanged builds near-instant
2. Deploy workflows do not run quality gates, so there is nothing to skip
3. The `.turbo/` cache is restored from `actions/cache` in CI

```yaml
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true

      - name: Restore Turbo cache
        uses: actions/cache@v4
        with:
          path: .turbo
          key: turbo-${{ runner.os }}-${{ github.sha }}
          restore-keys: turbo-${{ runner.os }}-

      - run: pnpm install --frozen-lockfile
      - run: pnpm build  # Turbo caches unchanged packages
```

---

## Non-Blocking Patterns

Since deploy workflows do not run quality gates, non-blocking patterns apply to the **pre-push hook** only.

### Security Scanning

Quick vulnerability scans run in the pre-push hook. These are warnings, not blockers:

```bash
# In preflight script — warn but don't fail
pnpm audit 2>/dev/null || echo "Security warnings detected (non-blocking)"
```

### Expensive Checks (Run Manually)

These checks are too slow for the pre-push hook and are not in the deploy workflow. Run them manually or on a schedule:

- **E2E tests** — Run locally before major releases: `pnpm test:e2e`
- **Code duplication analysis** — Run periodically: `pnpm check:duplication`
- **Full security audits** — Run before releases or on a weekly schedule
- **Coverage reports** — Run locally: `pnpm test --coverage`

---

## Environment Variables

Standard skip flags for the preflight script:

| Variable | Effect |
|----------|--------|
| `SKIP_BUILD=true` | Skip build step (faster iteration) |
| `SKIP_PREFLIGHT=true` | Skip the entire pre-push hook (emergency use only) |

---

## Quick Reference

### For Single-App Projects

```
Pre-push: type-check → lint → build → smoke test
Deploy:   install → build → bundle → rsync → restart
```

### For Monorepos

```
Pre-push: type-check → lint → build → smoke test → dep check
Deploy:   install → build (Turbo cached) → bundle → rsync → migrate → restart
```

### Decision Tree

```
Is it a quality check (types, lint, tests)?
├─ Yes → Pre-push hook only
└─ No  → Continue

Is it a build/deploy step?
├─ Yes → Deploy workflow only
└─ No  → Continue

Is it too slow for pre-push (<60 seconds)?
├─ Yes → Run manually or on a schedule
└─ No  → Add to pre-push hook

Should it block the push?
├─ Critical (types, lint, build, smoke test) → Blocking
└─ Informational (security scan) → Warning only
```

---

## Implementation Checklist

### Local Setup (Pre-Push Hook)

- [ ] Preflight script created (`scripts/preflight.sh`)
- [ ] Git hook installed (Husky `pre-push`)
- [ ] Type checking runs (`pnpm type-check`)
- [ ] Linting runs (`pnpm lint`)
- [ ] Build verification runs (`pnpm build`)
- [ ] Standalone smoke test boots the bundle
- [ ] Dependency verification runs (`pnpm check:deps`)
- [ ] Skip flags documented (`SKIP_BUILD`, `SKIP_PREFLIGHT`)
- [ ] Script completes in <60 seconds (with Turbo cache)

### Deploy Workflow Setup

- [ ] GitHub Actions workflow created (`.github/workflows/deploy.yml`)
- [ ] Concurrency configured (cancel in-progress)
- [ ] Turbo cache restored via `actions/cache` for `.turbo/`
- [ ] Build step only — no quality gates
- [ ] Artifact upload with `include-hidden-files: true`
- [ ] rsync deployment to server
- [ ] Database migrations run post-deploy
- [ ] Service restart via systemd

### Monorepo Additions

- [ ] Turbo configured for content-aware caching
- [ ] `.turbo/` cache shared between local and CI
- [ ] Cross-app blocking eliminated via Turbo cache

---

---

## .NET Backend Integration

### Comprehensive .NET Preflight Script

This script mirrors the depth of the Node.js preflight, including platform-specific checks for Azure/serverless deployments.

```bash
#!/bin/bash

# .NET Preflight Validation Script
#
# PURPOSE: Fast local gate for deployment-blocking issues
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
#
# DIVISION OF RESPONSIBILITY:
# ┌─────────────────────────────────────────────────────┐
# │ Preflight (local)      │ Deploy (GitHub Actions)   │
# ├─────────────────────────────────────────────────────┤
# │ ✓ Build verification   │ ✓ Build                   │
# │ ✓ Code formatting      │ ✓ Bundle preparation      │
# │ ✓ Security scanning    │ ✓ Deploy via rsync        │
# │ ✓ EF migration status  │ ✓ Database migrations     │
# │ ✓ Config validation    │ ✓ Service restart          │
# │ ✗ NO tests             │ ✗ NO quality gates        │
# └─────────────────────────────────────────────────────┘
#
# Usage:
#   ./preflight.sh              # Full preflight
#   ./preflight.sh --quick      # Skip build (faster)
#   SKIP_BUILD=true ./preflight.sh
#
# Environment:
#   CI=true           Enable CI mode (stricter checks)
#   SKIP_BUILD=true   Skip build step (faster iteration)
#   SKIP_FORMAT=true  Skip format verification

set -e

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

ERRORS=0
WARNINGS=0

print_step() { echo -e "\n${BLUE}━━━━ $1 ━━━━${NC}"; }
print_success() { echo -e "${GREEN}✓ $1${NC}"; }
print_error() { echo -e "${RED}✗ $1${NC}"; ((ERRORS++)); }
print_warn() { echo -e "${YELLOW}⚠ $1${NC}"; ((WARNINGS++)); }
print_info() { echo -e "  $1"; }

# Parse arguments
if [ "$1" = "--quick" ]; then
    SKIP_BUILD=true
fi

echo -e "${BLUE}🚀 .NET Preflight Checks${NC}"
echo ""

# ============================================
# 1. Validate solution structure
# ============================================
print_step "1/7 Validating solution structure"

# Find solution file
SLN_FILE=$(find . -maxdepth 2 -name "*.sln" | head -1)
if [ -z "$SLN_FILE" ]; then
    print_error "No .sln file found"
else
    print_success "Solution found: $SLN_FILE"
fi

# Check for required project files
if ls src/*/*.csproj 1> /dev/null 2>&1; then
    print_success "Project files found"
else
    print_warn "No projects found in src/ directory"
fi

# ============================================
# 2. Restore dependencies
# ============================================
print_step "2/7 Restoring dependencies"

if dotnet restore --locked-mode 2>/dev/null; then
    print_success "Dependencies restored (locked mode)"
elif dotnet restore; then
    print_warn "Dependencies restored (lockfile may be out of sync)"
    print_info "Run 'dotnet restore --use-lock-file' to update packages.lock.json"
else
    print_error "Dependency restore failed"
fi

# ============================================
# 3. Build verification
# ============================================
print_step "3/7 Running build verification"

if [ "$SKIP_BUILD" = "true" ]; then
    print_info "Skipping build (SKIP_BUILD=true)"
else
    if dotnet build --configuration Release --no-restore --verbosity minimal; then
        print_success "Build passed"
    else
        print_error "Build failed"
    fi
fi

# ============================================
# 4. Code formatting check
# ============================================
print_step "4/7 Checking code formatting"

if [ "$SKIP_FORMAT" = "true" ]; then
    print_info "Skipping format check (SKIP_FORMAT=true)"
else
    if dotnet format --verify-no-changes --verbosity minimal 2>/dev/null; then
        print_success "Code formatting valid"
    else
        print_warn "Code formatting issues detected"
        print_info "Run 'dotnet format' to fix formatting"
    fi
fi

# ============================================
# 5. Security vulnerability scan
# ============================================
print_step "5/7 Scanning for security vulnerabilities"

VULN_OUTPUT=$(dotnet list package --vulnerable --include-transitive 2>/dev/null || echo "")

if echo "$VULN_OUTPUT" | grep -q "has the following vulnerable packages"; then
    print_warn "Vulnerable packages detected"
    echo "$VULN_OUTPUT" | grep -A 100 "vulnerable packages" | head -20
else
    print_success "No known vulnerabilities"
fi

# ============================================
# 6. Entity Framework migration check
# ============================================
print_step "6/7 Checking Entity Framework migrations"

# Find projects with EF migrations
EF_PROJECTS=$(find . -path "*/Migrations/*.cs" -exec dirname {} \; 2>/dev/null | sort -u | head -5)

if [ -n "$EF_PROJECTS" ]; then
    for proj_dir in $EF_PROJECTS; do
        proj_name=$(basename "$(dirname "$proj_dir")")
        print_info "Found migrations in: $proj_name"
    done

    # Check for pending model changes (requires EF tools)
    if command -v dotnet-ef &> /dev/null; then
        # Find the startup project
        STARTUP_PROJECT=$(find . -name "*.Api.csproj" -o -name "*.Web.csproj" | head -1)
        if [ -n "$STARTUP_PROJECT" ]; then
            if dotnet ef migrations list --project "$STARTUP_PROJECT" --no-build 2>/dev/null | grep -q "(Pending)"; then
                print_warn "Pending migrations detected - ensure database is up to date"
            else
                print_success "No pending migrations"
            fi
        fi
    else
        print_info "EF tools not installed - skipping migration status check"
        print_info "Install with: dotnet tool install --global dotnet-ef"
    fi
else
    print_info "No Entity Framework migrations found"
fi

# ============================================
# 7. Configuration validation
# ============================================
print_step "7/7 Validating configuration"

CONFIG_OK=true

# Check for appsettings files
for proj_dir in src/*/; do
    if [ -f "${proj_dir}appsettings.json" ]; then
        # Validate JSON syntax
        if python3 -m json.tool "${proj_dir}appsettings.json" > /dev/null 2>&1 || \
           jq . "${proj_dir}appsettings.json" > /dev/null 2>&1; then
            print_success "Valid appsettings.json in $(basename "$proj_dir")"
        else
            print_error "Invalid JSON in ${proj_dir}appsettings.json"
            CONFIG_OK=false
        fi
    fi
done

# Check for sensitive data in config (basic check)
if grep -rE "(password|secret|apikey).*=.*['\"][^'\"]+['\"]" src/ --include="*.json" --include="*.cs" 2>/dev/null | grep -v "example\|placeholder\|changeme" | head -3; then
    print_warn "Potential hardcoded secrets detected - review before commit"
    CONFIG_OK=false
fi

if $CONFIG_OK; then
    print_success "Configuration validation passed"
fi

# ============================================
# Summary
# ============================================
echo ""
echo -e "${BLUE}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
echo -e "${BLUE}📊 PREFLIGHT SUMMARY${NC}"
echo -e "${BLUE}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"

if [ $ERRORS -eq 0 ]; then
    if [ $WARNINGS -gt 0 ]; then
        echo -e "${YELLOW}⚠ Passed with $WARNINGS warning(s)${NC}"
    else
        echo -e "${GREEN}✅ All checks passed${NC}"
    fi
    echo ""
    echo "✨ Ready for deployment"
    exit 0
else
    echo -e "${RED}❌ Failed with $ERRORS error(s) and $WARNINGS warning(s)${NC}"
    echo ""
    echo "🛑 Fix errors before deploying"
    exit 1
fi
```

### Deploy Workflow for .NET Projects

Since quality gates run in the pre-push hook, the deploy workflow only builds and ships:

```yaml
name: Deploy .NET

on:
  push:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-and-deploy:
    name: Build & Deploy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore --configuration Release

      - name: Publish
        run: dotnet publish --no-build --configuration Release -o ./publish

      - name: Deploy via rsync
        run: |
          rsync -rz --delete \
            -e "ssh -i ~/.ssh/deploy_key" \
            ./publish/ \
            ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_IP }}:/opt/my-app/

      - name: Run EF migrations
        run: |
          ssh -i ~/.ssh/deploy_key \
            ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_IP }} \
            "cd /opt/my-app && dotnet ef database update"

      - name: Restart service
        run: |
          ssh -i ~/.ssh/deploy_key \
            ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_IP }} \
            "sudo systemctl restart my-app"
```

> **No validate job, no test job, no format check.** The pre-push hook handles all of these locally.

### .NET Git Hooks (Without Husky)

Since .NET projects don't use Node.js, use Git's native hooks or a .NET-based solution.

**Option 1: Native Git Hook**

```bash
# .git/hooks/pre-push (make executable with chmod +x)
#!/bin/bash
./scripts/preflight.sh
```

**Option 2: Husky.Net (for .NET projects)**

```bash
# Install Husky.Net
dotnet tool install --global Husky

# Initialize in project
cd YourProject
dotnet husky install

# Add pre-push hook
dotnet husky add pre-push -c "dotnet build && dotnet format --verify-no-changes"
```

**Option 3: Polyglot Repository (Node + .NET)**

If your repo has a `package.json` at root, use standard Husky with a unified preflight:

```json
{
  "scripts": {
    "prepare": "husky",
    "preflight": "./scripts/preflight-unified.sh"
  }
}
```

### Key Differences from Node.js

| Aspect | Node.js/TypeScript | .NET |
|--------|-------------------|------|
| Type checking | `tsc --noEmit` | Built into `dotnet build` |
| Formatting | ESLint/Prettier | `dotnet format` |
| Security | `npm audit` | `dotnet list package --vulnerable` |
| Build | `npm run build` | `dotnet build` |
| Test | `npm test` | `dotnet test` |
| Lock file | `pnpm-lock.yaml` | `packages.lock.json` |
| Git hooks | Husky | Native hooks or Husky.Net |
| Monorepo tool | Turborepo | `dotnet build` (built-in solution support) |

---

## Platform-Specific Deployment Checks

Modern deployment platforms (Vercel, Azure, AWS Lambda) have specific requirements that should be validated during preflight. These checks prevent deployment failures that only surface in production.

### Vercel / Serverless (Node.js)

#### 1. Prisma Postinstall Script

Vercel requires Prisma client to be generated during the build. Apps with Prisma must have a `postinstall` script.

```bash
# Check: Verify apps with Prisma schema have postinstall
for pkg in apps/*/package.json; do
    app_dir=$(dirname "$pkg")
    if [ -f "$app_dir/prisma/schema.prisma" ]; then
        if ! grep -q '"postinstall"' "$pkg"; then
            echo "❌ Missing postinstall script in $pkg"
            echo "   Add: \"postinstall\": \"prisma generate\""
        fi
    fi
done
```

#### 2. Build-Time Client Initialization

External SDK clients (Stripe, Resend, OpenAI) initialized at module level will fail on Vercel because environment variables aren't available during build.

```bash
# Bad: Module-level initialization (fails on Vercel)
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!)  # ❌

# Good: Lazy initialization
let stripeClient: Stripe | null = null;
export const getStripe = () => {
    if (!stripeClient) {
        stripeClient = new Stripe(process.env.STRIPE_SECRET_KEY!);
    }
    return stripeClient;
};
```

**Preflight check:**

```bash
# Detect module-level SDK initialization
SEARCH_PATH="apps/*/lib apps/*/app apps/*/src"

# Stripe
if grep -rE "(const|let|var)\s+\w+\s*=\s*new Stripe\(" $SEARCH_PATH --include="*.ts" --include="*.tsx" 2>/dev/null | grep -v "getStripe" | grep -v "// ok"; then
    echo "⚠️ Stripe client initialized at module level - use lazy initialization"
fi

# Resend
if grep -rE "(const|let|var)\s+\w+\s*=\s*new Resend\(" $SEARCH_PATH --include="*.ts" --include="*.tsx" 2>/dev/null | grep -v "getResend"; then
    echo "⚠️ Resend client initialized at module level"
fi

# OpenAI / OpenRouter
if grep -rE "(const|let|var)\s+\w+\s*=\s*new OpenAI\(" $SEARCH_PATH --include="*.ts" --include="*.tsx" 2>/dev/null | grep -v "getOpenAI"; then
    echo "⚠️ OpenAI client initialized at module level"
fi
```

#### 3. File System in API Routes

Serverless functions can't use the `fs` module. Check API routes for incompatible imports.

```bash
# Check for fs imports in API routes (serverless incompatible)
SEARCH_API="apps/*/app/api apps/*/src/app/api"

if grep -rE "import.*from ['\"]fs['\"]|require\(['\"]fs['\"]\)" $SEARCH_API --include="*.ts" 2>/dev/null | grep -v "// server-ok"; then
    echo "⚠️ File system imports in API routes - incompatible with Edge runtime"
    echo "   Add '// server-ok' comment if intentionally using Node.js runtime"
fi
```

#### 4. Hardcoded Localhost References

Production code shouldn't reference localhost.

```bash
# Check for hardcoded localhost
if grep -rE "localhost:[0-9]+" apps/*/src --include="*.ts" --include="*.tsx" 2>/dev/null | grep -v "// dev-only" | grep -v ".env"; then
    echo "⚠️ Hardcoded localhost references found"
fi
```

### Azure / .NET Serverless

#### 1. Azure Functions Compatibility

Azure Functions have specific constraints similar to Vercel.

```bash
# Check for file system usage in Azure Functions
if grep -rE "System\.IO\.File\." src/ --include="*.cs" 2>/dev/null | grep -v "// azure-ok"; then
    echo "⚠️ File system usage detected - may not work in Azure Functions"
fi

# Check for static constructors with external dependencies
if grep -rE "static\s+\w+\s*\(" src/ --include="*.cs" | grep -E "(HttpClient|DbContext|new.*Client)"; then
    echo "⚠️ Static constructor with external dependency - use dependency injection"
fi
```

#### 2. Configuration Validation

```bash
# Verify required Azure configuration sections exist
if [ -f "appsettings.json" ]; then
    if ! grep -q "AzureAd\|Azure" appsettings.json 2>/dev/null; then
        echo "ℹ️ No Azure configuration found in appsettings.json"
    fi
fi

# Check for hardcoded connection strings
if grep -rE "Server=.*Password=" src/ --include="*.cs" --include="*.json" 2>/dev/null; then
    echo "❌ Hardcoded connection string detected - use environment variables"
fi
```

### Docker / Container Deployments

#### 1. Dockerfile Validation

```bash
# Check Dockerfile exists for containerized apps
for app_dir in apps/*/; do
    if [ -f "${app_dir}docker-compose.yml" ] || [ -f "${app_dir}Dockerfile" ]; then
        if [ ! -f "${app_dir}Dockerfile" ]; then
            echo "⚠️ docker-compose.yml found but no Dockerfile in $app_dir"
        fi
    fi
done

# Verify .dockerignore exists
if [ -f "Dockerfile" ] && [ ! -f ".dockerignore" ]; then
    echo "⚠️ Dockerfile exists but no .dockerignore - may include unnecessary files"
fi
```

#### 2. Multi-Stage Build Check

```bash
# Verify Dockerfile uses multi-stage builds for production
if [ -f "Dockerfile" ]; then
    if ! grep -q "FROM.*AS.*build\|FROM.*AS.*runtime" Dockerfile; then
        echo "ℹ️ Consider using multi-stage Dockerfile for smaller images"
    fi
fi
```

### Environment Variable Validation

Ensure required environment variables are documented and validated.

```bash
# Node.js: Check .env.example exists and is complete
if [ -f ".env.example" ]; then
    # Compare .env.example with actual .env
    if [ -f ".env" ]; then
        MISSING=$(comm -23 <(grep -oE "^[A-Z_]+=" .env.example | sort) <(grep -oE "^[A-Z_]+=" .env | sort))
        if [ -n "$MISSING" ]; then
            echo "⚠️ Missing environment variables in .env:"
            echo "$MISSING"
        fi
    fi
fi

# .NET: Validate appsettings structure
if command -v jq &> /dev/null && [ -f "appsettings.json" ]; then
    if ! jq empty appsettings.json 2>/dev/null; then
        echo "❌ Invalid JSON in appsettings.json"
    fi
fi
```

### Comprehensive Platform Check Script

Here's a unified function to add to any preflight script:

```bash
check_platform_compatibility() {
    local PLATFORM=${1:-"auto"}  # auto, vercel, azure, docker

    echo "🔍 Checking platform compatibility..."

    # Auto-detect platform
    if [ "$PLATFORM" = "auto" ]; then
        if [ -f "vercel.json" ]; then PLATFORM="vercel"
        elif [ -f "host.json" ]; then PLATFORM="azure"
        elif [ -f "Dockerfile" ]; then PLATFORM="docker"
        fi
    fi

    case $PLATFORM in
        vercel)
            echo "  Platform: Vercel (Node.js serverless)"
            check_prisma_postinstall
            check_build_time_init
            check_fs_in_api_routes
            ;;
        azure)
            echo "  Platform: Azure"
            check_azure_functions_compat
            check_dotnet_config
            ;;
        docker)
            echo "  Platform: Docker/Container"
            check_dockerfile
            check_dockerignore
            ;;
        *)
            echo "  Platform: Unknown (skipping platform checks)"
            ;;
    esac
}
```

---

## Single-App Projects

For non-monorepo projects, the strategy simplifies significantly.

### Next.js + NestJS (Standalone)

No Turborepo needed - run checks directly:

```bash
#!/bin/bash
# Standalone Next.js + NestJS Preflight

# Frontend checks
cd frontend
pnpm install --frozen-lockfile
pnpm type-check
pnpm lint
pnpm build

# Backend checks
cd ../backend
pnpm install --frozen-lockfile
pnpm type-check
pnpm lint
pnpm build
```

### Deploy Workflow for Standalone Next.js + NestJS

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  frontend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
      # Prepare standalone bundle, upload artifact, deploy via rsync...

  backend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: backend
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
      # Prepare deploy bundle, upload artifact, deploy via rsync...
```

> **No type-check, no lint, no test steps.** The pre-push hook handles all quality gates locally.

### .NET Web API (Standalone)

```bash
#!/bin/bash
# Standalone .NET API Preflight

dotnet restore
dotnet build --configuration Release
dotnet format --verify-no-changes

echo "✅ Ready for deployment"
```

---

## Hybrid Architectures

### Next.js Frontend + .NET Backend

Common in enterprise environments where frontend teams use React/Next.js while backend teams prefer .NET.

#### Project Structure

```
project/
├── frontend/           # Next.js app
│   ├── package.json
│   └── preflight.sh    # Node.js preflight
├── backend/            # .NET API
│   ├── MyApi.csproj
│   └── preflight.sh    # .NET preflight
└── .github/
    └── workflows/
        └── ci.yml      # Unified CI
```

#### Unified Preflight Script

```bash
#!/bin/bash
# Hybrid Project Preflight

ERRORS=0

echo "🚀 Running hybrid preflight..."

# Frontend (Next.js)
echo "📦 Checking frontend..."
cd frontend
if pnpm install --frozen-lockfile && pnpm type-check && pnpm lint && pnpm build; then
    echo "✅ Frontend passed"
else
    echo "❌ Frontend failed"
    ((ERRORS++))
fi
cd ..

# Backend (.NET)
echo "🔧 Checking backend..."
cd backend
if dotnet restore && dotnet build --configuration Release; then
    echo "✅ Backend passed"
else
    echo "❌ Backend failed"
    ((ERRORS++))
fi
cd ..

# Summary
if [ $ERRORS -eq 0 ]; then
    echo "✅ All checks passed - ready for deployment"
    exit 0
else
    echo "❌ $ERRORS component(s) failed"
    exit 1
fi
```

#### Unified Deploy Workflow

```yaml
name: Deploy

on:
  push:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  frontend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm build

  backend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: backend
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      - run: dotnet restore
      - run: dotnet build --no-restore --configuration Release
      - run: dotnet publish --no-build --configuration Release -o ./publish

  deploy:
    needs: [frontend, backend]
    runs-on: ubuntu-latest
    steps:
      # Download artifacts, rsync to server, run migrations, restart services
      - name: Deploy frontend and backend
        run: echo "Deploy via rsync..."
```

> **No test steps, no integration tests, no coverage.** Quality gates are handled by the pre-push hook. The deploy workflow builds both stacks and ships them.

---

## Architecture Decision Matrix

| Project Type | Preflight Tool | Deploy Strategy | Turbo Cache in CI |
|--------------|---------------|-----------------|-------------------|
| Turborepo monorepo | Bash + Turbo | Build + rsync | Yes (`actions/cache`) |
| Next.js standalone | Bash | Build + rsync | No (not needed) |
| NestJS standalone | Bash | Build + rsync | No (not needed) |
| .NET standalone | Bash | Build + rsync | No (not needed) |
| Next.js + NestJS | Bash (parallel) | Parallel build + rsync | No |
| Next.js + .NET | Bash (parallel) | Parallel build + rsync | No |
| Nx monorepo | Bash + Nx | Build + rsync | Yes (Nx cache) |

---

## Deployment Strategy

### Core Principle: Build Once, Deploy Artifacts

**Always ship pre-built artifacts, never build on the server.** This is the standard approach used by Vercel, AWS, Google Cloud, and all major CI/CD platforms.

| Approach | Recommended | Why |
|----------|-------------|-----|
| Build in CI, deploy artifact | **Yes** | Reproducible, fast deploys, clean separation |
| Build on server | No | Slow, unreliable, wastes server resources |
| Build locally, scp to server | No | Not reproducible, depends on local environment |

### Next.js: Standalone Mode (Required for Self-Hosted)

All self-hosted Next.js apps **must** use `output: 'standalone'` in `next.config.js`. This applies to both monorepo and standalone projects.

**Why standalone mode:**
- Traces only runtime dependencies (~7-80MB vs 1.5GB+ with full `node_modules`)
- Produces a self-contained `server.js` that replaces `next start`
- No need to install `node_modules` on the server
- Dramatically faster artifact upload, transfer, and deployment

**Configuration:**

```javascript
// next.config.js
const nextConfig = {
  output: 'standalone',
  outputFileTracingIncludes: {
    '/**': ['./node_modules/styled-jsx/**/*', './node_modules/@swc/helpers/**/*', './node_modules/@next/env/**/*'],
  },
  // ... other config
};
```

> **pnpm monorepo requirement:** `styled-jsx`, `@swc/helpers`, and `@next/env` are Next.js runtime dependencies that pnpm hoists outside the app's `node_modules`. The standalone tracer misses them. Add all three as explicit `dependencies` in the app's `package.json` and include them in `outputFileTracingIncludes` as shown above. The pre-push hook and preflight script validate this automatically.
>
> **Do NOT use `outputFileTracingRoot`** as an alternative. While it expands the tracer scope to the monorepo root, pnpm's `.pnpm` virtual store uses symlinks that break when the artifact is zipped/uploaded in CI. The result: files exist in `.pnpm/styled-jsx@.../node_modules/styled-jsx` but `node_modules/styled-jsx` is missing, so Node.js cannot resolve them. The explicit deps + `outputFileTracingIncludes` approach is the only reliable fix.

**Build artifact preparation (deploy workflow):**

```yaml
- name: Prepare standalone bundle
  run: |
    APP=apps/my-app/frontend
    # Copy standalone output (includes traced node_modules)
    cp -r $APP/.next/standalone/. /tmp/deploy/
    # Copy static assets into the app's location within standalone
    cp -r $APP/.next/static /tmp/deploy/$APP/.next/static
    # Copy public assets
    cp -r $APP/public /tmp/deploy/$APP/public 2>/dev/null || true
    echo "Bundle size:" && du -sh /tmp/deploy

- name: Upload build artifacts
  uses: actions/upload-artifact@v4
  with:
    name: my-app-frontend-build
    path: /tmp/deploy
    include-hidden-files: true
    retention-days: 1
```

> **Smoke tests run locally, not in CI.** The pre-push hook builds the standalone bundle, copies it with `cp -rL` (dereferencing symlinks to simulate the CI artifact pipeline), and boots `server.js` to catch missing dependencies. By the time code reaches the deploy workflow, it has already been validated.

> **`actions/upload-artifact@v4` gotcha:** Since v4.4.0, `include-hidden-files` defaults to `false`. This silently excludes `.next/` (a hidden directory) from the uploaded artifact. Always add `include-hidden-files: true`.

**Monorepo note:** Standalone output preserves the monorepo directory structure. The entry point is at `apps/my-app/frontend/server.js` within the bundle, not at the root.

**Server systemd service:**

```ini
[Service]
ExecStart=/usr/bin/node apps/my-app/frontend/server.js
Environment=NODE_ENV=production
Environment=PORT=3000
Environment=HOSTNAME=127.0.0.1
EnvironmentFile=/etc/my-app-frontend.env
```

### NestJS Backend: pnpm deploy --prod

For NestJS backends in a monorepo, use `pnpm deploy --prod` to create an isolated production bundle with resolved workspace dependencies.

**Important:** `pnpm deploy --prod` resolves workspace dependencies but also pulls in peer dependencies from the monorepo graph. Frontend-only packages (Next.js, Playwright, etc.) commonly leak into backend bundles via transitive peer deps. Always include a cleanup step.

```yaml
- name: Prepare deploy bundle
  run: |
    pnpm --filter @org/my-backend deploy --prod /tmp/deploy
    cp -r apps/my-app/backend/dist /tmp/deploy/dist
    cp -r apps/my-app/backend/prisma /tmp/deploy/prisma 2>/dev/null || true
    # Keep prisma.config.ts — Prisma 7 reads datasource URL from it
    echo "Before cleanup:" && du -sh /tmp/deploy
    cd /tmp/deploy
    # Remove frontend packages leaked via peer deps
    rm -rf node_modules/.pnpm/next@* node_modules/.pnpm/@next+swc-*
    rm -rf node_modules/.pnpm/@playwright+test@* node_modules/.pnpm/playwright@* node_modules/.pnpm/playwright-core@*
    rm -rf node_modules/.pnpm/@prisma+studio-core@*
    rm -rf node_modules/.pnpm/prettier@*
    rm -rf node_modules/.pnpm/@electric-sql+pglite@*
    rm -rf node_modules/{next,.next,@next,@playwright,playwright*,prettier}
    # Remove puppeteer browser binaries (keep puppeteer-core for runtime)
    find . -type d -name ".local-chromium" -exec rm -rf {} + 2>/dev/null || true
    find . -type d -name "chrome-linux64" -exec rm -rf {} + 2>/dev/null || true
    find . -type d -name "chrome-headless-shell-linux64" -exec rm -rf {} + 2>/dev/null || true
    find . -path "*/.cache/puppeteer" -exec rm -rf {} + 2>/dev/null || true
    echo "After cleanup:" && du -sh /tmp/deploy
```

> **Backend smoke tests run locally in the pre-push hook**, not in CI. The pre-push hook builds the backend bundle, boots `dist/main.js`, and checks for `Cannot find module` errors. `Cannot find module` errors crash during the synchronous require phase (first 2-3 seconds), before any async DB connection. If the process survives 3 seconds, all modules resolved. DB connection errors are expected and ignored.

> **Shared package design rule:** Shared packages that are used by both frontend and backend must not have top-level `import` of frontend-only modules. Use `import type` for type annotations and lazy `require()` inside functions that are only called from frontend code paths.

> **`tsconfig.build.json` for NestJS backends:** Must include `"include": ["src"]`. If root-level `.ts` files exist (e.g. `prisma.config.ts`), TypeScript infers `rootDir` as `./` and outputs `dist/src/main.js` instead of `dist/main.js`. Also set `nest-cli.json` asset `outDir` to `"dist"` (not `"dist/src"`).

### Heavy Dependencies in CI

Some production dependencies (e.g., puppeteer, chromium, sharp) download large binaries during install. Handle these in CI:

1. **Skip downloads at workflow level:**

```yaml
env:
  PUPPETEER_SKIP_DOWNLOAD: 'true'  # Must be workflow-level, not step-level
```

2. **Aggressive cleanup from deploy bundle** (see NestJS section above). Common bloat sources:
   - `next` + `@next/swc-*` (~500MB) - leaked via `next-auth` peer deps
   - `@playwright/test` (~17MB) - leaked via `next` peer deps
   - `@prisma/studio-core` (~53MB) - dev tooling
   - Duplicate `@prisma/client` versions - ensure all packages use the same major version

3. **Consider moving to devDependencies** if the binary is only needed for testing (e.g., puppeteer for E2E tests). Use `puppeteer-core` in production with a system-installed browser.

4. **Watch for duplicate package versions** across shared packages. A types package pinned to `@prisma/client@^6` while others use `^7` adds ~150MB of duplicate Prisma client.

### Reusable Deploy Workflow Pattern

For multi-app monorepos, create a reusable workflow that handles the common deploy steps:

```yaml
# .github/workflows/reusable-deploy.yml
on:
  workflow_call:
    inputs:
      app-name: { required: true, type: string }
      app-type: { required: true, type: string }  # frontend or backend
      artifact-name: { required: true, type: string }
      server-path: { required: true, type: string }
      service-name: { required: true, type: string }
      health-check-url: { required: false, type: string }
    secrets:
      SERVER_IP: { required: true }
      SERVER_USER: { required: true }
      SSH_KEY: { required: true }
      DATABASE_URL: { required: false }

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: ${{ inputs.artifact-name }}
          path: deploy-files
      - name: Setup SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_KEY }}" > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key
          ssh-keyscan -H ${{ secrets.SERVER_IP }} >> ~/.ssh/known_hosts
      - name: Deploy via rsync
        run: |
          rsync -rz --delete \
            -e "ssh -i ~/.ssh/deploy_key" \
            --no-times --no-perms --no-owner --no-group --omit-dir-times \
            deploy-files/ \
            ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_IP }}:${{ inputs.server-path }}/
      - name: Run database migrations
        if: inputs.app-type == 'backend'
        run: |
          ssh -i ~/.ssh/deploy_key -o ServerAliveInterval=30 \
            ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_IP }} \
            "cd ${{ inputs.server-path }} && DATABASE_URL='${{ secrets.DATABASE_URL }}' npx prisma migrate deploy"
      - name: Restart service
        run: |
          ssh -i ~/.ssh/deploy_key \
            ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_IP }} \
            "sudo systemctl restart ${{ inputs.service-name }}"
```

**Key patterns:**
- `DATABASE_URL` passed via GitHub Secrets, not read from server env files (avoids permission issues)
- `sudo systemctl restart` (passwordless sudo for this command only)
- No `--schema` flag — Prisma 7 reads schema path and datasource URL from `prisma.config.ts`
- No `systemctl is-active` check — the health check URL step handles service verification
- `npx prisma` to resolve from deployed `node_modules`

### Environment Variables

**Two separate concerns:**

1. **Runtime env vars** (for the running service): Store in `/etc/{service-name}.env` files, referenced by systemd `EnvironmentFile=`. Owned by `root:root 600`.

2. **Deploy-time env vars** (for migrations): Pass via GitHub Secrets in the workflow. Do **not** read from server env files during deploy — the deploy user typically cannot read root-owned files.

### Deployment Checklist

- [ ] Next.js apps use `output: 'standalone'`
- [ ] `styled-jsx`, `@swc/helpers`, `@next/env` added as explicit deps + `outputFileTracingIncludes`
- [ ] Pre-push hook runs standalone smoke test locally (boots `server.js`)
- [ ] `upload-artifact` uses `include-hidden-files: true` (required since v4.4.0)
- [ ] Turbo cache restored via `actions/cache` for `.turbo/` directory
- [ ] Deploy workflow has NO quality gates (build + deploy only)
- [ ] Frontend artifacts are <100MB (standalone mode)
- [ ] Backend artifacts <500MB after cleanup (remove leaked frontend deps)
- [ ] `PUPPETEER_SKIP_DOWNLOAD: 'true'` set at workflow-level env
- [ ] All shared packages use the same major version of heavy deps (e.g., `@prisma/client`)
- [ ] `DATABASE_URL` passed via GitHub Secrets for migrations
- [ ] Runtime env vars in `/etc/*.env` referenced by systemd `EnvironmentFile=`
- [ ] Systemd services configured for each app
- [ ] Deploy user has passwordless sudo for `systemctl restart {service}` only
- [ ] Health check configured (if available)
- [ ] Reusable deploy workflow handles common steps

---

## Related Standards

- [CI/CD Pipelines](/standards/development/ci-cd-pipelines) — Deploy workflow templates and caching strategies. This is the "deploy only" counterpart to the preflight strategy defined here.
- [Git Hooks (Husky)](/standards/development/git-hooks-husky) — Implementation details for the pre-push hook that enforces these quality gates.

## References

- [Turborepo Affected Filter](https://turbo.build/repo/docs/reference/run#--filter)
- [GitHub Actions Concurrency](https://docs.github.com/en/actions/using-jobs/using-concurrency)
- [Google's Approach to Monorepos](https://cacm.acm.org/magazines/2016/7/204032-why-google-stores-billions-of-lines-of-code-in-a-single-repository/)
- [.NET CLI Documentation](https://learn.microsoft.com/en-us/dotnet/core/tools/)
- [Nx Affected Commands](https://nx.dev/nx-api/nx/documents/affected)
- [Next.js Standalone Output](https://nextjs.org/docs/app/api-reference/config/next-config-js/output)

---

*This strategy ensures fast developer feedback loops locally while keeping deploy workflows lean and focused. The key insight: quality gates belong in the pre-push hook where developers get immediate feedback, and deploy workflows should only build and ship. This approach scales from standalone apps to complex monorepos across multiple tech stacks.*
