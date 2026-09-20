# Shell Scripting Standards

> Authoritative standards for Bash and POSIX shell scripts: automation, backups, deployment and developer tooling.

## Purpose

Shell scripts run unattended, often as the last line of defence (backups, deployments, cleanup). They fail differently from application code: an unquoted variable or an unchecked exit status does not raise an exception, it silently does the wrong thing to a filesystem. These standards target that failure mode.

Use this standard for any `.sh` file. PowerShell has its own conventions and is out of scope.

## Script Structure

Every script opens the same way, so a reader knows within five lines what it does and how it behaves on error.

```bash
#!/bin/bash

# =============================================================================
# Backup Orchestrator
# Runs each backup script in turn, captures output to a log, and prints a
# grouped summary. Exits non-zero when any script fails.
# =============================================================================

set -uo pipefail

# ── Configuration ────────────────────────────────────
readonly BACKUP_ROOT="${HOME}/backups"
readonly MAX_AGE_DAYS=7
```

Order: shebang, header comment explaining purpose and exit behaviour, `set` options, then configuration constants. Functions next, then the main flow. Keep configuration at the top where it can be changed without reading the logic.

### Shebang

Use `#!/bin/bash` when the script uses Bash features (arrays, `[[ ]]`, `local`). Use `#!/usr/bin/env bash` when the script must run on systems where Bash is not at `/bin/bash`, such as NixOS or a Homebrew-first macOS setup. Use `#!/bin/sh` only for genuinely POSIX scripts, and then do not use Bash features.

### Error handling options

| Option | Effect | When to use |
|---|---|---|
| `set -e` | Exit on any unchecked non-zero status | Short, linear scripts |
| `set -u` | Error on undefined variable | Always |
| `set -o pipefail` | A pipeline fails if any stage fails | Always |

`set -e` is not a safety net. It does not fire inside `if` conditions, `&&` chains, or command substitutions, which is precisely where mistakes hide. For a script that must continue past individual failures and report at the end (an orchestrator, a multi-target backup), prefer `set -uo pipefail` and check exit statuses explicitly:

```bash
if ! rsync -a "$src/" "$dest/"; then
    echo "❌ sync failed for $src"
    FAILED=$((FAILED + 1))
fi
```

## Quoting and Variables

Quote every expansion. An unquoted variable containing spaces becomes multiple arguments, which turns `rm -rf $DIR` into a catastrophe when `DIR` is empty or contains a space.

```bash
# Correct
rm -rf "${BACKUP_DIR:?BACKUP_DIR is not set}"
cp "$src" "$dest"

# Wrong
rm -rf $BACKUP_DIR
cp $src $dest
```

Use `${VAR:?message}` for any variable whose emptiness would be destructive. It aborts with a clear message rather than expanding to nothing.

| Form | Meaning |
|---|---|
| `${VAR:-default}` | Use `default` when unset or empty |
| `${VAR:?message}` | Abort with `message` when unset or empty |
| `${VAR:+value}` | Use `value` only when `VAR` is set |

Declare constants with `readonly` and function-local variables with `local`. Without `local`, every variable is global, and a helper silently overwrites its caller's state.

```bash
process_repo() {
    local repo="$1"
    local status
    status=$(git -C "$repo" status --porcelain)
    ...
}
```

Assign and declare on separate lines when capturing command output. `local status=$(cmd)` masks the exit status of `cmd`, because `local` itself succeeds.

## Never Suppress What You Do Not Check

Redirecting stderr to `/dev/null` and chaining with `&&` is the most common way a shell script lies about its own success:

```bash
# Wrong: a failed mv prints nothing and the script continues
mv "$src" "$dest" 2>/dev/null && echo "moved"
```

If the move fails, there is no output, no error and no clue. Suppress stderr only when the error is expected and handled, and say so:

```bash
# Expected to fail when the item does not exist, which is fine here
rm -f "$tmpfile" 2>/dev/null

# Otherwise: let errors surface, and check the status
if ! mv "$src" "$dest"; then
    echo "❌ failed to move $src to $dest"
    exit 1
fi
```

## Verify Destructive Operations

Any operation that moves, deletes or overwrites data must be verified afterwards, not assumed. The pattern is copy, verify, then remove:

```bash
# Safer than mv for anything that matters, especially across filesystems
src_count=$(find "$src" -type f | wc -l | tr -d ' ')
cp -R "$src" "$dest"
dst_count=$(find "$dest" -type f | wc -l | tr -d ' ')

if [ "$src_count" -eq "$dst_count" ] && [ "$dst_count" -gt 0 ]; then
    rm -rf "${src:?}"
else
    echo "❌ copy incomplete: $src_count source, $dst_count destination; source kept"
    exit 1
fi
```

This matters most on network and synced filesystems. On a Google Drive, OneDrive or Dropbox mount, `mv` is not atomic and may route content to the provider's trash rather than the destination, reporting success either way.

## Network and Synced Filesystems

Cloud storage mounts behave like filesystems until they do not. Scripts that write to them need three extra habits:

- **Do not trust `du`.** Files that exist but are not downloaded locally (Files On-Demand, streamed content) report as zero bytes. Count files with `find` when the answer matters.
- **Do not use `mv`.** Use copy, verify, remove, as above.
- **Do not measure freshness by file mtime.** Tools that preserve timestamps (`rsync -a`, `cp -p`) leave copied files carrying the source's dates, so a backup written seconds ago can read as months old. Write a marker file on every run instead:

```bash
date '+%F %T' > "$DEST/.backup-stamp"
```

Check the marker, not the content. It is the only thing that distinguishes "ran, nothing to copy" from "stopped running".

## Functions

Keep functions short and single-purpose. Document the contract when it is not obvious from the name:

```bash
# Age in days since a path was last written, or 99999 when absent.
# Prefers a .backup-stamp because rsync -a preserves source timestamps.
age_days() {
    local path="$1"
    ...
}
```

Return status codes for success and failure; use `echo` for the single value a caller captures. Do not mix progress output into a function whose output is captured, or the progress lines end up in the variable.

## Safe Iteration

Word splitting on filenames with spaces is a classic source of damage. Never parse `ls`. Use NUL-separated output where the input is arbitrary:

```bash
# Correct
while IFS= read -r -d '' file; do
    process "$file"
done < <(find "$dir" -type f -print0)

# Correct for line-based input
while IFS= read -r line; do
    ...
done < "$file"

# Wrong
for file in $(ls "$dir"); do
```

Use `IFS=` and `-r` on every `read`: `IFS=` preserves leading and trailing whitespace, `-r` stops backslashes being interpreted.

## Exit Codes

Reserve meaningful codes and document them in the header:

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | Failure, the script could not do its job |
| `2` | Completed with warnings that need attention |

A script whose job is to report (an audit, a health check) should exit non-zero when it *finds* problems, and the caller must distinguish that from the script itself failing. State which is which in the header comment.

## Temporary Files

Create temporary files with `mktemp` and always clean up with a trap, so an interrupted run does not leave litter:

```bash
tmp=$(mktemp)
trap 'rm -f "$tmp"' EXIT
```

Build output in a temporary file and move it into place at the end. A half-written file that replaces a good one is worse than no new file at all.

## Logging Output

Write progress to stdout and problems to stderr, so a caller can separate them:

```bash
echo "Backing up ${name}..."
echo "❌ ${name}: destination not mounted" >&2
```

Prefix a machine-readable summary line when another script parses the output, and note in a comment that something depends on it:

```bash
# Machine-parseable line consumed by backups.sh (extract_detail):
echo "Summary: $TOTAL_FILES files, $HUMAN_TOTAL across $REPOS repos"
```

## Secrets

Never hardcode credentials. Read them from the environment, a gitignored `.env`, or a secret manager:

```bash
# Correct
DB_PASSWORD="${DB_PASSWORD:?DB_PASSWORD is not set}"

# Wrong
DB_PASSWORD="hunter2"
```

Credentials passed on a command line are visible in `ps` to every user on the machine. Prefer a config file with `0600` permissions, a environment variable, or the tool's own credential file. For MySQL, use `--defaults-extra-file` rather than `-p"$PASSWORD"`.

Never write a private key, token or password into cloud storage in plaintext, including as a side effect of backing up a directory that contains one.

## Validation and Linting

Run [ShellCheck](https://www.shellcheck.net/) on every script; it catches the quoting and word-splitting classes above automatically.

```bash
brew install shellcheck
shellcheck script.sh
```

Check syntax without executing during development:

```bash
bash -n script.sh
```

Add ShellCheck to CI and to pre-commit hooks for repositories containing shell scripts (see [Git Hooks (Husky)](/standards/development/git-hooks-husky)). Suppress a rule inline only with a comment explaining why:

```bash
# shellcheck disable=SC2053  # glob match against $glob is intended here
[[ "$base" == $glob ]]
```

## Prose Style

Comments, log lines and error messages follow the same rules as all other prose: UK English, and never an em dash. Explain *why*, not *what*: the code already says what it does.

```bash
# Correct: explains the non-obvious constraint
# Fetch in parallel; a single `op item get` costs about ten seconds, so a
# serial loop over a few dozen items turns a fast audit into a slow one.

# Wrong: restates the code
# Loop over the items and get each one
```

## Related Standards

- [Git Standards](/standards/development/git-standards): commit conventions for script changes
- [Git Hooks (Husky)](/standards/development/git-hooks-husky): wiring ShellCheck into pre-commit
- [Environment Management](/standards/development/environment-management): `.env` conventions and secret handling
- [Backup & Disaster Recovery](/standards/reliability/backup-disaster-recovery): what backup scripts must guarantee
- [Infrastructure Security](/standards/governance/infrastructure-security): credential handling on servers
