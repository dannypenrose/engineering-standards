# CI/CD Pipeline Standards

> Authoritative CI/CD pipeline standards for GitHub Actions deployments to self-hosted infrastructure. Covers all stack types: Next.js, .NET, Node.js monorepos, WordPress, and static sites.

## Purpose

Define consistent, fast, and reliable deployment pipelines across all projects. These standards are derived from production experience deploying to DigitalOcean and apply to any self-hosted target using rsync + systemd.

## Core Principles

1. **OneRunner** — Build and deploy in a single job. No artefact upload/download between jobs.
2. **Cache everything** — npm/pnpm/NuGet packages and framework build caches.
3. **Concurrency control** — Prevent overlapping deploys to the same app.
4. **Path-based triggers** — Only deploy when relevant files change.
5. **Manual trigger** — Every deploy workflow supports `workflow_dispatch`.
6. **Quality gates locally, not in CI** — Pre-push hooks handle lint, type-check, and test. CI only builds and deploys.
7. **SSH keepalive** — Prevent broken pipes during long operations.

## OneRunner Pattern

All deploy workflows use a single job that checks out, builds, and deploys from the same runner. This eliminates artefact transfer overhead and duplicate installs.

```yaml
jobs:
  deploy:                           # or build-and-deploy
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
      # Setup runtime, cache, install, build, SSH, rsync, restart
      # All in one job — no artefact upload/download
```

**Why not multi-job?** The old pattern uploaded build artefacts then downloaded them in a separate deploy job. For large bundles, the upload alone took 7+ minutes. OneRunner eliminates this entirely.

**Exception:** Monorepo workflows may use a lightweight `detect-changes` job before the main `build-and-deploy` job to determine which apps changed. This is acceptable because the detection job is fast (~5s) and carries no artefacts.

## Workflow Skeleton

Every deploy workflow follows this structure:

```yaml
name: Deploy {APP_NAME}

on:
  push:
    branches:
      - main
    paths:
      - '{SOURCE_DIR}/**'
  workflow_dispatch:

concurrency:
  group: deploy-{APP_NAME}-${{ github.ref }}
  cancel-in-progress: true

env:
  # Runtime versions as workflow-level env vars
  NODE_VERSION: '20'          # Node.js projects
  # DOTNET_VERSION: '8.0.x'  # .NET projects

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: prod          # If secrets are scoped to a GitHub environment
    steps:
      # 1. Checkout
      # 2. Setup runtime + cache
      # 3. Install dependencies
      # 4. Build
      # 5. Setup SSH
      # 6. rsync to server
      # 7. Restart service
```

### Required Elements

| Element | Purpose | Required |
|---------|---------|----------|
| `paths:` trigger | Only deploy when relevant files change | Yes |
| `workflow_dispatch:` | Allow manual deploys | Yes |
| `concurrency:` | Prevent overlapping deploys | Yes |
| `environment:` | Access environment-scoped secrets | If secrets are scoped |
| Runtime version env var | Consistent, easy to update | Yes |
| Dependency caching | Faster installs | Yes |
| Build caching | Faster builds | Yes (where available) |
| SSH keepalive | Prevent broken pipes | Yes |

> **GitHub Environments:** If secrets (SSH keys, API tokens, database URLs) are configured under a [GitHub environment](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment) rather than as repo-level secrets, the job **must** include `environment: <name>`. Without it, `${{ secrets.* }}` resolves to empty strings and the build will fail silently or with cryptic errors like `Invalid URL`.

## Concurrency Control

Every deploy workflow must include a concurrency group:

```yaml
concurrency:
  group: deploy-{APP_NAME}-${{ github.ref }}
  cancel-in-progress: true
```

- **Group format:** `deploy-{APP_NAME}-${{ github.ref }}`
- **`cancel-in-progress: true`** — If a new push arrives while deploying, cancel the in-progress deploy and start the new one.

## Caching Strategies

### Node.js (npm)

```yaml
- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version: ${{ env.NODE_VERSION }}
    cache: 'npm'
    cache-dependency-path: '{SOURCE_DIR}/package-lock.json'
```

**Requirement:** `package-lock.json` must be committed. Without it, the cache key cannot be generated and the step fails.

### Node.js (pnpm)

```yaml
- uses: pnpm/action-setup@v4
- uses: actions/setup-node@v4
  with:
    node-version: ${{ env.NODE_VERSION }}
    cache: 'pnpm'
```

### .NET (NuGet)

```yaml
- name: Cache NuGet packages
  uses: actions/cache@v4
  with:
    path: ~/.nuget/packages
    key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
    restore-keys: |
      ${{ runner.os }}-nuget-
```

### Next.js Build Cache

Speeds up incremental builds by caching `.next/cache`:

```yaml
- name: Cache Next.js build
  uses: actions/cache@v4
  with:
    path: {SOURCE_DIR}/.next/cache
    key: ${{ runner.os }}-nextjs-${{ hashFiles('{SOURCE_DIR}/package-lock.json') }}-${{ hashFiles('{SOURCE_DIR}/**/*.tsx', '{SOURCE_DIR}/**/*.ts') }}
    restore-keys: |
      ${{ runner.os }}-nextjs-${{ hashFiles('{SOURCE_DIR}/package-lock.json') }}-
      ${{ runner.os }}-nextjs-
```

### Turborepo Cache (Monorepos)

```yaml
- name: Restore Turbo cache
  uses: actions/cache@v4
  with:
    path: .turbo
    key: turbo-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}-${{ hashFiles('packages/**/package.json') }}-${{ github.sha }}
    restore-keys: |
      turbo-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}-${{ hashFiles('packages/**/package.json') }}-
      turbo-${{ runner.os }}-
```

Three-tier key strategy:
1. **Exact match** (lockfile + packages + commit SHA) — full cache hit.
2. **Lockfile + packages match** — same deps and shared package metadata, Turbo invalidates changed packages.
3. **OS match** (runner OS only) — fallback, avoids cold-start rebuilds.

> **Important:** Include `packages/**/package.json` in the cache key. Without it, changes to shared package metadata (version bumps, new exports) won't bust the Turbo cache, causing stale builds of downstream apps.

## SSH & Deployment

### SSH Setup

```yaml
- name: Setup SSH Agent
  uses: webfactory/ssh-agent@v0.9.0
  with:
    ssh-private-key: ${{ secrets.SSH_KEY }}

- name: Add server to known_hosts
  run: |
    mkdir -p ~/.ssh
    ssh-keyscan -H ${{ secrets.SERVER_IP }} >> ~/.ssh/known_hosts
```

### rsync Flags

Always use the same flag set for consistency:

```yaml
rsync -rz --delete \
  --no-times --no-perms --no-owner --no-group --omit-dir-times \
  {SOURCE}/ \
  ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_IP }}:{DEST}/
```

| Flag | Purpose |
|------|---------|
| `-r` | Recursive |
| `-z` | Compress during transfer |
| `--delete` | Remove files on server that are no longer in source |
| `--no-times --no-perms --no-owner --no-group` | Avoid permission conflicts with systemd service user |
| `--omit-dir-times` | Skip directory timestamp updates |
| `-L` | Dereference symlinks (required for monorepo backends using `pnpm deploy`) |

**Monorepo backends (`pnpm deploy --prod`):** Add `-L` (`--copy-links`) to dereference workspace symlinks. `pnpm deploy` creates symlinks for workspace packages (e.g. `@org/*`) in `node_modules/`. Without `-L`, rsync silently skips them and shared module code never reaches the server.

> **Important:** If the deploy bundle cleanup removes large packages (e.g. `next`, `@playwright`, `prettier`), run `find . -xtype l -delete 2>/dev/null || true` afterwards to prune dangling symlinks. With `-L`, rsync will error on broken symlinks (`symlink has no referent`, exit code 23).

```yaml
# Backend (monorepo with workspace packages)
rsync -rzL --delete \
  --no-times --no-perms --no-owner --no-group --omit-dir-times \
  /tmp/deploy-backend/ \
  ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_IP }}:{DEST}/
```

**Shared package `dist/` in monorepos:** If the `packages/` directory has a `.gitignore` that excludes `dist/`, `pnpm deploy` will also exclude built output from the bundle. Add `"files": ["dist"]` to each shared package's `package.json` to override `.gitignore` and ensure compiled code is included.

**Common excludes:**
- `.NET`: `--exclude 'appsettings*.json'` — server config stays on server
- `Next.js`: No excludes needed (build output only)

### SSH Keepalive

Always use `-o ServerAliveInterval=30` on SSH commands that may take time (installs, migrations, restarts):

```yaml
- name: Install and Restart
  run: |
    ssh -o ServerAliveInterval=30 ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_IP }} <<EOF
      cd /var/www/{APP_NAME}
      npm ci --omit=dev
      sudo systemctl restart {APP_NAME}
    EOF
```

### SERVER_USER

`SERVER_USER` must be the **app-specific deploy user** (e.g., `my-app`, `api-my-app`), not `ops` or `root`. This ensures rsync writes files owned by the correct user, matching the systemd service.

## Quality Gates: Pre-Push, Not CI

Deploy workflows do **not** run type-check, lint, test, or smoke tests. These are handled by local pre-push hooks.

| Local preflight (pre-push) | Deploy workflow (GitHub Actions) |
|---|---|
| Type checking | Build only |
| Linting | rsync to server |
| Build verification | Database migrations (if applicable) |
| Smoke tests | Service restart |
| Dependency verification | |

**Trade-off:** This relies on developers not bypassing the hook (`--no-verify`). For teams, add a separate `ci-quality` workflow on pull requests instead of the deploy path.

## Stack-Specific Patterns

### Next.js (npm, standalone repo)

```yaml
name: Deploy {APP_NAME}

on:
  push:
    branches: [main]
    paths: ['{SOURCE_DIR}/**']
  workflow_dispatch:

concurrency:
  group: deploy-{APP_NAME}-${{ github.ref }}
  cancel-in-progress: true

env:
  NODE_VERSION: '20'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          cache-dependency-path: '{SOURCE_DIR}/package-lock.json'

      - name: Cache Next.js build
        uses: actions/cache@v4
        with:
          path: {SOURCE_DIR}/.next/cache
          key: ${{ runner.os }}-nextjs-${{ hashFiles('{SOURCE_DIR}/package-lock.json') }}-${{ hashFiles('{SOURCE_DIR}/**/*.tsx', '{SOURCE_DIR}/**/*.ts') }}
          restore-keys: |
            ${{ runner.os }}-nextjs-${{ hashFiles('{SOURCE_DIR}/package-lock.json') }}-
            ${{ runner.os }}-nextjs-

      - name: Install Dependencies
        working-directory: {SOURCE_DIR}
        run: npm ci

      - name: Build Application
        working-directory: {SOURCE_DIR}
        env:
          NODE_ENV: production
        run: npm run build

      - name: Setup SSH Agent
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.SSH_KEY }}

      - name: Add server to known_hosts
        run: |
          mkdir -p ~/.ssh
          ssh-keyscan -H ${{ secrets.SERVER_IP }} >> ~/.ssh/known_hosts

      - name: Sync Files
        run: |
          rsync -rz --delete \
            --no-times --no-perms --no-owner --no-group --omit-dir-times \
            {SOURCE_DIR}/.next/ \
            ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_IP }}:/var/www/{APP_NAME}/.next/

          rsync -rz \
            --no-times --no-perms --no-owner --no-group --omit-dir-times \
            {SOURCE_DIR}/package.json {SOURCE_DIR}/package-lock.json \
            ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_IP }}:/var/www/{APP_NAME}/

          rsync -rz \
            --no-times --no-perms --no-owner --no-group --omit-dir-times \
            {SOURCE_DIR}/public/ \
            ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_IP }}:/var/www/{APP_NAME}/public/

      - name: Install and Restart
        run: |
          ssh -o ServerAliveInterval=30 ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_IP }} <<EOF
            cd /var/www/{APP_NAME}
            npm ci --omit=dev
            sudo systemctl restart {APP_NAME}
          EOF
```

#### NEXT_PUBLIC Environment Variables

Two approaches:

1. **`.env.production` committed to repo** (preferred for non-sensitive values). Next.js loads it automatically during build. No need to add `NEXT_PUBLIC_*` vars to GitHub Secrets.

2. **GitHub Secrets** (required for sensitive values). Add each variable to the `Build Application` step's `env:` block.

Never put secrets with a `NEXT_PUBLIC_` prefix — they become visible in the browser bundle.

#### Prisma (if applicable)

Add these steps between Install and Build:

```yaml
- name: Generate Prisma Client
  working-directory: {SOURCE_DIR}
  run: npx prisma generate
```

And sync the Prisma schema + generated client in the Sync Files step:

```yaml
# After other rsync commands
if [ -d "{SOURCE_DIR}/prisma" ]; then
  rsync -rz \
    --no-times --no-perms --no-owner --no-group --omit-dir-times \
    {SOURCE_DIR}/prisma/ \
    ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_IP }}:/var/www/{APP_NAME}/prisma/
fi
```

Run `prisma generate` on the server after install to produce the correct Linux binary.

### .NET (ASP.NET Core)

```yaml
name: Deploy {APP_NAME}

on:
  push:
    branches: [main]
    paths: ['{PROJECT_DIR}/**']
  workflow_dispatch:

concurrency:
  group: deploy-{APP_NAME}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('{PROJECT_DIR}/**/*.csproj') }}
          restore-keys: |
            ${{ runner.os }}-nuget-

      - name: Publish Application
        run: dotnet publish {PROJECT_DIR}/{PROJECT_NAME}.csproj -c Release -o publish

      - name: Setup SSH Agent
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.SSH_KEY }}

      - name: Add server to known_hosts
        run: |
          mkdir -p ~/.ssh
          ssh-keyscan -H ${{ secrets.SERVER_IP }} >> ~/.ssh/known_hosts

      - name: Deploy Files
        run: |
          rsync -rz --delete \
            --no-times --no-perms --no-owner --no-group --omit-dir-times \
            --exclude 'appsettings*.json' \
            publish/ \
            ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_IP }}:/var/www/{APP_NAME}/

      - name: Restart Service
        run: |
          ssh -o ServerAliveInterval=30 ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_IP }} \
            "sudo systemctl restart {APP_NAME}"
```

**.NET-specific notes:**
- `dotnet publish -c Release -o publish` produces a self-contained output directory.
- Always exclude `appsettings*.json` — server config stays on the server.
- Server env vars are loaded from `/etc/{APP_NAME}.env` via `EnvironmentFile=` in systemd.
- `ProtectSystem=strict` in the systemd unit locks down filesystem access.

#### EF Core Migrations (if applicable)

Add migration step after deploy, before restart:

```yaml
- name: Run Migrations
  run: |
    ssh -o ServerAliveInterval=30 ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_IP }} \
      "cd /var/www/{APP_NAME} && dotnet {APP_DLL}.dll --migrate"
```

Or handle migrations on application startup via `Database.Migrate()`.

### Monorepo (Turborepo + pnpm)

Key additions beyond the standard skeleton for monorepo deployments:

- **`detect-changes` job** using `dorny/paths-filter@v3` to determine which apps changed.
- **Turbo cache** via `actions/cache@v4` for `.turbo/` directory.
- **Filtered builds** using `pnpm --filter @org/{APP_NAME}... build`.
- **Separate SSH keys per role** — frontend and backend deploy users each have their own key.
- **`NODE_OPTIONS: '--max-old-space-size=4096'`** to prevent OOM during builds.
- **`PUPPETEER_SKIP_DOWNLOAD: 'true'`** if puppeteer is a dependency.

### WordPress

WordPress deploys are simpler — no build step, just rsync themes and mu-plugins:

```yaml
- name: Deploy theme
  run: |
    rsync -az --delete \
      wp-content/themes/{THEME_NAME}/ \
      ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_IP }}:/var/www/{APP_NAME}/wp-content/themes/{THEME_NAME}/
```

No service restart needed. No caching required.

### Static Sites

Static site deploys are the simplest — checkout and rsync:

```yaml
- name: Deploy Files
  run: |
    rsync -rz --delete \
      --no-times --no-perms --no-owner --no-group --omit-dir-times \
      {SOURCE_DIR}/ \
      ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_IP }}:/var/www/{APP_NAME}/
```

No service restart, no install, no build (unless using a static site generator).

## Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|---|---|---|
| Multi-job with artefact transfer | Slow (7+ min upload), duplicates install | OneRunner: single job |
| Quality gates in deploy CI | Duplicates pre-push hook work | Pre-push hooks locally |
| Reusable deploy workflow | Adds indirection without real DRY benefit | Inline deploy steps |
| `--no-verify` on git push | Bypasses quality gates | Never skip hooks |
| `npm install --production` | Deprecated flag | `npm ci --omit=dev` |
| Hardcoded Node version | Hard to update | `env.NODE_VERSION` |
| Missing `concurrency:` | Overlapping deploys corrupt state | Always add concurrency group |
| `SERVER_USER: ops` | Permission conflicts with systemd | Use app-specific deploy user |

## Checklist

When creating a new deploy workflow, verify:

- [ ] `paths:` trigger scoped to relevant directories
- [ ] `workflow_dispatch:` present for manual deploys
- [ ] `concurrency:` group with `cancel-in-progress: true`
- [ ] `environment:` set if secrets are scoped to a GitHub environment
- [ ] Runtime version as workflow-level `env:` variable
- [ ] Dependency cache configured (`setup-node` cache, `actions/cache` for NuGet)
- [ ] Build cache configured (Next.js `.next/cache`, Turbo `.turbo/`)
- [ ] Single job (OneRunner) — no artefact upload/download
- [ ] rsync uses standard flag set (`-rz --delete --no-times --no-perms --no-owner --no-group --omit-dir-times`)
- [ ] SSH keepalive (`-o ServerAliveInterval=30`) on long-running SSH commands
- [ ] `SERVER_USER` is the app-specific deploy user
- [ ] No lint/test/type-check steps (these belong in pre-push hooks)
- [ ] Lockfile committed to repo (required for cache key generation)
- [ ] Server config files excluded from rsync (`appsettings*.json`, `.env`)

## Related Standards

- [CI Preflight Strategy](/standards/quality/ci-preflight-strategy) — Defines the local-first quality gate system. Deploy workflows here are the "deploy only" half of that strategy.
- [Git Hooks (Husky)](/standards/development/git-hooks-husky) — Pre-push hook implementation that enforces quality gates locally.
- [Git Standards](/standards/development/git-standards) — Branch naming, commit conventions, and merge strategies.
