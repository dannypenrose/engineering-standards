# AI Code Review Standards

> Guidelines for reviewing code generated or modified by AI tools. AI-generated code must meet the same quality bar as human-written code — these standards define what to look for and how to verify it.

## Purpose

AI tools produce code that compiles and often passes basic tests, but can introduce subtle issues that human review catches. This standard defines a systematic review process for AI-generated code to ensure it meets project quality, security, and maintainability standards.

## Core Principles

1. **Same bar, different risks** - AI code meets the same standards as human code, but has different failure modes
2. **Trust but verify** - AI output is a starting point, not a final product
3. **Pattern conformance** - AI may invent its own patterns instead of following established ones
4. **Security by default** - AI tools may not prioritise security unless explicitly instructed
5. **Understand before approving** - Never approve code you don't understand, regardless of source

## AI-Specific Failure Modes

AI-generated code has characteristic weaknesses that differ from typical human mistakes:

| Failure Mode | Description | Detection Strategy |
|---|---|---|
| **Pattern invention** | Creates new patterns instead of following existing ones | Compare against established code in the same area |
| **Over-engineering** | Adds unnecessary abstractions, configurations, or error handling | Check if the complexity is justified by requirements |
| **Phantom APIs** | References functions, methods, or packages that don't exist | Verify all imports and external references resolve |
| **Stale knowledge** | Uses deprecated APIs or outdated patterns | Check framework version compatibility |
| **Shallow error handling** | Catches errors but doesn't handle them meaningfully | Trace error paths through to user-visible behaviour |
| **Test theatre** | Tests that pass but don't verify meaningful behaviour | Review assertions — do they test outcomes or implementation? |
| **Context drift** | Later code contradicts earlier decisions in the same session | Review the full changeset holistically, not file-by-file |
| **Confident incorrectness** | Produces plausible but wrong logic with no uncertainty markers | Manually verify business logic and edge cases |

## Review Checklist

### 1. Correctness

- [ ] **Logic is sound** — trace through the code mentally with sample inputs
- [ ] **Edge cases handled** — null, empty, boundary values, error states
- [ ] **Business rules correct** — matches requirements, not just technical correctness
- [ ] **No phantom references** — all imports, types, and function calls resolve to real code
- [ ] **API contracts honoured** — request/response shapes match what consumers expect

### 2. Pattern Conformance

- [ ] **Follows existing patterns** — compare against similar code already in the codebase
- [ ] **Naming conventions** — matches project style (camelCase, PascalCase, etc.)
- [ ] **File placement** — new files are in the correct directory per project structure
- [ ] **Error handling style** — matches how errors are handled elsewhere in the project
- [ ] **No new abstractions** unless explicitly requested — AI tends to over-abstract

### 3. Security

- [ ] **No hardcoded secrets** — API keys, passwords, tokens, connection strings
- [ ] **Input validation** — all external inputs validated at system boundaries
- [ ] **SQL injection** — parameterised queries, no string concatenation
- [ ] **XSS prevention** — outputs sanitised, no `dangerouslySetInnerHTML` without justification
- [ ] **Authentication/authorisation** — protected routes and endpoints checked
- [ ] **Tenant isolation** — multi-tenant data properly scoped (if applicable)
- [ ] **Error messages** — no sensitive information leaked to end users

### 4. Testing

- [ ] **Tests exist** — new code has corresponding tests
- [ ] **Tests are meaningful** — assertions verify behaviour, not just "it runs"
- [ ] **Edge cases covered** — not just the happy path
- [ ] **Negative cases included** — invalid inputs, error conditions, permission failures
- [ ] **No test pollution** — tests are independent, no shared mutable state
- [ ] **Mocks are appropriate** — mock at boundaries, not internal implementations

### 5. Performance

- [ ] **No N+1 queries** — database access patterns are efficient
- [ ] **Appropriate data fetching** — not over-fetching or under-fetching
- [ ] **No unnecessary re-renders** — React components use memoisation where needed
- [ ] **Async operations** — blocking calls are async where they should be
- [ ] **Resource cleanup** — connections, streams, event listeners properly disposed

### 6. Maintainability

- [ ] **Readable without explanation** — code is self-documenting
- [ ] **No dead code** — unused variables, functions, imports removed
- [ ] **Appropriate complexity** — simplest solution that meets requirements
- [ ] **Dependencies justified** — no new packages added without clear need
- [ ] **Documentation updated** — CLAUDE.md, feature index, API docs as needed

## Review Process

### Quick Review (< 50 lines changed)

For small, focused changes:

1. Read the diff
2. Run through the correctness and security checklists
3. Verify pattern conformance with one or two existing examples
4. Confirm tests pass

### Standard Review (50-200 lines changed)

For typical feature work:

1. **Understand intent** — What was the AI asked to do?
2. **Read holistically** — Review the full changeset, not file-by-file
3. **Check pattern conformance** — Open comparable existing code side by side
4. **Run the full checklist** — All 6 sections above
5. **Run tests locally** — Don't trust CI alone for AI-generated code
6. **Test manually** — Click through the feature, try edge cases

### Deep Review (> 200 lines or architectural changes)

For large or structural changes:

1. **Review the plan first** — Was a plan approved before implementation?
2. **Architectural alignment** — Does it match architecture docs?
3. **Full checklist** — All 6 sections, extra scrutiny on security and performance
4. **Integration testing** — How does it interact with existing systems?
5. **Rollback plan** — What happens if this needs to be reverted?
6. **Second reviewer** — Get another pair of eyes for significant changes

## Common AI Review Findings

### Finding: Invented Pattern

**Symptom:** AI creates a new utility function when one already exists.

```typescript
// AI wrote this
function formatDate(date: Date): string {
  return date.toISOString().split('T')[0];
}

// But the project already has this
import { formatDate } from '@/lib/date-utils';
```

**Fix:** Replace with existing utility. If the AI's version is better, update the existing utility.

### Finding: Over-Engineering

**Symptom:** AI adds configuration, feature flags, or abstraction layers that weren't requested.

```typescript
// AI wrote this (unrequested abstraction)
interface NotificationStrategy { send(msg: string): Promise<void> }
class EmailStrategy implements NotificationStrategy { ... }
class SlackStrategy implements NotificationStrategy { ... }
class NotificationService {
  constructor(private strategy: NotificationStrategy) {}
}

// What was needed
async function sendEmail(to: string, subject: string, body: string) {
  await emailClient.send({ to, subject, body });
}
```

**Fix:** Simplify to what was actually requested. Three lines is better than a premature abstraction.

### Finding: Shallow Error Handling

**Symptom:** Errors are caught but swallowed or logged without meaningful handling.

```typescript
// AI wrote this
try {
  const result = await api.createUser(data);
  return result;
} catch (error) {
  console.error('Error creating user:', error);
  return null; // Caller has no idea what went wrong
}
```

**Fix:** Propagate errors meaningfully — rethrow, return error types, or handle with user-visible feedback.

### Finding: Test Theatre

**Symptom:** Tests that assert implementation details instead of behaviour.

```typescript
// AI wrote this (test theatre)
it('should call repository.save', async () => {
  await service.createUser(userData);
  expect(mockRepo.save).toHaveBeenCalledTimes(1);
});

// Better: test observable behaviour
it('should persist the user and return the created record', async () => {
  const result = await service.createUser(userData);
  expect(result.email).toBe(userData.email);
  expect(result.id).toBeDefined();
});
```

## Integration with Existing Standards

AI code review is an overlay on top of standard code review practices:

- [Code Review Guidelines](/standards/quality/code-review-guidelines) — General review practices apply first
- [Security Guidelines](/standards/governance/security-guidelines) — All OWASP checks apply
- [Testing Strategy](/standards/quality/testing-strategy) — Test quality standards apply
- Language-specific standards ([TypeScript](/standards/typescript), [.NET](/standards/dotnet), [Python](/standards/python)) — Coding conventions apply

## Checklist Summary

Before approving any AI-generated code:

- [ ] Logic is correct and handles edge cases
- [ ] Follows existing project patterns (not invented ones)
- [ ] No security vulnerabilities (OWASP top 10)
- [ ] Tests are meaningful, not theatrical
- [ ] No unnecessary complexity or over-engineering
- [ ] All imports and references resolve to real code
- [ ] Documentation updated where needed
- [ ] You understand every line being merged
