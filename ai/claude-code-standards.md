# Claude Code Standards

> This is the Claude Code-specific implementation guide. For platform-agnostic principles that apply to all AI coding agents, see [Agentic Coding Standards](./agentic-coding-standards).

This standard defines best practices for using Claude Code across all projects, ensuring consistent AI-assisted development workflows aligned with enterprise engineering practices.

## Core Principles

1. **Context is King**: Provide Claude with structured context to minimize exploration time
2. **Document as You Go**: Maintain documentation in real-time, not as an afterthought
3. **Orchestrate Intelligently**: Use sub-agents and parallel execution for complex tasks
4. **Quality Gates**: Integrate Claude workflows with existing CI/CD and review processes
5. **Standardize Knowledge**: Maintain consistent CLAUDE.md patterns across projects

---

## Project Structure Standards

### Required Files

Every project should include these Claude-specific files:

```
project-root/
├── CLAUDE.md                    # Project-level guidance
├── .claude/
│   └── feature-index.md         # File location reference
├── apps/
│   └── my-app/
│       ├── CLAUDE.md            # App-specific guidance
│       └── .claude/
│           └── feature-index.md # App-level file index
└── docs/                        # Project documentation
```

### CLAUDE.md Standards

The CLAUDE.md file is Claude's primary context source. The template below is designed for **maximum information density** - every line serves a purpose. Adapt sections to your project's needs.

~~~~markdown
# [Project Name] - CLAUDE.md

> **Feature Index**: [.claude/feature-index.md](.claude/feature-index.md) | **Docs**: [docs/](docs/)

---

## Quick Reference

| Resource | Location |
|----------|----------|
| Frontend | `localhost:3000` (always running) |
| Backend  | `localhost:3001` (always running) |
| Tests    | `pnpm test` |
| Build    | `pnpm build` |
| Lint     | `pnpm lint` |

**NEVER start dev servers** - they run in background.

---

## Architecture

[One-paragraph summary: framework, patterns, key decisions]

```
project/
├── src/           # Source code
├── tests/         # Test files
├── docs/          # Documentation
└── .claude/       # Claude context files
```

**Stack**: [e.g., Next.js 14 + TypeScript + Prisma + PostgreSQL]

**Key Patterns**:
- [Pattern 1]: [Brief description]
- [Pattern 2]: [Brief description]

---

## Sub-Agent Strategy

Deploy sub-agents when: **>10K tokens** OR **>5 files** OR **complexity >0.7**

| Task Type | Agent | When to Use |
|-----------|-------|-------------|
| Frontend  | `nextjs-frontend-architect` | React/TS, components, performance |
| Backend   | `dotnet-senior-engineer` | .NET APIs, enterprise patterns |
| Database  | `entity-framework-specialist` | EF Core, migrations, queries |
| Research  | `Explore` | File discovery, codebase questions |
| Planning  | `Plan` | Architecture, implementation design |
| Quality   | `quality-engineer` | Testing, edge cases, QA |

**Orchestration patterns**:
- **Sequential**: Frontend → Backend → Database (hand off context)
- **Parallel**: Independent analyses in single message (multiple Task calls)
- **Specialist-led**: Domain expert coordinates secondary agents

---

## MCP Servers

**Priority**: Context7 (docs) → Sequential (analysis) → Playwright (E2E)

**Fallback**: MCP unavailable → WebSearch → Manual implementation

---

## Quality Gates

**Before coding**: Read files first. Plan with TodoWrite. Validate approach.

**During coding**: ONE task in_progress. Mark complete immediately. Test after changes.

**After coding**:
```bash
pnpm type-check && pnpm lint && pnpm test
```

---

## Documentation Maintenance (CRITICAL)

**Update docs BEFORE marking any task complete.** This is not optional.

After completing each requested task, update relevant documentation:

1. **Feature Index** (`.claude/feature-index.md`)
   - Update if files were added, moved, or renamed
   - Add new feature areas when created

2. **CLAUDE.md**
   - Update if patterns, commands, or structure changed
   - Document new conventions or decisions

3. **Project Docs** (`docs/` folder)
   - Update if features or APIs changed
   - Keep examples current with implementation

**Workflow**: Complete task → Update docs → Run quality checks → Mark complete

---

## Task Completion Checklist

Before marking ANY task as complete:

- [ ] Code changes tested and working
- [ ] Feature index updated (if files changed)
- [ ] CLAUDE.md updated (if patterns changed)
- [ ] Docs updated (if APIs/features changed)
- [ ] Quality checks passing (`pnpm type-check && pnpm lint && pnpm test`)
- [ ] No secrets or sensitive data in commits

---

## Project-Specific Notes

[Add critical project context: gotchas, conventions, architectural decisions]
~~~~

#### Template Customization Guide

| Section | Required | Customize For |
|---------|----------|---------------|
| Quick Reference | ✅ | Ports, commands specific to project |
| Architecture | ✅ | Tech stack, folder structure, patterns |
| Sub-Agent Strategy | ⚡ | Only include agents relevant to project |
| MCP Servers | ⚡ | Remove if not using MCPs |
| Quality Gates | ✅ | Project-specific checks |
| Documentation Maintenance | ✅ | **Always include** - critical workflow |
| Task Completion Checklist | ✅ | Project-specific validation steps |
| Project-Specific Notes | ✅ | Gotchas, conventions, tribal knowledge |

**Legend**: ✅ = Always include | ⚡ = Include if applicable

### Feature Index Standards

The feature index accelerates file discovery:

```markdown
# Feature Index

Quick reference for locating files by feature area.

## Authentication
- `src/auth/login.tsx` - Login page component
- `src/auth/auth.service.ts` - Auth service
- `lib/auth.ts` - Auth utilities

## Dashboard
- `src/dashboard/page.tsx` - Main dashboard
- `src/components/dashboard/` - Dashboard components

## API
- `src/api/users/route.ts` - Users endpoint
- `src/api/auth/route.ts` - Auth endpoints
```

**Update triggers**:
- Files added, moved, or renamed
- New feature areas created
- Major refactoring completed

---

## Sub-Agent Orchestration

### When to Use Sub-Agents

Deploy sub-agents when:

| Criteria | Threshold |
|----------|-----------|
| Token count for analysis | >10K tokens |
| Files to modify | >5 files |
| Complexity score | >0.7 |
| Domain expertise needed | Specialized knowledge |

### Available Specialists

The following agents are available globally. This list is not exhaustive—new agents can and should be created for project-specific needs.

#### Core Agents

| Agent | Use Case |
|-------|----------|
| `general-purpose` | Complex multi-step tasks, comprehensive searches, cross-domain analysis |
| `Explore` | Fast codebase exploration, file discovery, pattern searching |
| `Plan` | Architecture design, implementation planning, trade-off analysis |

#### Frontend & UI Specialists

| Agent | Use Case |
|-------|----------|
| `nextjs-frontend-architect` | Next.js, React, TypeScript, performance optimization, component architecture |
| `frontend-ui-engineer` | HTML/CSS/JavaScript, Tailwind CSS, responsive design, accessibility |
| `frontend-architect` | Accessible, performant UIs, modern frameworks, user experience |
| `material-ui-specialist` | MUI theming, custom components, design systems, styling optimization |
| `saas-ux-specialist` | SaaS application UX, user flows, wireframes, conversion optimization |

#### Backend & Architecture Specialists

| Agent | Use Case |
|-------|----------|
| `dotnet-senior-engineer` | .NET architecture, ASP.NET Core, performance, enterprise patterns |
| `entity-framework-specialist` | EF Core, migrations, LINQ optimization, database design |
| `odata-implementation-expert` | OData endpoints, query building, filtering/sorting/pagination |
| `php-backend-specialist` | WordPress, Laravel, server-side PHP architecture |
| `backend-architect` | Reliable backend systems, data integrity, security, fault tolerance |
| `system-architect` | Scalable architecture, maintainability, long-term technical decisions |
| `python-expert` | Production-ready Python, SOLID principles, modern best practices |

#### Quality & Analysis Specialists

| Agent | Use Case |
|-------|----------|
| `quality-engineer` | Testing strategies, edge case detection, comprehensive QA |
| `performance-engineer` | Measurement-driven analysis, bottleneck elimination |
| `security-engineer` | Vulnerability identification, compliance, security best practices |
| `refactoring-expert` | Code quality, technical debt reduction, clean code principles |
| `root-cause-analyst` | Problem investigation, evidence-based analysis, hypothesis testing |

#### Documentation & Learning

| Agent | Use Case |
|-------|----------|
| `technical-writer` | Clear documentation, audience-tailored content, usability focus |
| `learning-guide` | Programming concepts, progressive learning, practical examples |
| `socratic-mentor` | Discovery learning through strategic questioning |

#### Research & Planning

| Agent | Use Case |
|-------|----------|
| `deep-research-agent` | Comprehensive research, adaptive strategies, intelligent exploration |
| `requirements-analyst` | Transform ambiguous ideas into concrete specifications |
| `web-feature-analyst` | Analyze existing web features for documentation and modernization |

#### DevOps & Infrastructure

| Agent | Use Case |
|-------|----------|
| `devops-architect` | Infrastructure automation, deployment, reliability, observability |

#### Business & Strategy

| Agent | Use Case |
|-------|----------|
| `business-panel-experts` | Multi-expert business strategy synthesis (Christensen, Porter, Drucker, etc.) |
| `uk-workplace-advisor` | UK employment law, HR policies, workplace guidance |

#### Domain-Specific

| Agent | Use Case |
|-------|----------|
| `wow-addon-lua-expert` | World of Warcraft addon development, Lua scripting, WoW API |
| `claude-code-guide` | Claude Code features, hooks, MCP servers, IDE integrations |
| `statusline-setup` | Configure Claude Code status line settings |

### Creating New Agents

**The agent list above is not fixed.** When working on projects that require specialized knowledge not covered by existing agents, create new agents tailored to those needs.

#### When to Create a New Agent

- Existing agents don't cover the domain expertise needed
- A project requires deep specialization (e.g., game development, blockchain, ML/AI)
- Repeated complex tasks would benefit from a dedicated specialist
- Cross-domain expertise is needed that no single agent provides

#### Example: Game Development Project

If starting a game development project, you might need agents like:

```typescript
// Hypothetical game development agents
Task({ subagent_type: "unity-game-architect", ... })      // Unity/C# game architecture
Task({ subagent_type: "unreal-blueprint-expert", ... })   // Unreal Engine blueprints
Task({ subagent_type: "game-physics-specialist", ... })   // Physics systems, collision
Task({ subagent_type: "shader-graphics-engineer", ... })  // HLSL/GLSL, rendering
Task({ subagent_type: "game-ai-specialist", ... })        // NPC behavior, pathfinding
Task({ subagent_type: "multiplayer-networking", ... })    // Netcode, synchronization
```

#### Agent Creation Guidelines

1. **Clear specialization**: Define a focused domain of expertise
2. **Distinct from existing**: Don't duplicate capabilities of existing agents
3. **Documented capabilities**: List specific skills and use cases
4. **Integration patterns**: Define how it coordinates with other agents

### Orchestration Patterns

#### Sequential Chain
Hand off context between specialists:

```
Frontend Architect → Backend Engineer → Database Specialist
     (UI design)        (API impl)        (schema opt)
```

#### Parallel Execution
Independent work streams with integration:

```typescript
// Launch multiple agents simultaneously
Task({ subagent_type: "nextjs-frontend-architect", ... })
Task({ subagent_type: "dotnet-senior-engineer", ... })
Task({ subagent_type: "general-purpose", focus: "testing" })
```

#### Specialist-Led
Primary domain expert coordinates with secondary specialists:

```typescript
// Frontend-led feature development
Task({
  subagent_type: "nextjs-frontend-architect",
  prompt: "Implement dashboard feature. Coordinate with backend
           for API contracts and testing agent for test coverage."
})
```

### Parallel Task Execution

Always parallelize independent operations:

```typescript
// ✅ GOOD: Parallel independent searches
Glob("**/auth/**/*.ts")     // Runs simultaneously
Grep("useAuth")              // Runs simultaneously
Read("src/lib/auth.ts")      // Runs simultaneously

// ❌ BAD: Sequential when parallel is possible
Glob("**/auth/**/*.ts")
// wait...
Grep("useAuth")
// wait...
Read("src/lib/auth.ts")
```

**Parallelization rules**:
- File reads with no dependencies → parallel
- Independent searches → parallel
- Multiple agent launches → parallel (single message, multiple Task calls)
- Dependent operations → sequential with `&&` chaining

---

## Quality Assurance Workflows

### Pre-Implementation Phase

1. **Explore codebase** using Explore agent for understanding
2. **Plan implementation** using Plan agent for design
3. **Create todo list** using TodoWrite for tracking
4. **Validate approach** with user before coding

### Implementation Phase

1. **Read before write**: Always read files before modifying
2. **Single focus**: ONE task in_progress at a time
3. **Immediate completion**: Mark todos complete immediately
4. **Test integration**: Run tests after significant changes

### Post-Implementation Phase

1. **Update documentation**:
   - Feature index if files changed
   - CLAUDE.md if patterns changed
   - Docs folder if APIs changed

2. **Run quality checks**:
   ```bash
   pnpm type-check && pnpm lint && pnpm test
   ```

3. **Review changes**:
   - Git status review
   - Affected package analysis
   - Security consideration

### End-of-Session Checklist

```markdown
## Session End Checklist

- [ ] All todos marked complete or documented as blocked
- [ ] Feature index updated if files added/moved/renamed
- [ ] CLAUDE.md updated if patterns changed
- [ ] Docs updated if features/APIs changed
- [ ] Tests passing for modified code
- [ ] No uncommitted sensitive files
- [ ] Pre-commit hooks passing
```

---

## MCP Server Integration

### Priority Order

1. **Context7**: Library documentation and framework patterns
2. **Sequential**: Complex multi-step analysis and reasoning
3. **Playwright**: Browser automation and E2E testing

### Configuration

MCP servers are configured at two levels:

| Scope | Location | Managed Via | Use Case |
|-------|----------|-------------|----------|
| **User (global)** | `~/.claude.json` | `claude mcp add --scope user` | Servers needed across all projects (Context7, Sequential Thinking, Playwright) |
| **Project** | `.mcp.json` in project root | Manual file or `claude mcp add --scope project` | Project-specific servers (e.g. MSSQL for database projects) |

#### Adding Global Servers (New Machine)

```bash
claude mcp add --scope user context7 -- npx -y @upstash/context7-mcp@latest
claude mcp add --scope user sequential-thinking -- npx -y @modelcontextprotocol/server-sequential-thinking
claude mcp add --scope user playwright -- npx -y @playwright/mcp@latest
```

#### Adding Project Servers

Create `.mcp.json` in the project root:

```json
{
  "mcpServers": {
    "mssql": {
      "command": "node",
      "args": ["/path/to/mssql-mcp-server/server.mjs"]
    }
  }
}
```

### Fallback Strategy

```
Context7 unavailable → WebSearch for docs → Manual implementation
Sequential timeout → Native Claude analysis → Note limitations
MCP unavailable → Standard tools → Manual validation
```

---

## Git Integration

### Pre-Commit Hook Integration

Integrate Claude workflows with Husky hooks:

```sh
# .husky/pre-commit
#!/bin/sh

# Documentation reminder (non-blocking)
# Warns if code changed without doc updates
```

See [Git Hooks (Husky)](/standards/development/git-hooks-husky) for full implementation.

### Commit Standards

When Claude creates commits:

1. Clear, descriptive messages following conventional commits
2. Single logical change per commit
3. Co-Author attribution: `Co-Authored-By: Claude <noreply@anthropic.com>`
4. Never commit secrets or sensitive files

### Branch Strategy

- Feature branches with descriptive names
- PR creation with summary and test plan
- Affected-only checks before push

---

## Performance Optimization

### Token Efficiency

| Technique | Implementation |
|-----------|----------------|
| Feature index | Maintain up-to-date file locations |
| Targeted reads | Read specific files, not entire directories |
| Smart search | Use Glob/Grep over shell commands |
| Batch operations | Combine related tasks |
| Agent delegation | Offload to sub-agents for complex analysis |

### Context Management

- **75% usage**: Consider compressing context
- **Repeated searches**: Cache results within session
- **Large files**: Read specific line ranges when possible

### Thresholds

| Trigger | Condition |
|---------|-----------|
| Sub-agent activation | >10K tokens OR >5 files OR complexity >0.7 |
| Wave mode | complexity ≥0.7 + files >20 + operation_types >2 |
| Loop mode | Auto-detect: polish, refine, enhance keywords |
| Parallel execution | >7 directories OR >50 files |

---

## Security Standards

### Never Commit

- `.env` files with secrets
- API keys or tokens
- Credentials or passwords
- Private keys

### Code Security

- Validate all inputs at system boundaries
- Use parameterized queries (no SQL injection)
- Sanitize outputs (no XSS)
- Follow least privilege principle

### Review Requirements

- Security-sensitive changes require explicit approval
- Authentication/authorization changes need extra scrutiny
- Dependency updates should check for vulnerabilities

---

## Testing Integration

### Test-Driven Workflow

1. **Understand requirements**
2. **Write failing test** (Red)
3. **Implement minimum code** (Green)
4. **Refactor** (Refactor)
5. **Repeat**

### Coverage Standards

| Area | Minimum Coverage |
|------|-----------------|
| Critical paths | 80% |
| New features | 70% |
| Bug fixes | Include regression test |
| Utilities | 90% |

### Test Execution

```bash
# Run affected tests only
pnpm turbo run test --filter=@org/changed-package...

# Full test suite before major changes
pnpm test

# Coverage report
pnpm test:coverage
```

---

## Continuous Learning

### Pattern Recognition

Document successful approaches:

```markdown
## Patterns That Worked

### [Pattern Name]
- **Context**: When this was used
- **Solution**: What was implemented
- **Outcome**: Results achieved
```

### Anti-Patterns

Document what to avoid:

```markdown
## Anti-Patterns

### [Problem Name]
- **What happened**: Description of issue
- **Why it failed**: Root cause
- **Better approach**: Alternative solution
```

### Knowledge Persistence

- Update CLAUDE.md with new patterns
- Add to feature index when discovering file locations
- Document architectural decisions in ADRs

---

## Monorepo Considerations

### Affected-Only Strategy

Only check/build changed packages:

```bash
# Detect changes
CHANGED_APPS=$(git diff --name-only origin/main | grep "^apps/" | cut -d'/' -f2 | sort -u)

# Run checks on affected only
pnpm turbo run test lint --filter=@org/${app}...
```

### Package Consolidation

- Apps consume shared packages (never duplicate)
- Extend shared code when customization needed
- Import from `@org/package-name`, not relative paths

### Cross-Package Changes

1. Identify all affected packages
2. Update in dependency order
3. Run affected-only tests
4. Review downstream impacts

---

## Workflow Templates

### New Feature Workflow

```
1. Explore → Understand existing patterns
2. Plan → Design implementation approach
3. TodoWrite → Create task breakdown
4. Implement → Code with TDD
5. Test → Verify functionality
6. Document → Update docs and indexes
7. Review → Pre-commit checks
8. Commit → Following conventions
```

### Bug Fix Workflow

```
1. Reproduce → Understand the issue
2. Explore → Find relevant code
3. Root cause → Identify the problem
4. Test → Write failing test
5. Fix → Implement minimal fix
6. Verify → Tests pass
7. Document → Update if behavior changed
8. Commit → Include issue reference
```

### Refactoring Workflow

```
1. Assess → Understand current state
2. Plan → Define target state
3. Tests → Ensure coverage exists
4. Refactor → Incremental changes
5. Verify → Tests still pass
6. Document → Update affected docs
7. Review → Check for regressions
```

---

## Related Standards

- [Git Hooks (Husky)](/standards/development/git-hooks-husky) - Pre-commit and pre-push hooks
- [CI Preflight Strategy](/standards/quality/ci-preflight-strategy) - CI/CD integration
- [Testing Strategy](/standards/quality/testing-strategy) - Test patterns and coverage
- [Code Review Guidelines](/standards/quality/code-review-guidelines) - Review processes
- [Coding Standards](/standards/typescript/coding-standards-monorepo) - Code style and patterns
