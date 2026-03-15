# Git Hooks with Husky

This standard defines the approach for implementing Git hooks across all projects using Husky, ensuring consistent quality gates and developer experience.

## Why Husky

Husky provides a modern, maintainable approach to Git hooks:

- **Version controlled**: Hooks live in `.husky/` directory, tracked in Git
- **Team consistency**: All developers get the same hooks automatically
- **Easy setup**: Auto-installs via `prepare` script in package.json
- **Cross-platform**: Works on macOS, Linux, and Windows
- **No manual installation**: No need to manually copy hooks to `.git/hooks/`

## Standard Hook Structure

### Directory Layout

```
project-root/
├── .husky/
│   ├── _/              # Husky internals (auto-generated)
│   ├── pre-commit      # Runs before each commit
│   ├── pre-push        # Runs before pushing to remote
│   └── commit-msg      # Validates commit messages (optional)
├── package.json        # Contains prepare script
└── ...
```

### Package.json Setup

```json
{
  "scripts": {
    "prepare": "husky"
  },
  "devDependencies": {
    "husky": "^9.0.0"
  }
}
```

## Standard Hooks

### Pre-Commit Hook

**Purpose**: Fast checks before committing. Combines a blocking dependency check with a non-blocking documentation reminder.

**Principles**:
- Should complete in under 5 seconds
- Blocking checks: undeclared dependency imports (prevents CI failures)
- Non-blocking warnings: documentation update reminders
- Focus on staged files only

**Standard Implementation**:

```sh
#!/bin/sh

# PRE-COMMIT HOOK
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 1. Checks staged files for undeclared dependency imports (blocking)
# 2. Warns if code was changed without documentation updates (non-blocking)

# Get staged files
STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACMR)

# ── UNDECLARED DEPENDENCY CHECK (blocking) ──────────────────────
# Catches imports of packages not declared in the nearest package.json.
# pnpm hoisting makes undeclared packages work locally but they fail in CI.
HAS_FRONTEND_FILES=false
for file in $STAGED_FILES; do
    case "$file" in
        apps/*/frontend/src/*.ts|apps/*/frontend/src/*.tsx|apps/*/frontend/src/**/*.ts|apps/*/frontend/src/**/*.tsx)
            HAS_FRONTEND_FILES=true
            break
            ;;
    esac
done

if [ "$HAS_FRONTEND_FILES" = "true" ]; then
    node tools/check-frontend-deps.mjs --staged
    if [ $? -ne 0 ]; then
        echo ""
        echo "❌ Undeclared frontend dependencies found. Commit aborted."
        echo "   Either add the dependency or remove the dead import."
        exit 1
    fi
fi

# ── DOCUMENTATION UPDATE REMINDER (non-blocking) ───────────────
CODE_CHANGED=false
DOCS_CHANGED=false

for file in $STAGED_FILES; do
    case "$file" in
        *.ts|*.tsx|*.js|*.jsx|*.py|*.go|*.rs|*.java|*.cs)
            CODE_CHANGED=true
            ;;
    esac
    case "$file" in
        *.md|*.mdx|*CLAUDE.md|*feature-index.md)
            DOCS_CHANGED=true
            ;;
    esac
done

if [ "$CODE_CHANGED" = "true" ] && [ "$DOCS_CHANGED" = "false" ]; then
    echo ""
    echo "⚠️  Documentation Update Reminder"
    echo ""
    echo "Code files were modified but no documentation was updated."
    echo "Consider updating:"
    echo "  - .claude/feature-index.md (if files added/moved/renamed)"
    echo "  - CLAUDE.md (if patterns or structure changed)"
    echo "  - docs/*.mdx (if features or APIs changed)"
    echo ""
fi

exit 0
```

### Pre-Push Hook

**Purpose**: Comprehensive quality checks before code reaches remote.

**Principles**:
- Can take longer (30-60 seconds acceptable)
- Should block on failures
- Use affected-only strategy in monorepos
- Run type-check, lint, and build

**Standard Implementation (Monorepo)**:

```sh
#!/bin/sh

# AFFECTED-ONLY PRE-PUSH HOOK
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# Only checks apps that have changes in this push.
# This is the pattern used by big tech companies (Google, Meta, Vercel)
# to avoid blocking work on App A due to issues in App B.

echo "🚀 Running affected-only preflight checks..."
echo ""

# Get the remote and branch we're pushing to
remote="$1"
url="$2"

# Read stdin for refs being pushed
while read local_ref local_sha remote_ref remote_sha; do
    # If remote_sha is all zeros, this is a new branch - compare to main
    if [ "$remote_sha" = "0000000000000000000000000000000000000000" ]; then
        remote_sha=$(git rev-parse origin/main 2>/dev/null || echo "HEAD~10")
    fi

    # Find which apps have changes
    CHANGED_FILES=$(git diff --name-only "$remote_sha" "$local_sha" 2>/dev/null || git diff --name-only HEAD~5)

    # Extract unique app names from changed files
    CHANGED_APPS=$(echo "$CHANGED_FILES" | grep "^apps/" | cut -d'/' -f2 | sort -u)
    CHANGED_PACKAGES=$(echo "$CHANGED_FILES" | grep "^packages/" | cut -d'/' -f2 | sort -u)

    if [ -z "$CHANGED_APPS" ] && [ -z "$CHANGED_PACKAGES" ]; then
        echo "📦 No app/package changes detected - running quick checks only"
        echo ""

        # Just run lint and type-check (fast)
        pnpm turbo run lint type-check --output-logs=errors-only
        if [ $? -ne 0 ]; then
            echo ""
            echo "❌ Quick checks failed. Push aborted."
            exit 1
        fi
    else
        echo "📦 Changes detected in:"
        for app in $CHANGED_APPS; do
            echo "   - apps/$app"
        done
        for pkg in $CHANGED_PACKAGES; do
            echo "   - packages/$pkg"
        done
        echo ""

        # Build Turbo filter for affected apps
        FILTER=""
        for app in $CHANGED_APPS; do
            # Check if it's a split structure (frontend/backend) or flat
            if [ -d "apps/$app/frontend" ]; then
                FILTER="$FILTER --filter=@org/${app}-frontend... --filter=@org/${app}-backend..."
            else
                FILTER="$FILTER --filter=@org/${app}..."
            fi
        done

        # Also add changed packages
        for pkg in $CHANGED_PACKAGES; do
            FILTER="$FILTER --filter=@org/${pkg}..."
        done

        # Check backend dependencies are declared
        echo "🔍 Checking backend dependencies..."
        node tools/check-backend-deps.mjs
        if [ $? -ne 0 ]; then
            echo "❌ Undeclared backend dependencies found. Push aborted."
            exit 1
        fi

        # Check frontend dependencies are declared
        echo "🔍 Checking frontend dependencies..."
        node tools/check-frontend-deps.mjs
        if [ $? -ne 0 ]; then
            echo "❌ Undeclared frontend dependencies found. Push aborted."
            exit 1
        fi

        echo "🔍 Running checks for affected packages..."
        echo ""

        # Run type-check and lint for affected packages
        pnpm turbo run type-check lint $FILTER --output-logs=errors-only
        if [ $? -ne 0 ]; then
            echo ""
            echo "❌ Affected package checks failed. Push aborted."
            echo "   Tip: Run 'pnpm preflight <app-name>' to see full details"
            exit 1
        fi

        # Run build for affected packages (skip if SKIP_BUILD is set)
        if [ "$SKIP_BUILD" != "true" ]; then
            echo ""
            echo "🏗️  Building affected packages..."
            pnpm turbo run build $FILTER --output-logs=errors-only
            if [ $? -ne 0 ]; then
                echo ""
                echo "❌ Build failed. Push aborted."
                echo "   Tip: Set SKIP_BUILD=true to skip builds in pre-push"
                exit 1
            fi

            # Validate standalone bundles after successful build (smoke test)
            # Uses cp -rL to dereference symlinks, simulating the CI artifact pipeline
            # where pnpm's .pnpm symlinks break after zip/upload
            for app in $CHANGED_APPS; do
                if [ -d "apps/$app/frontend" ]; then
                    APP_DIR="apps/$app/frontend"
                else
                    APP_DIR="apps/$app"
                fi

                if grep -q "output.*standalone" "$APP_DIR/next.config."* 2>/dev/null; then
                    STANDALONE="$APP_DIR/.next/standalone"
                    if [ ! -f "$STANDALONE/$APP_DIR/server.js" ]; then
                        echo "❌ Standalone bundle missing server.js for $app"
                        exit 1
                    fi
                    # Smoke test: copy with dereferenced symlinks and boot server.js
                    SMOKE_DIR=$(mktemp -d)
                    cp -rL "$STANDALONE/." "$SMOKE_DIR/"
                    cp -r "$APP_DIR/.next/static" "$SMOKE_DIR/$APP_DIR/.next/static"
                    PORT=9876 timeout 4 node "$SMOKE_DIR/$APP_DIR/server.js" &
                    SMOKE_PID=$!
                    sleep 2
                    if ! kill -0 $SMOKE_PID 2>/dev/null; then
                        echo "❌ Standalone smoke test failed for $app — missing dependency"
                        echo "   Add it as explicit dep + outputFileTracingIncludes in next.config.js"
                        rm -rf "$SMOKE_DIR"
                        exit 1
                    fi
                    kill $SMOKE_PID 2>/dev/null; wait $SMOKE_PID 2>/dev/null || true
                    rm -rf "$SMOKE_DIR"
                    echo "✓ Standalone smoke test passed for $app"
                fi
            done
        fi
    fi
done

echo ""
echo "✅ Affected-only checks passed. Proceeding with push..."
```

### Commit-Msg Hook (Optional)

**Purpose**: Validate commit message format.

```sh
#!/bin/sh

# Validate commit message follows conventional commits
commit_msg=$(cat "$1")

# Pattern: type(scope): description
pattern="^(feat|fix|docs|style|refactor|test|chore|build|ci|perf|revert)(\(.+\))?: .{1,72}"

if ! echo "$commit_msg" | grep -qE "$pattern"; then
    echo "❌ Invalid commit message format"
    echo ""
    echo "Expected: type(scope): description"
    echo "Types: feat, fix, docs, style, refactor, test, chore, build, ci, perf, revert"
    echo ""
    echo "Examples:"
    echo "  feat(auth): add OAuth2 login"
    echo "  fix: resolve memory leak in worker"
    echo "  docs(api): update endpoint documentation"
    exit 1
fi
```

## Installation

### New Project Setup

```bash
# Install Husky
pnpm add -D husky

# Initialize Husky
pnpm exec husky init

# Create hooks
echo '#!/bin/sh\n# Your hook content here' > .husky/pre-commit
echo '#!/bin/sh\n# Your hook content here' > .husky/pre-push

# Make executable
chmod +x .husky/pre-commit .husky/pre-push
```

### Team Member Setup

After cloning, hooks are installed automatically via the `prepare` script:

```bash
pnpm install  # Triggers "prepare" which runs "husky"
```

## Best Practices

### Do

- Keep pre-commit hooks fast (< 5 seconds)
- Use affected-only checks in monorepos
- Provide clear error messages with remediation steps
- Allow escape hatches (`--no-verify`) for emergencies
- Document skip environment variables (`SKIP_BUILD=true`)

### Don't

- Run full test suites in pre-commit
- Block commits for warnings (use pre-push instead)
- Require network access in pre-commit
- Silently fail - always provide feedback

## Escape Hatches

For emergency commits that need to bypass hooks:

```bash
# Skip all hooks
git commit --no-verify -m "emergency fix"
git push --no-verify

# Skip specific checks
SKIP_BUILD=true git push
```

**Note**: Use sparingly and document why hooks were skipped.

## Monorepo Considerations

### Affected-Only Strategy

In monorepos, running checks on all packages is slow and unnecessary. The affected-only strategy:

1. Detects which packages changed
2. Runs checks only on those packages and their dependents
3. Uses Turborepo's `--filter` for efficient targeting

### Package Detection

```sh
# Extract changed apps
CHANGED_APPS=$(git diff --name-only "$base_sha" "$head_sha" | grep "^apps/" | cut -d'/' -f2 | sort -u)

# Extract changed packages
CHANGED_PACKAGES=$(git diff --name-only "$base_sha" "$head_sha" | grep "^packages/" | cut -d'/' -f2 | sort -u)
```

### Turborepo Integration

```sh
# Run checks on affected packages and dependents
pnpm turbo run type-check lint --filter=@org/changed-package...
```

The `...` suffix includes all packages that depend on the changed package.

## Troubleshooting

### Hooks Not Running

1. Check hooks are executable: `ls -la .husky/`
2. Reinstall Husky: `pnpm prepare`
3. Check Git version: `git --version` (requires 2.9+)

### Hooks Running Slowly

1. Profile hook execution time
2. Move slow checks to pre-push
3. Implement affected-only strategy
4. Use caching (Turborepo handles this)

### Windows Issues

Use Git Bash or WSL. Native Windows cmd may have path issues.

## Related Standards

- [CI Preflight Strategy](/standards/quality/ci-preflight-strategy) - CI/CD integration
- [Testing Strategy](/standards/quality/testing-strategy) - What to test in hooks
- [Code Review Guidelines](/standards/quality/code-review-guidelines) - Pre-merge checks
