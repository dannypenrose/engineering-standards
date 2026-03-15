# Agentic Coding Standards

> Platform-agnostic standards for working with AI coding agents. Defines the principles, workflows, and guardrails that apply regardless of which agent is used -- Claude Code, Codex, Gemini, or any future agentic coding tool.

## Purpose

AI coding agents are increasingly capable, but they are probabilistic systems with inherent limitations. Without structured guidance, they will invent patterns, ignore conventions, introduce subtle bugs, and produce code that compiles but doesn't conform to your engineering standards.

This document defines the approach for using AI coding agents effectively: how to provide them with structured context, enforce standards automatically through skills, and maintain human oversight at every stage.

## Core Principles

1. **Standards-driven development** -- AI agents must operate within the boundaries of your engineering standards, not their training data's general conventions
2. **Enforce, don't instruct** -- Embedding standards into agent skills is more reliable than repeating rules in every conversation
3. **Human review is mandatory** -- AI output is a draft, never a final product, regardless of how good the agent is
4. **Context determines quality** -- The quality of AI output is directly proportional to the quality of context provided
5. **Platform-agnostic thinking** -- Good practices for AI-assisted development apply across all agents and will outlast any individual tool

---

## Standards Enforcement Through Skills

The most effective way to ensure AI agents follow your engineering standards is to encode those standards into reusable skills that activate automatically based on context.

### The Problem

Without structured enforcement, AI agents:

- Apply generic conventions from their training data instead of your project's specific standards
- Produce inconsistent code across sessions (no memory of previous decisions)
- Skip checks that aren't explicitly requested every time
- Invent patterns rather than following established ones

### The Solution: Skills-Based Enforcement

Skills are instruction files that an agent loads automatically when it detects relevant context. They bridge the gap between your documented standards and the agent's behaviour by:

1. **Detecting the tech stack** -- Examining project files to determine which standards apply
2. **Fetching the relevant standard** -- Loading the actual standard document, not relying on the agent's training data
3. **Applying rules silently** -- Following conventions without lecturing or dumping rules into the conversation
4. **Citing when correcting** -- Referencing specific standard sections when the developer's approach conflicts

```
Developer works on API endpoint
        |
        v
Skill detects NestJS project
        |
        v
Skill fetches TypeScript API design standard
        |
        v
Agent writes code following the standard
        |
        v
Developer reviews output against the standard
```

### Skill Architecture

A well-designed skill follows this structure:

| Phase | What It Does | Why It Matters |
|-------|-------------|---------------|
| **Detection** | Identifies the tech stack from project markers | Ensures the correct standard is applied |
| **Fetch** | Loads the authoritative standard document | Prevents the agent from relying on stale training data |
| **Apply** | Follows the standard during code generation | Produces consistent, standards-compliant output |
| **Cite** | References specific rules when correcting | Gives developers confidence in the correction |

### Reference Implementation

A reference implementation of this approach is available:

- **[Agent Skills](https://github.com/dannypenrose/agent-skills)** -- 8 Claude Code skills covering API design, coding standards, database and migrations, deployment, observability, platform-specific patterns, security, and testing
- **[Engineering Standards](https://github.com/dannypenrose/engineering-standards)** -- 50 standards documents that the skills reference as their source of truth

This pattern is not Claude-specific. Any agentic coding tool that supports custom instructions or skill-like mechanisms can implement the same approach.

---

## Context Architecture

Every AI coding agent benefits from structured, persistent context. The specific file names vary by platform, but the pattern is universal.

### Required Context Layers

| Layer | Purpose | Example |
|-------|---------|---------|
| **Project context** | Architecture, tech stack, conventions, key commands | `CLAUDE.md`, `.cursorrules`, `codex.md`, `.github/copilot-instructions.md` |
| **File index** | Quick reference for locating files by feature | Feature index, codebase map |
| **Standards** | Engineering rules the agent must follow | External standards documents fetched by skills |
| **Session context** | Task-specific instructions for the current conversation | Prompts, constraints, references to existing patterns |

### Platform-Specific Context Files

| Platform | Primary Context File | Notes |
|----------|---------------------|-------|
| Claude Code | `CLAUDE.md` | Loaded automatically at session start |
| GitHub Copilot | `.github/copilot-instructions.md` | Repository-level instructions |
| Cursor | `.cursorrules` | Project-level rules |
| Codex | `codex.md`, `AGENTS.md` | Agent instructions |
| Windsurf | `.windsurfrules` | Project configuration |

Regardless of platform, the context file should contain:

- **Quick reference** -- Ports, commands, stack summary
- **Architecture** -- Patterns, conventions, key decisions
- **Quality gates** -- What checks to run and when
- **Documentation rules** -- What to update after changes

### Context Quality Checklist

- [ ] Tech stack and framework versions specified
- [ ] Key patterns and conventions documented
- [ ] File structure overview included
- [ ] Common commands listed (build, test, lint)
- [ ] Things the agent should never do are explicit
- [ ] References to existing code patterns where applicable

---

## Workflow Standards

These workflows apply to any AI coding agent.

### Pre-Implementation

1. **Explore** -- Have the agent investigate the codebase before making changes
2. **Plan** -- Review the agent's proposed approach before any code is written
3. **Scope** -- Define clear boundaries for what should and shouldn't change

### During Implementation

1. **Read before write** -- The agent must read existing files before modifying them
2. **Follow existing patterns** -- New code should match the style and structure of surrounding code
3. **Single focus** -- One logical change at a time, not multiple unrelated modifications
4. **Test as you go** -- Run tests after significant changes, don't batch everything to the end

### Post-Implementation

1. **Review every line** -- AI output must be reviewed by an experienced engineer before merging
2. **Run quality gates** -- Type checking, linting, and tests must pass
3. **Update documentation** -- If the change affects documented behaviour, update the docs
4. **Check for AI failure modes** -- See [AI Code Review](./ai-code-review) for common issues

---

## AI Failure Modes

All AI coding agents share common failure patterns. Understanding these helps you review AI output more effectively.

| Failure Mode | Description | Mitigation |
|---|---|---|
| **Pattern invention** | Creates new utilities or patterns when existing ones should be used | Point the agent to existing code; use skills that enforce project patterns |
| **Over-engineering** | Adds unnecessary abstractions, configurations, or error handling | Constrain scope explicitly; review for unnecessary complexity |
| **Hallucinated APIs** | References functions, packages, or methods that don't exist | Verify all imports resolve; use up-to-date documentation sources |
| **Stale knowledge** | Uses deprecated APIs or outdated framework patterns | Provide framework version in context; use documentation-fetching tools |
| **Shallow error handling** | Catches errors but swallows them or logs without handling | Trace error paths through to user-visible behaviour |
| **Test theatre** | Tests that pass but don't verify meaningful behaviour | Review assertions -- do they test outcomes or just "it ran"? |
| **Context drift** | Later code contradicts earlier decisions within the same session | Review the full changeset holistically, not file-by-file |
| **Confident incorrectness** | Produces plausible but wrong logic with no uncertainty markers | Manually verify business logic and edge cases |

These failure modes are covered in depth in the [AI Code Review](./ai-code-review) standard.

---

## Security Considerations

### Universal Rules (All Platforms)

- **Never share secrets** with AI agents -- no API keys, passwords, tokens, or connection strings in prompts
- **Never commit AI output without review** -- especially for authentication, authorisation, or data access code
- **Validate all AI-generated SQL** -- check for injection vulnerabilities even when using an ORM
- **Check dependency suggestions** -- AI may suggest outdated or vulnerable packages
- **Review security headers and CORS** -- AI often produces overly permissive configurations

### Context File Security

- Context files (CLAUDE.md, .cursorrules, etc.) are committed to the repository -- never include secrets in them
- Use `.env` references and `requireEnv()` patterns instead of actual values
- If a context file references infrastructure, use placeholders, not real hostnames or IPs

---

## Quality Metrics

### What Good AI-Assisted Development Looks Like

- AI output follows project standards without manual correction
- Code reviews catch fewer standards violations over time (skills are improving)
- New features follow existing patterns consistently
- Test coverage is maintained or improved
- Documentation stays current with implementation

### Warning Signs

- Frequent "that's not how we do it" corrections in reviews
- AI-generated code that compiles but doesn't match project architecture
- Increasing technical debt from AI sessions
- Standards being applied inconsistently across sessions
- Developers rubber-stamping AI output without thorough review

---

## Platform-Specific Guides

For detailed platform-specific implementation:

- **Claude Code** -- See [Claude Code Standards](./claude-code-standards) for CLAUDE.md templates, sub-agent orchestration, MCP server integration, and Claude-specific workflows
- **Prompt Engineering** -- See [Prompt Engineering](./prompt-engineering) for writing effective prompts across any platform
- **AI Code Review** -- See [AI Code Review](./ai-code-review) for reviewing AI-generated code regardless of source

---

## Disclaimer

AI coding agents are probabilistic systems. Even with well-crafted skills, comprehensive standards, and structured context, they can misapply rules, skip checks, hallucinate APIs, or produce output that appears correct but deviates in subtle ways. Skills and standards reduce the frequency of these issues -- they do not eliminate them.

**All AI-assisted output must be reviewed by an experienced software engineer before being committed, merged, or deployed.** AI is a productivity tool, not a substitute for professional judgement.
