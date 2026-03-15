# Engineering Standards

This directory contains the authoritative engineering standards that apply across all projects. These are the **single source of truth** for development practices.

## Purpose

These standards ensure consistency and quality across all applications by establishing clear requirements for:

- Coding practices
- API design
- Testing strategies
- Security guidelines
- Observability & reliability
- Code reviews
- Accessibility
- Architecture decisions
- Data governance
- Operational excellence

## Standards by Category

### [Development](./development)

Tooling, source control, dependency management, and cross-cutting development patterns:

- Local development setup, deployment, and environment management
- Git standards, hooks, and dependency management
- Code sharing patterns and mobile/desktop/extension development
- API versioning, caching, containerization, and real-time communication
- Internationalization, state management, and data migration

### [Quality](./quality)

Testing, review, CI pipelines, and performance:

- Testing strategy and coverage targets
- Code review guidelines
- CI preflight strategy
- Performance budgets

### [Reliability](./reliability)

Observability, SLOs, incident management, and resilience:

- Observability and monitoring
- Error budgets and SLOs
- Incident management
- Load testing and chaos engineering

### [Governance](./governance)

Security, compliance, and architecture decisions:

- Security guidelines
- Data governance and privacy
- Feature flags
- Documentation and accessibility standards
- Database standards and ADR templates

### [TypeScript Standards](./typescript)

Implementation standards for TypeScript/JavaScript projects:

- Next.js + NestJS coding standards
- Monorepo conventions
- API design patterns
- React best practices

### [.NET Standards](./dotnet)

Implementation standards for .NET Core projects:

- C# coding standards
- ASP.NET Core API design
- Entity Framework patterns
- .NET specific tooling

### [Python Standards](./python)

Implementation standards for Python projects:

- Python coding standards (PEP 8 aligned)
- FastAPI/Django patterns
- Type hints and validation
- Python-specific tooling

### [AI-Assisted Development](./ai)

Standards for AI-assisted development across all agentic coding platforms:

- Agentic coding standards (platform-agnostic principles and skills-based enforcement)
- Claude Code best practices
- Prompt engineering guidelines
- AI code review patterns

## Industry Alignment

These standards are modelled on engineering practices at leading technology organisations and aligned with established industry frameworks including OWASP, NIST, W3C (WCAG), and the SRE discipline. The goal is production-grade rigour regardless of team size or project scope.

## How to Use

1. **Reference during development** - Consult the relevant standard before starting work
2. **Follow the requirements** - These are mandatory standards, not suggestions
3. **Enforce via agent skills** - Standards are loaded automatically by [agent skills](https://github.com/dannypenrose/agent-skills) during AI-assisted development
4. **Propose changes via PR** - If a standard needs updating, submit a pull request with justification

## Contributing

To propose changes to standards:

1. Create a branch with your proposed changes
2. Update the relevant standards document
3. Submit a PR with clear justification for the change
4. Get approval from at least one maintainer

Standards should be:

- **Actionable** - Clear rules that can be followed
- **Justified** - Explain why each rule exists
- **Maintained** - Updated when practices evolve
