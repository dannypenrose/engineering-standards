# AI-Assisted Development Standards

Standards and best practices for AI-assisted development with Claude Code, covering project setup, sub-agent orchestration, quality workflows, and security.

## Purpose

Establish consistent practices for leveraging AI tools effectively in software development while maintaining code quality, security, and team collaboration standards.

## Standards Index

| Standard | Description |
| -------- | ----------- |
| [Claude Code Standards](./claude-code-standards) | Project structure, CLAUDE.md templates, sub-agent orchestration, quality workflows, MCP integration, and security |
| [Prompt Engineering](./prompt-engineering) | Writing effective prompts, task decomposition, constraint techniques, and iterative refinement |
| [AI Code Review](./ai-code-review) | Reviewing AI-generated code, AI-specific failure modes, security checks, and pattern conformance |

## Core Principles

1. **AI as assistant, not replacement** - AI augments developer judgment, not replaces it
2. **Security awareness** - Never share secrets, credentials, or PII with AI tools
3. **Code review required** - AI-generated code must pass normal review processes
4. **Context is king** - Better context produces better results
5. **Iterative refinement** - Work with AI in conversation, not one-shot requests

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

- Security-critical code
- Authentication/authorization logic
- Cryptographic implementations
- Complex business logic
- Performance-critical paths
- Production deployment scripts

## Integration with Standards

AI-assisted development should follow all other engineering standards:

- [Security Guidelines](../governance/security-guidelines) - Apply to all AI-generated code
- [Testing Strategy](../quality/testing-strategy) - Test AI-generated code thoroughly
- [Code Review Guidelines](../quality/code-review-guidelines) - Review AI code like human code
- [Documentation Standards](../governance/documentation-standards) - Document AI-assisted decisions

## AI Code Review Checklist

When reviewing AI-generated code:

- [ ] Verify logic correctness
- [ ] Check for security vulnerabilities
- [ ] Ensure test coverage exists
- [ ] Validate error handling
- [ ] Confirm documentation accuracy
- [ ] Check for hardcoded values
- [ ] Verify consistent style
- [ ] Test edge cases
