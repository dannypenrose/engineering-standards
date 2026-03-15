# Git Standards

> Authoritative Git workflow and convention standards for branching, commits, and version control.

## Purpose

Establish consistent Git practices across all projects to ensure clean history, clear communication, and efficient collaboration.

## Commit Message Standards

### Conventional Commits Format

All commits must follow the [Conventional Commits](https://www.conventionalcommits.org/) specification:

```
<type>(<scope>): <subject>

[optional body]

[optional footer(s)]
```

### Commit Types

| Type | Description | Example |
|------|-------------|---------|
| `feat` | New feature | `feat(auth): add OAuth2 login flow` |
| `fix` | Bug fix | `fix(api): resolve null pointer in user lookup` |
| `docs` | Documentation only | `docs(readme): update installation steps` |
| `style` | Code style (formatting, semicolons) | `style: apply prettier formatting` |
| `refactor` | Code change that neither fixes nor adds | `refactor(db): extract query builder` |
| `perf` | Performance improvement | `perf(search): add index for user queries` |
| `test` | Adding or correcting tests | `test(auth): add unit tests for JWT validation` |
| `build` | Build system or dependencies | `build: upgrade webpack to v5` |
| `ci` | CI configuration | `ci: add GitHub Actions workflow` |
| `chore` | Maintenance tasks | `chore: update .gitignore` |
| `revert` | Revert previous commit | `revert: feat(auth): add OAuth2 login flow` |

### Subject Line Rules

1. **Length**: Maximum 72 characters (50 is ideal)
2. **Case**: Use lowercase (except proper nouns)
3. **Tense**: Use imperative mood ("add" not "added" or "adds")
4. **Punctuation**: No period at the end
5. **Content**: Describe *what* the commit does, not *how*

### Good vs Bad Examples

```bash
# ✅ Good commits
feat(cart): add product quantity validation
fix: prevent duplicate form submissions
refactor(auth): extract token refresh logic
docs(api): document rate limiting behavior
perf(search): optimize full-text search query

# ❌ Bad commits
Fixed bug                          # No type, vague
feat: Add new feature.             # Vague, has period
FEAT(AUTH): Added OAuth            # Wrong case, past tense
feat(authentication-service): add the new oauth2 login flow for users  # Too long
WIP                                # Not descriptive
misc changes                       # Completely useless
```

### Scope Guidelines

Scope should identify the affected area:

- **Feature/module**: `auth`, `cart`, `payments`, `search`
- **Layer**: `api`, `ui`, `db`, `config`
- **App (monorepo)**: `web`, `mobile`, `admin`, `backend`

```bash
# Feature scope
feat(auth): implement password reset flow
fix(payments): handle declined card gracefully

# Layer scope
refactor(api): standardize error response format
style(ui): apply design system spacing

# Monorepo app scope
feat(web): add dark mode toggle
fix(mobile): resolve gesture conflict
```

### Commit Body

Use the body for commits that need explanation:

```bash
fix(auth): prevent session fixation attack

The previous implementation reused session IDs after login,
allowing attackers to hijack sessions. Now generates a new
session ID upon successful authentication.

Refs: SECURITY-1234
```

**Body guidelines:**
- Wrap at 72 characters
- Explain *why* the change was made
- Describe *what* problem it solves
- Reference relevant issues/tickets

### Breaking Changes

Mark breaking changes clearly:

```bash
feat(api)!: change user endpoint response format

BREAKING CHANGE: The /api/users endpoint now returns a paginated
response object instead of a raw array. Clients must update to
handle the new { data: [], meta: {} } structure.
```

Or in footer:

```bash
refactor(auth): migrate to JWT tokens

BREAKING CHANGE: Session-based auth is removed. All clients must
implement JWT token refresh.
```

## Branch Naming Conventions

### Format

```
<type>/<ticket>-<short-description>
```

### Branch Types

| Prefix | Purpose | Example |
|--------|---------|---------|
| `feature/` | New features | `feature/AUTH-123-oauth-login` |
| `fix/` | Bug fixes | `fix/BUG-456-null-pointer` |
| `hotfix/` | Production emergency fixes | `hotfix/PROD-789-payment-failure` |
| `refactor/` | Code refactoring | `refactor/TECH-012-extract-service` |
| `docs/` | Documentation | `docs/update-api-reference` |
| `test/` | Test additions | `test/AUTH-123-integration-tests` |
| `chore/` | Maintenance | `chore/upgrade-dependencies` |
| `release/` | Release preparation | `release/v2.1.0` |

### Naming Rules

1. **Lowercase only**: `feature/auth-login` not `Feature/Auth-Login`
2. **Hyphens for spaces**: `add-user-profile` not `add_user_profile`
3. **Include ticket number**: `AUTH-123` when applicable
4. **Keep short but descriptive**: 3-5 words max
5. **No special characters**: Except hyphens and forward slash

### Examples

```bash
# ✅ Good branch names
feature/AUTH-123-oauth-google
fix/BUG-456-cart-total-calculation
hotfix/payment-gateway-timeout
refactor/extract-validation-utils
docs/api-authentication-guide

# ❌ Bad branch names
Feature/OAuth                      # Uppercase, vague
my-feature                         # No type prefix
feature/AUTH-123-add-the-new-oauth-login-flow-with-google-provider  # Too long
fix_bug                            # Underscore, vague
john/testing                       # Personal prefix
```

## Workflow Strategy

### Trunk-Based Development (Recommended)

For mature teams with good CI/CD, trunk-based development provides faster iteration:

```
main ─────●─────●─────●─────●─────●─────●─────●──▶
          │     │     │           │     │
          └──●──┘     └────●──────┘     └──●──┘
        short-lived feature branches (1-2 days)
```

**Rules:**
- Feature branches live maximum 2-3 days
- All code goes through PR review
- Main is always deployable
- Feature flags for incomplete features
- Small, frequent merges

### GitHub Flow (Small Teams)

For smaller teams or projects:

```
main ─────●─────────────●─────────────●──────▶
          │             │             │
          └──feature────┘             │
                        └───feature───┘
```

**Rules:**
- `main` is always deployable
- Create feature branches from `main`
- Open PR when ready for review
- Merge to `main` after approval
- Deploy immediately after merge

### GitFlow (Release Management)

For projects with scheduled releases:

```
main ────●─────────────────────●─────────────▶
         │                     ↑
develop ─●──●──●──●──●──●──────●──●──●──●────▶
            ↑     ↑        ↑
         feature  feature  release/v1.0
```

Use only when:
- Multiple versions in production
- Formal release cycles required
- Regulatory/compliance requirements

## Merge Strategy

### Squash and Merge (Default)

Use squash merge for feature branches:

```bash
# Creates single commit on main
git checkout main
git merge --squash feature/AUTH-123-oauth
git commit -m "feat(auth): add OAuth login flow (#123)"
```

**Benefits:**
- Clean, linear history
- Easy to revert features
- Commit message summarizes entire feature

**When to use:**
- Feature branches with messy history
- Standard feature development
- Most PRs

### Merge Commit (Preserve History)

Use when history is valuable:

```bash
git checkout main
git merge --no-ff feature/complex-refactor
```

**When to use:**
- Complex refactors with meaningful intermediate commits
- Release branches
- When commit history tells a story

### Rebase (Personal Branches)

Use rebase to keep feature branches current:

```bash
# Update feature branch with main
git checkout feature/my-feature
git rebase main
```

**Rules:**
- Never rebase shared/public branches
- Rebase before opening PR
- Resolve conflicts during rebase

## Tagging Standards

### Semantic Versioning

Follow [SemVer](https://semver.org/) for release tags:

```
v<MAJOR>.<MINOR>.<PATCH>[-<prerelease>]
```

| Component | When to Increment | Example |
|-----------|-------------------|---------|
| MAJOR | Breaking changes | `v2.0.0` |
| MINOR | New features (backward compatible) | `v1.1.0` |
| PATCH | Bug fixes | `v1.0.1` |

### Tag Format

```bash
# Production releases
v1.0.0
v1.2.3
v2.0.0

# Pre-releases
v2.0.0-alpha.1
v2.0.0-beta.2
v2.0.0-rc.1

# Build metadata
v1.0.0+build.123
```

### Creating Tags

```bash
# Lightweight tag (not recommended for releases)
git tag v1.0.0

# Annotated tag (recommended)
git tag -a v1.0.0 -m "Release v1.0.0: OAuth integration"

# Push tags
git push origin v1.0.0
git push origin --tags  # Push all tags
```

## Commit Best Practices

### Atomic Commits

Each commit should be:

1. **Self-contained**: Works independently
2. **Single purpose**: One logical change
3. **Buildable**: Code compiles/passes tests
4. **Deployable**: Could go to production alone

```bash
# ❌ Bad: Multiple unrelated changes
git commit -m "fix auth bug, add logging, update deps"

# ✅ Good: Separate commits
git commit -m "fix(auth): validate token expiry"
git commit -m "feat(logging): add structured request logging"
git commit -m "build: update axios to v1.4.0"
```

### Commit Frequency

- Commit early and often (locally)
- Each commit represents a working state
- Squash before merging to main

### What NOT to Commit

- Secrets, credentials, API keys
- Build artifacts (`dist/`, `node_modules/`)
- IDE configurations (`.idea/`, `.vscode/` unless shared)
- OS files (`.DS_Store`, `Thumbs.db`)
- Large binary files (use Git LFS)
- Environment-specific configs (`.env.local`)

## Git Configuration

### Recommended Global Config

```bash
# Identity
git config --global user.name "Your Name"
git config --global user.email "your.email@company.com"

# Default branch
git config --global init.defaultBranch main

# Pull strategy
git config --global pull.rebase true

# Push default
git config --global push.default current

# Auto-setup remote tracking
git config --global push.autoSetupRemote true

# Better diff
git config --global diff.algorithm histogram

# Commit signing (if required)
git config --global commit.gpgsign true
```

### Useful Aliases

```bash
# Shortcuts
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status

# Better log
git config --global alias.lg "log --oneline --graph --decorate"
git config --global alias.last "log -1 HEAD --stat"

# Undo helpers
git config --global alias.unstage "reset HEAD --"
git config --global alias.undo "reset --soft HEAD~1"
```

## Pull Request Guidelines

### PR Title

Follow same format as commits:

```
feat(auth): add OAuth2 login with Google (#123)
```

### PR Size

| Size | Lines | Recommendation |
|------|-------|----------------|
| XS | < 50 | Ideal |
| S | 50-200 | Good |
| M | 200-500 | Maximum |
| L | 500+ | Must split |

### Draft PRs

Use draft PRs for:
- Early feedback requests
- Work in progress
- CI validation before review

## Anti-Patterns to Avoid

### Commit Anti-Patterns

- **WIP commits on main**: Use draft PRs instead
- **Fixup commits**: Squash before merge
- **Merge commits in feature branches**: Rebase instead
- **Force pushing shared branches**: Coordinate with team

### Branch Anti-Patterns

- **Long-lived feature branches**: Merge frequently
- **Direct commits to main**: Always use PRs
- **Personal branch namespaces**: Use type prefixes
- **Abandoned branches**: Delete after merge

## Checklist

### Repository Setup

- [ ] Branch protection on `main`
- [ ] Require PR reviews
- [ ] Require status checks
- [ ] Squash merge as default
- [ ] Delete branches on merge
- [ ] Signed commits (if required)

### Developer Setup

- [ ] Git configured with correct identity
- [ ] SSH keys configured
- [ ] GPG signing (if required)
- [ ] Useful aliases configured
- [ ] Commit template (optional)

### Per-Commit

- [ ] Message follows conventional format
- [ ] Scope is appropriate
- [ ] Subject is imperative mood
- [ ] No secrets or sensitive data
- [ ] Tests pass locally
- [ ] Lint/format checks pass

## References

- [Conventional Commits](https://www.conventionalcommits.org/)
- [Semantic Versioning](https://semver.org/)
- [Git Flow](https://nvie.com/posts/a-successful-git-branching-model/)
- [GitHub Flow](https://docs.github.com/en/get-started/quickstart/github-flow)
- [Trunk Based Development](https://trunkbaseddevelopment.com/)
- [Google Engineering Practices](https://google.github.io/eng-practices/)
- [Angular Commit Guidelines](https://github.com/angular/angular/blob/main/CONTRIBUTING.md#commit)
