# Dependency Management

> Authoritative dependency management standards for security scanning, license compliance, and update policies.

## Purpose

Establish policies for managing third-party dependencies to ensure security, maintainability, and reliability across all projects.

## Core Principles

1. **Security first** - Vulnerabilities must be addressed promptly
2. **Minimal footprint** - Only add dependencies that provide clear value
3. **Version pinning** - Reproducible builds are non-negotiable
4. **Regular updates** - Stay current to reduce security debt
5. **License compliance** - Know what you're shipping
6. **Audit trail** - Track why dependencies were added

## Dependency Evaluation

### Before Adding a Dependency

```markdown
## New Dependency Checklist

Before adding a new dependency, evaluate:

- [ ] **Necessity** - Can this be done with existing deps or stdlib?
- [ ] **Maintenance** - Is it actively maintained? (commits in last 6 months)
- [ ] **Popularity** - Is it widely used? (npm downloads, GitHub stars)
- [ ] **Size** - What's the bundle impact? (use bundlephobia.com)
- [ ] **Security** - Any known vulnerabilities? (snyk.io, npm audit)
- [ ] **License** - Is the license compatible? (see License Matrix)
- [ ] **Types** - Does it have TypeScript types?
- [ ] **Quality** - Does it have tests? Good documentation?
```

### Evaluation Matrix

| Factor | Weight | Threshold |
| ------ | ------ | --------- |
| Last commit | 20% | < 6 months ago |
| Open issues response | 15% | < 30 day average |
| Weekly downloads | 15% | > 10,000 for npm |
| GitHub stars | 10% | > 500 |
| Bundle size | 15% | < 50KB gzipped |
| Known vulnerabilities | 25% | 0 critical/high |

### Alternative Approaches

```typescript
// Before adding a dependency, consider:

// 1. Native/stdlib solution
// ❌ import _ from 'lodash';
// ❌ const merged = _.merge(obj1, obj2);
// ✅ const merged = { ...obj1, ...obj2 };

// 2. Smaller, focused packages
// ❌ import moment from 'moment'; // 300KB
// ✅ import { format } from 'date-fns'; // 2KB per function

// 3. Copy small utilities (with attribution)
// For < 50 lines, consider vendoring instead of adding dep
// src/lib/utils/debounce.ts (from lodash, MIT license)

// 4. Dev dependencies vs runtime
// ❌ dependencies: { "prettier": "^3.0.0" }
// ✅ devDependencies: { "prettier": "^3.0.0" }
```

## Version Strategy

### Pinning Policy

```json
// package.json
{
  "dependencies": {
    // ✅ Pin exact versions for production dependencies
    "next": "14.1.0",
    "react": "18.2.0",
    "prisma": "5.8.1"
  },
  "devDependencies": {
    // ✅ Allow minor updates for dev tools
    "typescript": "^5.3.0",
    "eslint": "^8.56.0",
    "vitest": "^1.2.0"
  }
}
```

### Version Range Guidelines

| Dependency Type | Version Range | Rationale |
| --------------- | ------------- | --------- |
| Frameworks (Next, Nest) | Exact pin | Breaking changes common |
| Database clients | Exact pin | Schema compatibility |
| UI libraries | Exact pin | Visual regressions |
| Utilities (lodash, date-fns) | Minor (`^`) | Stable APIs |
| Dev tools | Minor (`^`) | Not shipped to users |
| Types (`@types/*`) | Minor (`^`) | Declarations only |

### Lockfile Requirements

```bash
# Always commit lockfiles
# .gitignore should NOT include:
# - package-lock.json
# - pnpm-lock.yaml
# - yarn.lock
# - packages.lock.json (.NET)

# Install with frozen lockfile in CI
pnpm install --frozen-lockfile   # pnpm
npm ci                            # npm
yarn install --frozen-lockfile    # yarn
dotnet restore --locked-mode      # .NET
```

## Update Cadence

### Update Schedule

| Update Type | Frequency | Process |
| ----------- | --------- | ------- |
| Security patches | Immediately | Automated PR, expedited review |
| Patch versions | Weekly | Automated PR, standard review |
| Minor versions | Bi-weekly | Manual review, test thoroughly |
| Major versions | Quarterly | Planned upgrade, ADR if significant |

### Automated Updates

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 10
    groups:
      # Group minor/patch updates
      minor-and-patch:
        patterns:
          - "*"
        update-types:
          - "minor"
          - "patch"
    ignore:
      # Don't auto-update major versions
      - dependency-name: "*"
        update-types: ["version-update:semver-major"]
    labels:
      - "dependencies"
    commit-message:
      prefix: "chore(deps)"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

### Renovate Alternative

```json
// renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended",
    ":semanticCommits",
    "group:allNonMajor"
  ],
  "schedule": ["before 6am on monday"],
  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security"]
  },
  "packageRules": [
    {
      "matchUpdateTypes": ["major"],
      "labels": ["major-update"],
      "automerge": false
    },
    {
      "matchUpdateTypes": ["minor", "patch"],
      "automerge": true,
      "automergeType": "pr",
      "requiredStatusChecks": ["ci"]
    },
    {
      "matchPackagePatterns": ["^@types/"],
      "automerge": true
    }
  ]
}
```

## Security Scanning

### Vulnerability Response SLAs

| Severity | Response Time | Resolution Time |
| -------- | ------------- | --------------- |
| Critical | 4 hours | 24 hours |
| High | 24 hours | 7 days |
| Medium | 7 days | 30 days |
| Low | 30 days | 90 days |

### Scanning Tools

```bash
# Node.js
npm audit                    # Built-in
pnpm audit                   # Built-in
npx snyk test                # Comprehensive, needs account

# .NET
dotnet list package --vulnerable --include-transitive

# Universal
# Snyk, Dependabot, or Renovate for automated scanning
```

### CI Integration

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: '0 6 * * *' # Daily at 6 AM

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Run security audit
        run: pnpm audit --audit-level=high
        continue-on-error: true

      - name: Upload audit results
        if: failure()
        run: |
          pnpm audit --json > audit-results.json
          # Send to security dashboard or create issue

  snyk:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
```

## License Compliance

### Approved Licenses

| License | Category | Commercial Use | Notes |
| ------- | -------- | -------------- | ----- |
| MIT | Permissive | ✅ Yes | Preferred |
| Apache 2.0 | Permissive | ✅ Yes | Patent grant included |
| BSD (2/3-clause) | Permissive | ✅ Yes | Similar to MIT |
| ISC | Permissive | ✅ Yes | Simplified MIT |
| CC0 | Public Domain | ✅ Yes | No restrictions |

### Requires Review

| License | Category | Concern |
| ------- | -------- | ------- |
| LGPL | Weak copyleft | Linking requirements |
| MPL 2.0 | Weak copyleft | File-level copyleft |
| EPL | Weak copyleft | Patent retaliation |

### Prohibited Licenses

| License | Category | Reason |
| ------- | -------- | ------ |
| GPL v2/v3 | Strong copyleft | Viral licensing |
| AGPL | Strong copyleft | Network use triggers copyleft |
| SSPL | Restricted | Service provider restrictions |
| Commercial | Proprietary | Requires license purchase |

### License Scanning

```bash
# Check licenses in project
npx license-checker --summary
npx license-checker --onlyAllow "MIT;Apache-2.0;BSD-2-Clause;BSD-3-Clause;ISC;CC0-1.0"

# Fail CI on prohibited licenses
npx license-checker --failOn "GPL;AGPL;SSPL"
```

```yaml
# .github/workflows/license.yml
name: License Check

on: [push, pull_request]

jobs:
  license:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: pnpm install --frozen-lockfile
      - name: Check licenses
        run: |
          npx license-checker --onlyAllow "MIT;Apache-2.0;BSD-2-Clause;BSD-3-Clause;ISC;CC0-1.0;0BSD;Unlicense"
```

## Dependency Documentation

### Package.json Metadata

```json
{
  "name": "@company/service",
  "version": "1.0.0",
  "description": "Brief service description",
  "license": "UNLICENSED",
  "private": true,
  "engines": {
    "node": ">=20.0.0",
    "pnpm": ">=8.0.0"
  },
  "packageManager": "pnpm@8.14.0",
  "dependencies": {
    // Group and comment large dependency lists
    // === Framework ===
    "next": "14.1.0",
    "react": "18.2.0",

    // === Database ===
    "@prisma/client": "5.8.1",

    // === Utilities ===
    "zod": "3.22.4"
  }
}
```

### Dependency Decisions Log

```markdown
<!-- docs/dependencies.md -->
# Dependency Decisions

## Why We Use X

### next (Framework)
- **Added:** 2024-01-01
- **Why:** React framework with built-in SSR, routing, and optimization
- **Alternatives considered:** Remix (less mature), Vite+React (need more setup)
- **Concerns:** Large bundle, vendor lock-in

### prisma (ORM)
- **Added:** 2024-01-01
- **Why:** Type-safe database access, excellent DX, auto-generated types
- **Alternatives considered:** Drizzle (newer, less mature), TypeORM (worse DX)
- **Concerns:** Build step required, migration lock-in
```

## Undeclared Dependency Detection

### The Problem

In pnpm monorepos, workspace hoisting makes packages available locally even when they're not declared in an app's `package.json`. This means code works in development but fails in CI or production where only declared dependencies are installed.

Common symptoms:
- `Cannot find module 'X'` in CI builds
- Type errors for missing type declarations in CI
- Dead files importing packages that were never added as dependencies

### Static Analysis Tools

Two tools scan source files for imports and verify they're declared in the nearest `package.json`:

```bash
# Backend dependency checker (NestJS apps)
node tools/check-backend-deps.mjs

# Frontend dependency checker (Next.js apps)
node tools/check-frontend-deps.mjs

# Check only staged files (used in pre-commit hook)
node tools/check-frontend-deps.mjs --staged
```

**How they work**:
1. Discover all apps in the monorepo (split or flat structure)
2. Walk source files and extract `import`/`require` specifiers
3. Build an allowed set: declared deps + one level of transitive deps
4. Report any packages imported but not in the allowed set

**What they skip**:
- Relative imports (`./`, `../`)
- Node.js builtins (`fs`, `path`, `crypto`, etc.)
- Path aliases (`@/`, `lib/`)
- Type-only imports (`import type`)
- Test files (`.spec.ts`, `.test.ts`)

### Hook Integration

| Hook | Tool | Mode | Blocking |
|------|------|------|----------|
| Pre-commit | `check-frontend-deps.mjs --staged` | Staged files only | Yes |
| Pre-push | `check-backend-deps.mjs` | Full scan | Yes |
| Pre-push | `check-frontend-deps.mjs` | Full scan | Yes |

## Monorepo Considerations

### Shared Dependencies

```yaml
# pnpm-workspace.yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

```json
// Root package.json
{
  "devDependencies": {
    // Shared dev dependencies at root
    "typescript": "^5.3.0",
    "eslint": "^8.56.0",
    "prettier": "^3.2.0"
  }
}
```

### Version Consistency

```bash
# Check for duplicate packages with different versions
pnpm why <package-name>

# Deduplicate
pnpm dedupe
```

## .NET Dependency Management

### NuGet Package Management

```xml
<!-- Directory.Build.props (at solution root) -->
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
</Project>

<!-- Directory.Packages.props -->
<Project>
  <ItemGroup>
    <!-- Centralized version management -->
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="8.0.1" />
    <PackageVersion Include="Serilog" Version="3.1.1" />
    <PackageVersion Include="FluentValidation" Version="11.9.0" />
  </ItemGroup>
</Project>

<!-- Individual .csproj files -->
<ItemGroup>
  <!-- No version specified - uses central version -->
  <PackageReference Include="Microsoft.EntityFrameworkCore" />
  <PackageReference Include="Serilog" />
</ItemGroup>
```

### .NET Vulnerability Scanning

```bash
# Check for vulnerable packages
dotnet list package --vulnerable

# Include transitive dependencies
dotnet list package --vulnerable --include-transitive

# Generate SBOM
dotnet CycloneDX <project.csproj> -o sbom.xml
```

## Software Bill of Materials (SBOM)

### Generating SBOM

```bash
# Node.js - CycloneDX format
npx @cyclonedx/cyclonedx-npm --output-file sbom.json

# .NET
dotnet CycloneDX <solution.sln> -o sbom.xml

# Universal - Syft
syft . -o cyclonedx-json > sbom.json
```

### SBOM in CI

```yaml
# Generate and store SBOM with each release
- name: Generate SBOM
  run: npx @cyclonedx/cyclonedx-npm --output-file sbom.json

- name: Upload SBOM
  uses: actions/upload-artifact@v4
  with:
    name: sbom
    path: sbom.json
```

## Checklist

### Project Setup

- [ ] Lockfile committed to version control
- [ ] Engines field specifies Node/runtime version
- [ ] Private field set for non-published packages
- [ ] License checker configured
- [ ] Dependabot/Renovate configured

### Ongoing Maintenance

- [ ] Weekly security scans running
- [ ] Automated update PRs reviewed
- [ ] License compliance checked
- [ ] Dependency decisions documented
- [ ] Unused dependencies removed

### Security Response

- [ ] Vulnerability SLAs defined
- [ ] Alert notifications configured
- [ ] Escalation path documented
- [ ] SBOM generated for releases

## References

- [npm Security Best Practices](https://docs.npmjs.com/packages-and-modules/securing-your-code)
- [Snyk Best Practices](https://snyk.io/learn/open-source-security/)
- [OWASP Dependency Check](https://owasp.org/www-project-dependency-check/)
- [License Compatibility](https://choosealicense.com/appendix/)
- [CycloneDX SBOM](https://cyclonedx.org/)
- [Dependabot Documentation](https://docs.github.com/en/code-security/dependabot)
