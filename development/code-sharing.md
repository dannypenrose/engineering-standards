# Code Sharing Standard

> **Status:** Active
> **Applies to:** All monorepo projects
> **Last Updated:** January 2025

## Purpose

This standard establishes mandatory code sharing practices for monorepo architectures. The fundamental principle is that **shared code is the primary value proposition of a monorepo**. A monorepo without aggressive code sharing is merely a multi-repo with extra overhead.

This standard uses RFC 2119 terminology: MUST, MUST NOT, SHOULD, SHOULD NOT, MAY.

---

## 1. The Shared-First Principle

### 1.1 Core Requirement

All code that could potentially be used by more than one application MUST be placed in a shared package rather than duplicated across applications.

### 1.2 Decision Tree

Before writing any code, developers MUST evaluate placement using this decision tree:

```
Could another application use this functionality?
    │
    ├── YES → MUST place in shared package
    │         ├── UI Component       → @{org}/ui
    │         ├── Backend Module     → @{org}/backend-core
    │         ├── Utility Function   → @{org}/utils
    │         ├── Type Definition    → @{org}/types or @{org}/database
    │         ├── Validation Schema  → @{org}/validation
    │         └── Domain Logic       → @{org}/{domain}
    │
    └── NO (truly app-specific) → MAY place in application
              │
              └── MUST ask: "Could this become generic with minor refactoring?"
                        ├── YES → MUST refactor and place in shared package
                        └── NO → MAY keep in application with documented justification
```

### 1.3 Justification Requirement

When code is kept in an application rather than a shared package, the developer MUST document the justification in one of:
- Code comments explaining why the code is truly app-specific
- Pull request description
- Architecture Decision Record (ADR)

---

## 2. Package Boundaries

### 2.1 Import Rules

Applications MUST only import from:
- Shared packages (`@{org}/*`)
- External dependencies (npm packages)
- Internal application code

Applications MUST NOT import from:
- Other applications (cross-app imports are strictly forbidden)
- Package internals (only public API exports are permitted)

```typescript
// ✅ CORRECT
import { DataTable } from '@org/ui';
import { BlogModule } from '@org/backend-core';
import { formatDate } from '@org/utils';

// ❌ FORBIDDEN - cross-app import
import { UserCard } from '../../other-app/components/UserCard';

// ❌ FORBIDDEN - internal import
import { internalHelper } from '@org/ui/src/internal/helpers';
```

### 2.2 Public API Definition

Each shared package MUST define its public API through explicit exports in its entry point (`index.ts` or `index.js`). All other files are considered internal implementation details.

```typescript
// packages/ui/src/index.ts - This IS the public API
export { DataTable } from './DataTable';
export { Modal } from './Modal';
export { Button } from './Button';
// Internal files NOT exported are private to the package
```

---

## 3. Prohibited Practices

### 3.1 Code Duplication

The following practices are PROHIBITED:

| Practice | Violation |
|----------|-----------|
| Copying utilities between applications | MUST use shared package |
| Duplicating components across applications | MUST use shared package |
| Creating similar services in multiple applications | MUST use shared module |
| Copying types/interfaces between applications | MUST use shared types |
| Forking shared code for "slight modifications" | MUST use extension patterns |

### 3.2 Red Flag Statements

The following statements indicate a violation in progress:

- "I'll create a utility in this app..." → Check shared packages first
- "I need a component similar to..." → Import and configure existing component
- "I'll copy this from another app..." → NEVER permitted
- "This is slightly different so I'll duplicate..." → Use extension patterns
- "It's faster to just copy..." → Technical debt that compounds

---

## 4. Extension Patterns

When application-specific behavior is required, developers MUST use extension patterns rather than duplication.

### 4.1 Component Composition

```typescript
// ✅ CORRECT: Wrap shared component with app-specific behavior
import { DataTable } from '@org/ui';

export function CustomDataTable({ data }) {
  return (
    <DataTable
      data={data}
      columns={appColumns}
      onRowClick={handleAppRowClick}
      renderActions={(row) => <AppActions row={row} />}
    />
  );
}
```

### 4.2 Service Extension

```typescript
// ✅ CORRECT: Extend shared service with app-specific logic
import { BlogService } from '@org/backend-core';

@Injectable()
export class AppBlogService extends BlogService {
  async createPost(dto: CreatePostDto, authorId: string) {
    await this.validateCompliance(dto); // App-specific
    return super.createPost(dto, authorId); // Shared logic
  }
}
```

### 4.3 Module Configuration

```typescript
// ✅ CORRECT: Configure shared module at registration
import { BlogModule } from '@org/backend-core';

@Module({
  imports: [
    BlogModule.forRoot({
      maxPostsPerUser: 50,
      enableDrafts: true,
      customCategories: ['Reports', 'Analytics'],
    }),
  ],
})
export class AppModule {}
```

---

## 5. Package Ownership

### 5.1 Package Responsibility Matrix

Each shared package MUST have clearly defined ownership:

| Package Type | Contains | Ownership |
|--------------|----------|-----------|
| `@{org}/ui` | React components, hooks, themes | Frontend team |
| `@{org}/backend-core` | Backend modules, services, guards | Backend team |
| `@{org}/utils` | Pure utilities, formatters, helpers | Platform team |
| `@{org}/types` | TypeScript interfaces, type definitions | Platform team |
| `@{org}/validation` | Validation schemas (Zod, Yup, etc.) | Platform team |
| `@{org}/config` | Configuration, constants, feature flags | Platform team |
| `@{org}/testing` | Test utilities, mocks, fixtures | Platform team |

### 5.2 Contribution Requirements

Changes to shared packages MUST:
- Follow the package's contribution guidelines
- Include appropriate tests
- Update package documentation
- Consider backward compatibility
- Be reviewed by package owners

---

## 6. Migration Requirements

### 6.1 Discovering Duplicated Code

When duplicated code is discovered across applications, it MUST be migrated to a shared package.

### 6.2 Migration Process

1. **Identify** the most complete/correct implementation
2. **Move** to the appropriate shared package
3. **Export** from the package's public API
4. **Update** all consuming applications
5. **Delete** all duplicate copies
6. **Test** all affected applications

### 6.3 Migration Timeline

- Critical duplications (security, business logic): MUST be migrated within current sprint
- Standard duplications: SHOULD be migrated within 2 sprints
- Low-priority duplications: MUST be tracked and migrated within quarter

---

## 7. Quality Gates

### 7.1 Pre-Merge Checklist

All pull requests MUST verify:

- [ ] No new utilities duplicated across applications
- [ ] No new components that could be shared
- [ ] No code copied from other applications
- [ ] Extension patterns used instead of duplication
- [ ] Package boundaries respected
- [ ] Cross-app imports absent

### 7.2 Code Review Requirements

Reviewers MUST ask:

1. "Could this be in a shared package?"
2. "Does similar code exist elsewhere in the monorepo?"
3. "Is this extending shared code or duplicating it?"
4. "Are package boundaries respected?"

### 7.3 Automated Enforcement

Projects SHOULD implement automated checks:

- Lint rules preventing cross-app imports
- Dependency graph validation
- Duplicate code detection in CI/CD pipeline

---

## 8. Exceptions

### 8.1 Valid Exceptions

Code MAY remain in applications when it is:

- **Truly application-specific**: Unique business logic that cannot apply elsewhere
- **Experimental**: Prototype code that may be discarded
- **Configuration**: Application-specific settings and environment variables
- **One-off implementations**: Temporary solutions with documented removal timeline

### 8.2 Exception Documentation

All exceptions MUST be documented with:

- Justification for keeping code in application
- Criteria for when to reconsider sharing
- Review date for reassessment

---

## 9. Enforcement

### 9.1 Violation Consequences

| Severity | Example | Action |
|----------|---------|--------|
| Critical | Security logic duplicated | Block merge, immediate remediation |
| High | Business logic duplicated | Block merge until addressed |
| Medium | UI component duplicated | Merge with tech debt ticket |
| Low | Utility function duplicated | Warning, tracked for cleanup |

### 9.2 Escalation Path

1. Automated check flags violation
2. Code reviewer identifies issue
3. Author addresses or documents exception
4. Package owner approves exception if warranted
5. Architecture team reviews persistent patterns

---

## 10. Core Sharing Principles

1. **Atomic Changes**: Fix everything in the same commit
2. **Project Boundaries**: Applications import from packages, never from each other
3. **Public API Boundaries**: Clear exports, no internal access
4. **Visibility Rules**: Controlled access to sensitive packages
5. **Consistency**: Uniform patterns across all code

---

## References

- [Google's Monorepo](https://research.google/pubs/pub45424/)
- [Monorepo Explained](https://monorepo.tools/)
- [Nx Project Boundaries](https://nx.dev/concepts/enforce-project-boundaries)
- [Turborepo Handbook](https://turbo.build/repo/docs/handbook)
