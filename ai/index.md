# AI-Assisted Development Standards

Standards and best practices for AI-assisted development, covering agentic coding workflows, standards enforcement through skills, prompt engineering, and code review for AI-generated output. These standards apply across all AI coding agents -- Claude Code, Codex, Gemini, Cursor, and any future agentic tool.

## Purpose

Establish consistent practices for leveraging AI coding agents effectively while maintaining code quality, security, and engineering standards. The key insight: encoding your standards into reusable skills that agents load automatically produces far more consistent results than relying on per-conversation instructions.

## Standards Index

| Standard | Description |
| -------- | ----------- |
| [Agentic Coding Standards](./agentic-coding-standards) | Platform-agnostic principles for working with any AI coding agent -- standards enforcement through skills, context architecture, workflow standards, failure modes, and security |
| [Claude Code Standards](./claude-code-standards) | Claude-specific implementation guide -- CLAUDE.md templates, sub-agent orchestration, MCP integration, and Claude Code workflows |
| [Prompt Engineering](./prompt-engineering) | Writing effective prompts, task decomposition, constraint techniques, and iterative refinement |
| [AI Code Review](./ai-code-review) | Reviewing AI-generated code, AI-specific failure modes, security checks, and pattern conformance |

## Core Principles

1. **Standards-driven development** -- AI agents must operate within the boundaries of your engineering standards, enforced through skills, not ad-hoc instructions
2. **AI as assistant, not authority** -- AI augments developer judgement; all output requires human review
3. **Enforce, don't instruct** -- Embedding standards into agent skills is more reliable than repeating rules in every conversation
4. **Context determines quality** -- Better context produces better results across every platform
5. **Platform-agnostic thinking** -- Good practices for AI-assisted development apply across all agents

## The Skills-Based Approach

The recommended approach for any project using AI coding agents:

1. **Document your standards** -- Maintain authoritative engineering standards as the single source of truth
2. **Create enforcement skills** -- Build skills that detect the tech stack and fetch the relevant standard automatically
3. **Configure your agent** -- Install skills so they activate without manual invocation
4. **Review all output** -- AI-generated code must pass the same review bar as human-written code

See [Agentic Coding Standards](./agentic-coding-standards) for the full approach, and the [Agent Skills](https://github.com/dannypenrose/agent-skills) repository for a reference implementation.

## When to Use AI Assistance

### Ideal Use Cases

- Boilerplate code generation
- Test case generation
- Documentation writing
- Code refactoring suggestions
- Bug investigation and debugging
- Code explanation and learning
- API integration scaffolding

### Exercise Caution

- Security-critical code (authentication, authorisation, encryption)
- Complex business logic with nuanced edge cases
- Performance-critical paths
- Production deployment scripts
- Cryptographic implementations

## Integration with Standards

AI-assisted development should follow all other engineering standards:

- [Security Guidelines](../governance/security-guidelines) -- Apply to all AI-generated code
- [Testing Strategy](../quality/testing-strategy) -- Test AI-generated code thoroughly
- [Code Review Guidelines](../quality/code-review-guidelines) -- Review AI code like human code
- [Documentation Standards](../governance/documentation-standards) -- Document AI-assisted decisions
