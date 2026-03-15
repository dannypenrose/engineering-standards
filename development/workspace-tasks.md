# VS Code Workspace Task Configuration

> Standard patterns for configuring VS Code workspace tasks across all projects. Ensures consistent developer experience with automatic startup, error visibility, and split terminal layouts.

## Purpose

Define a repeatable structure for `.code-workspace` task configurations so that every project launches its development servers automatically, highlights errors in terminal output, and presents a consistent split-pane layout.

## Core Principles

1. **Auto-start on open** - Development servers launch when the workspace opens
2. **Compound orchestration** - A single parent task coordinates child tasks in parallel
3. **Error visibility** - Terminal output highlights errors, failures, and warnings in red
4. **Problem matcher integration** - Errors surface in the VS Code Problems panel
5. **Split terminal layout** - All server tasks appear side by side in a dedicated group
6. **Build artifact exclusion** - Generated files are excluded from explorer and search

## Workspace File Structure

Every `.code-workspace` file should follow this structure:

```jsonc
{
  "folders": [
    {
      "name": "Project Root",
      "path": "./"
    }
  ],
  "settings": {
    // Build artifact exclusions
  },
  "tasks": {
    "version": "2.0.0",
    "tasks": [
      // Compound parent task
      // Individual server tasks
    ]
  }
}
```

### Folder Configuration

Always provide a descriptive `name` for the root folder:

```jsonc
{
  "folders": [
    {
      "name": "Project Root",
      "path": "./"
    }
  ]
}
```

## Settings: Build Artifact Exclusions

Exclude generated directories from the file explorer and search to reduce noise. Adjust per stack.

### Node.js / Next.js Projects

```jsonc
"settings": {
  "files.exclude": {
    "**/node_modules": true,
    "**/dist": true,
    "**/build": true,
    "**/.next": true,
    "**/.cache": true,
    "**/.turbo": true
  },
  "search.exclude": {
    "**/node_modules": true,
    "**/dist": true,
    "**/build": true,
    "**/.next": true,
    "**/.cache": true,
    "**/.turbo": true
  },
  "explorer.excludeGitIgnore": false
}
```

### .NET Projects (additional entries)

```jsonc
"**/bin": true,
"**/obj": true
```

### Python Projects (additional entries)

```jsonc
"**/__pycache__": true,
"**/.venv": true,
"**/*.egg-info": true
```

## Task Configuration

### Compound Parent Task

The parent task is the only task with `runOn: "folderOpen"`. It orchestrates child tasks in parallel:

```jsonc
{
  "label": "Start Full Stack",
  "dependsOn": ["npm: dev - frontend", "npm: dev - backend"],
  "dependsOrder": "parallel",
  "runOptions": { "runOn": "folderOpen" }
}
```

**Rules:**
- Only the parent task uses `runOptions`
- Child task labels are referenced in `dependsOn`
- `dependsOrder` is always `"parallel"` for dev server tasks
- Label format: `"Start Full Stack"` (consistent across projects)

### Child Task Template

Each server task follows this pattern:

```jsonc
{
  "label": "<runtime>: <script> - <layer>",
  "type": "shell",
  "command": "<start command> 2>&1 | sed -E \"s/(error|fail|exception|rejected|failed)/$(printf '\\033[31m')&$(printf '\\033[0m')/ig\"",
  "options": { "cwd": "${workspaceFolder}/<path>" },
  "isBackground": true,
  "presentation": { "group": "server-split", "panel": "dedicated" },
  "problemMatcher": "<matcher>"
}
```

### Label Convention

Labels follow the pattern `<runtime>: <script> - <layer>`:

| Stack | Label Example |
|-------|---------------|
| Node.js frontend | `npm: dev - frontend` |
| Node.js backend | `npm: dev - backend` |
| .NET backend | `dotnet: watch - backend` |
| Python backend | `python: run - backend` |

### Error Highlighting

All commands pipe through `sed` to colorize error keywords in red:

```bash
<command> 2>&1 | sed -E "s/(error|fail|exception|rejected|failed)/$(printf '\033[31m')&$(printf '\033[0m')/ig"
```

For .NET tasks, also highlight `warn`:

```bash
dotnet watch run 2>&1 | sed -E "s/(error|fail|exception|rejected|failed|warn)/$(printf '\033[31m')&$(printf '\033[0m')/ig"
```

**What this does:**
- `2>&1` merges stderr into stdout so all output is processed
- `sed -E` uses extended regex for grouping
- Matches are case-insensitive (`/ig`)
- Matched text is wrapped in ANSI red (`\033[31m`) and reset (`\033[0m`)

### Problem Matchers

Use the correct built-in problem matcher for each stack:

| Stack | Problem Matcher | What It Catches |
|-------|-----------------|-----------------|
| TypeScript (watch mode) | `$tsc-watch` | TS compilation errors in watch output |
| TypeScript (build) | `$tsc` | TS compilation errors in build output |
| .NET | `$msCompile` | C#/F# compilation errors and warnings |
| ESLint | `$eslint-stylish` | Linting errors and warnings |
| Generic | `[]` | No problem matching (use as fallback) |

### Presentation

All server tasks share the same presentation group for a split layout:

```jsonc
"presentation": {
  "group": "server-split",
  "panel": "dedicated"
}
```

- **`group: "server-split"`** - All tasks with this group appear as split panes in a single terminal panel
- **`panel: "dedicated"`** - Each task gets its own pane within the group
- Do **not** set `"reveal": "always"` — let VS Code manage focus

## Complete Examples

### Node.js Monorepo (Frontend + Backend)

```jsonc
{
  "folders": [{ "name": "Project Root", "path": "./" }],
  "settings": {
    "files.exclude": {
      "**/node_modules": true,
      "**/dist": true,
      "**/build": true,
      "**/.turbo": true,
      "**/.next": true,
      "**/.cache": true
    },
    "search.exclude": {
      "**/node_modules": true,
      "**/dist": true,
      "**/build": true,
      "**/.turbo": true,
      "**/.next": true,
      "**/.cache": true
    },
    "explorer.excludeGitIgnore": false
  },
  "tasks": {
    "version": "2.0.0",
    "tasks": [
      {
        "label": "Start Full Stack",
        "dependsOn": ["npm: dev - frontend", "npm: dev - backend"],
        "dependsOrder": "parallel",
        "runOptions": { "runOn": "folderOpen" }
      },
      {
        "label": "npm: dev - frontend",
        "type": "shell",
        "command": "npm run dev 2>&1 | sed -E \"s/(error|fail|exception|rejected|failed)/$(printf '\\033[31m')&$(printf '\\033[0m')/ig\"",
        "options": { "cwd": "${workspaceFolder}/apps/frontend" },
        "isBackground": true,
        "presentation": { "group": "server-split", "panel": "dedicated" },
        "problemMatcher": "$tsc-watch"
      },
      {
        "label": "npm: dev - backend",
        "type": "shell",
        "command": "npm run dev 2>&1 | sed -E \"s/(error|fail|exception|rejected|failed)/$(printf '\\033[31m')&$(printf '\\033[0m')/ig\"",
        "options": { "cwd": "${workspaceFolder}/apps/backend" },
        "isBackground": true,
        "presentation": { "group": "server-split", "panel": "dedicated" },
        "problemMatcher": "$tsc-watch"
      }
    ]
  }
}
```

### Next.js + .NET (Mixed Stack)

```jsonc
{
  "folders": [{ "name": "Project Root", "path": "./" }],
  "settings": {
    "files.exclude": {
      "**/node_modules": true,
      "**/dist": true,
      "**/build": true,
      "**/.next": true,
      "**/.cache": true,
      "**/bin": true,
      "**/obj": true
    },
    "search.exclude": {
      "**/node_modules": true,
      "**/dist": true,
      "**/build": true,
      "**/.next": true,
      "**/.cache": true,
      "**/bin": true,
      "**/obj": true
    },
    "explorer.excludeGitIgnore": false
  },
  "tasks": {
    "version": "2.0.0",
    "tasks": [
      {
        "label": "Start Full Stack",
        "dependsOn": ["npm: dev - frontend", "dotnet: watch - backend"],
        "dependsOrder": "parallel",
        "runOptions": { "runOn": "folderOpen" }
      },
      {
        "label": "npm: dev - frontend",
        "type": "shell",
        "command": "npm run dev 2>&1 | sed -E \"s/(error|fail|exception|rejected|failed)/$(printf '\\033[31m')&$(printf '\\033[0m')/ig\"",
        "options": { "cwd": "${workspaceFolder}/frontend" },
        "isBackground": true,
        "presentation": { "group": "server-split", "panel": "dedicated" },
        "problemMatcher": "$tsc-watch"
      },
      {
        "label": "dotnet: watch - backend",
        "type": "shell",
        "command": "dotnet watch run 2>&1 | sed -E \"s/(error|fail|exception|rejected|failed|warn)/$(printf '\\033[31m')&$(printf '\\033[0m')/ig\"",
        "options": { "cwd": "${workspaceFolder}/backend" },
        "isBackground": true,
        "presentation": { "group": "server-split", "panel": "dedicated" },
        "problemMatcher": "$msCompile"
      }
    ]
  }
}
```

## First-Time Setup

When opening a workspace with `runOn: "folderOpen"` for the first time, VS Code will prompt:

> **This workspace has tasks that run automatically. Allow?**

Click **"Allow and Run"** to enable automatic task execution. This preference is stored per workspace and only asked once.

To manage this later: `Settings > task.allowAutomaticTasks`

## Checklist

When creating a new workspace task configuration, verify:

- [ ] Root folder has a descriptive `name`
- [ ] Build artifacts excluded from `files.exclude` and `search.exclude`
- [ ] Single compound parent task with `runOn: "folderOpen"`
- [ ] Child tasks referenced in parent's `dependsOn` array
- [ ] `dependsOrder` set to `"parallel"`
- [ ] All child tasks use `sed` error highlighting pipe
- [ ] Correct `problemMatcher` for each stack
- [ ] All server tasks share `group: "server-split"` and `panel: "dedicated"`
- [ ] `cwd` paths are correct relative to `${workspaceFolder}`
- [ ] Labels follow `<runtime>: <script> - <layer>` convention
