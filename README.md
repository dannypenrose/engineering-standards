# Engineering Standards

[![License: CC BY-NC-ND 4.0](https://img.shields.io/badge/License-CC%20BY--NC--ND%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-nd/4.0/)

A comprehensive collection of 50 engineering standards covering the full software development lifecycle -- from coding conventions and API design through to security, reliability, deployment, and AI-assisted development.

These standards are opinionated, production-tested, and designed to enforce consistency across projects regardless of team size. They draw on practices from leading technology organisations and align with established frameworks including OWASP, NIST, W3C (WCAG), and the SRE discipline.

## What's Inside

### Language & Framework Standards

| Category | Documents | Covers |
|----------|-----------|--------|
| [TypeScript](./typescript) | 4 | Coding standards and API design for both monorepo (Turborepo) and standalone (Next.js + NestJS) architectures |
| [.NET](./dotnet) | 2 | C# coding standards and ASP.NET Core API design patterns |
| [Python](./python) | 2 | PEP 8-aligned coding standards and FastAPI/Django API design |

### Development Practices

| Category | Documents | Covers |
|----------|-----------|--------|
| [Development](./development) | 19 | API versioning, caching, CI/CD pipelines, containerisation, Chrome extensions, code sharing, data migration, dependency management, deployment, desktop (Tauri), environment management, Git standards & hooks, internationalisation, local development, mobile (Expo/React Native), real-time communication, state management, workspace tasks |

### Quality & Reliability

| Category | Documents | Covers |
|----------|-----------|--------|
| [Quality](./quality) | 4 | Testing strategy with coverage targets, code review guidelines, CI preflight strategy, performance budgets |
| [Reliability](./reliability) | 6 | Observability (logs, metrics, traces), error budgets & SLOs, incident management, load testing, chaos engineering, backup & disaster recovery |

### Governance & Security

| Category | Documents | Covers |
|----------|-----------|--------|
| [Governance](./governance) | 10 | Security guidelines (OWASP Top 10), infrastructure security, data governance & privacy, database standards, documentation standards, feature flags, accessibility (WCAG), ADR templates, open-source policy, self-hosting |

### AI-Assisted Development

| Category | Documents | Covers |
|----------|-----------|--------|
| [AI](./ai) | 4 | Agentic coding standards (platform-agnostic), Claude Code best practices, prompt engineering guidelines, AI code review patterns |

## Design Principles

Every standard in this collection follows the same principles:

- **Actionable over aspirational** -- concrete rules with code examples, not vague guidance
- **Justified** -- each rule explains *why* it exists, not just what to do
- **Checklistable** -- most standards include enforcement checklists for quick validation
- **Stack-aware** -- language-specific standards account for framework idioms rather than applying generic advice
- **Production-grade** -- written for real systems handling real traffic, not toy projects

## How These Standards Are Used

These standards serve as the **single source of truth** for development practices across projects. They are referenced in three ways:

### 1. Direct Reference

Consult the relevant standard before starting work on a feature, migration, deployment, or review. Each document is self-contained with quick-reference sections for common checks.

### 2. AI-Assisted Development

Standards are integrated into [Claude Code](https://claude.ai/claude-code) workflows via [Agent Skills](https://github.com/dannypenrose/agent-skills) that automatically detect the tech stack being worked on and fetch the relevant standard before writing or reviewing code. This ensures AI-generated code follows the same conventions as human-written code.

### 3. Compliance Auditing

A standards audit skill scans any project against its applicable standards and produces a compliance report with severity-rated findings and remediation guidance.

## Repository Structure

```
engineering-standards/
|-- ai/                     # AI-assisted development standards
|-- development/            # Cross-cutting development practices
|-- dotnet/                 # .NET / C# specific standards
|-- governance/             # Security, compliance, and architecture
|-- python/                 # Python specific standards
|-- quality/                # Testing, reviews, and performance
|-- reliability/            # Observability, SRE, and resilience
|-- typescript/             # TypeScript / JavaScript standards
|-- index.md                # Category overview and navigation
|-- LICENSE                 # CC BY-NC-ND 4.0
'-- README.md
```

## Industry Alignment

These standards are modelled on engineering practices at leading technology organisations and aligned with:

| Framework | Applied In |
|-----------|-----------|
| **OWASP Top 10** | Security guidelines, input validation, authentication |
| **NIST** | Infrastructure security, incident management |
| **W3C WCAG 2.2** | Accessibility standards |
| **SRE (Google)** | Error budgets, SLOs/SLIs, incident response |
| **12-Factor App** | Environment management, containerisation, deployment |
| **Conventional Commits** | Git standards |

## At a Glance

- **51** standards documents
- **8** categories
- **~27,500** lines of actionable guidance
- Covers **TypeScript**, **.NET**, and **Python** stacks
- Framework-specific patterns for **Next.js**, **NestJS**, **ASP.NET Core**, **FastAPI**, **Expo**, **Tauri**, and more
- Full lifecycle: coding &rarr; testing &rarr; security &rarr; deployment &rarr; observability &rarr; incident response

## Disclaimer

These standards have been developed and refined by Danny Penrose, who has over 20 years of experience in the technology industry. However, when used as context for AI-assisted development, it is important to understand that large language models are probabilistic systems. They can misinterpret rules, omit requirements, apply standards from the wrong context, or generate output that appears correct but deviates from the standard in subtle ways.

**These standards improve the quality of AI-generated output -- they do not guarantee it.** No set of instructions, however thorough, can eliminate the inherent limitations of current AI models. All AI-assisted output should be reviewed by an experienced software engineer before being committed, merged, or deployed. Treat AI as a capable assistant, not an authority.

The author accepts no liability for defects, security vulnerabilities, or other issues arising from code produced with the assistance of these standards.

## License

This work is licensed under [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/).

You may view and reference these standards for personal, non-commercial use. You may **not** modify, adapt, redistribute, or use them commercially without explicit written permission from the author. For commercial licensing enquiries, please [get in touch](https://github.com/dannypenrose).

Copyright (c) 2025-2026 Danny Penrose
