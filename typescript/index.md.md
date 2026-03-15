# TypeScript Standards

Implementation standards for TypeScript and JavaScript projects, including Next.js frontends and NestJS backends.

## Purpose

These standards provide concrete implementations of universal engineering principles using TypeScript-specific patterns, tools, and best practices.

## Standards Index

### Coding Standards

| Standard | Target Architecture |
| -------- | ------------------- |
| [Monorepo Standards](./coding-standards-monorepo) | Turborepo + Next.js + NestJS monorepo |
| [Next.js + NestJS Standards](./coding-standards-nextjs-nestjs) | Standalone full-stack applications |

### API Design

| Standard | Target Architecture |
| -------- | ------------------- |
| [Monorepo API Design](./api-design-monorepo) | Shared API conventions for monorepo |
| [Next.js + NestJS API Design](./api-design-nextjs-nestjs) | Standalone application API design |

## Tech Stack Coverage

### Frontend

- **Next.js** - React framework with App Router
- **React** - Component library
- **TypeScript** - Static typing
- **Tailwind CSS** - Utility-first styling
- **React Query / SWR** - Data fetching
- **Zod** - Runtime validation

### Backend

- **NestJS** - Node.js framework
- **Prisma** - Database ORM
- **TypeScript** - Static typing
- **class-validator** - DTO validation
- **Passport** - Authentication

### Tooling

- **pnpm** - Package manager
- **Turborepo** - Monorepo build system
- **ESLint** - Linting
- **Prettier** - Formatting
- **Vitest / Jest** - Testing
- **Playwright** - E2E testing

## Universal Standards Reference

These TypeScript implementations build upon:

- [Security Guidelines](../governance/security-guidelines) - Authentication, authorization, input validation
- [Database Standards](../governance/database-standards) - Schema design, migrations, indexing, naming conventions
- [Testing Strategy](../quality/testing-strategy) - Unit, integration, E2E testing patterns
- [Observability Standards](../reliability/observability-standards) - Logging, metrics, tracing
- [Performance Budgets](../quality/performance-budgets) - Core Web Vitals, bundle sizes
- [CI/CD Pipelines](../development/ci-cd-pipelines) - Next.js deploy workflow template, npm/pnpm caching

## Quick Reference

### Project Structure (Monorepo)

```text
apps/
├── web/           # Next.js frontend
├── api/           # NestJS backend
└── docs/          # Documentation

packages/
├── ui/            # Shared components
├── config/        # Shared configuration
├── types/         # Shared TypeScript types
└── utils/         # Shared utilities
```

### Project Structure (Standalone)

```text
src/
├── app/           # Next.js App Router
├── components/    # React components
├── lib/           # Utilities and services
├── api/           # NestJS modules (if colocated)
└── types/         # TypeScript types
```
