# Testing Strategy

> Authoritative testing standards for consistent quality assurance across all projects and tech stacks.

## Purpose

Establish a comprehensive testing strategy that ensures code quality, prevents regressions, and enables confident deployments while maintaining developer productivity.

## Core Principles

1. **Test pyramid adherence** - More unit tests than integration tests than E2E tests
2. **Test behavior, not implementation** - Tests should survive refactoring
3. **Fast feedback loops** - Tests should run quickly in development
4. **Deterministic results** - No flaky tests in the main branch
5. **Coverage as a guide** - Use coverage to find gaps, not as a goal
6. **Test-driven when appropriate** - TDD for complex business logic

## Testing Pyramid

```
                    ┌─────────────┐
                    │    E2E      │  ~10% of tests
                    │   Tests     │  Slow, expensive, high confidence
                    ├─────────────┤
                    │ Integration │  ~20% of tests
                    │   Tests     │  Medium speed, real dependencies
                    ├─────────────┤
                    │   Unit      │  ~70% of tests
                    │   Tests     │  Fast, isolated, focused
                    └─────────────┘
```

## Test Types

### Unit Tests

**Purpose**: Test individual functions, methods, or classes in isolation

**Characteristics**:
- Run in milliseconds
- No external dependencies (database, network, file system)
- Mock all collaborators
- One logical assertion per test

```typescript
// Example: Unit test for a utility function
describe('formatCurrency', () => {
  it('should format positive numbers with currency symbol', () => {
    expect(formatCurrency(1234.56)).toBe('$1,234.56');
  });

  it('should handle zero', () => {
    expect(formatCurrency(0)).toBe('$0.00');
  });

  it('should format negative numbers with parentheses', () => {
    expect(formatCurrency(-100)).toBe('($100.00)');
  });
});
```

### Integration Tests

**Purpose**: Test interactions between components or with real dependencies

**Characteristics**:
- Run in seconds
- May use real database (test containers)
- Test actual API endpoints
- Verify component integration

```typescript
// Example: Integration test for an API endpoint
describe('POST /api/v1/users', () => {
  beforeEach(async () => {
    await db.users.deleteMany();
  });

  it('should create a user and return 201', async () => {
    const response = await request(app)
      .post('/api/v1/users')
      .send({ email: 'test@example.com', name: 'Test User' })
      .expect(201);

    expect(response.body.user).toMatchObject({
      email: 'test@example.com',
      name: 'Test User',
    });

    const dbUser = await db.users.findUnique({
      where: { email: 'test@example.com' },
    });
    expect(dbUser).not.toBeNull();
  });
});
```

### End-to-End (E2E) Tests

**Purpose**: Test complete user workflows through the UI

**Characteristics**:
- Run in minutes
- Use real browser (Playwright, Cypress)
- Test critical user journeys only
- More susceptible to flakiness

```typescript
// Example: E2E test for user login
test('user can log in and see dashboard', async ({ page }) => {
  await page.goto('/login');
  await page.fill('[data-testid="email"]', 'user@example.com');
  await page.fill('[data-testid="password"]', 'password123');
  await page.click('[data-testid="login-button"]');

  await expect(page).toHaveURL('/dashboard');
  await expect(page.locator('h1')).toContainText('Welcome');
});
```

## Coverage Requirements

### Minimum Thresholds

| Code Type | Line Coverage | Branch Coverage | Function Coverage |
| --------- | ------------- | --------------- | ----------------- |
| Business logic | 80% | 80% | 90% |
| Utilities | 90% | 85% | 95% |
| API controllers | 70% | 70% | 80% |
| UI components | 60% | 50% | 70% |
| Overall project | 70% | 60% | 75% |

### Critical Paths (100% coverage required)

- Authentication and authorization
- Payment processing
- Data validation
- Security-sensitive operations
- Core business rules

## Test Organization

### File Structure

```
src/
├── services/
│   ├── user.service.ts
│   └── user.service.spec.ts      # Co-located unit tests
├── controllers/
│   ├── user.controller.ts
│   └── user.controller.spec.ts
tests/
├── integration/
│   ├── users.integration.spec.ts
│   └── auth.integration.spec.ts
├── e2e/
│   ├── login.e2e.spec.ts
│   └── checkout.e2e.spec.ts
└── fixtures/
    ├── users.ts
    └── products.ts
```

### Naming Conventions

| Test Type | File Pattern | Example |
| --------- | ------------ | ------- |
| Unit | `*.spec.ts` or `*.test.ts` | `user.service.spec.ts` |
| Integration | `*.integration.spec.ts` | `users.integration.spec.ts` |
| E2E | `*.e2e.spec.ts` | `checkout.e2e.spec.ts` |

## Test Patterns

### Arrange-Act-Assert (AAA)

```typescript
it('should return user by ID', async () => {
  // Arrange
  const mockUser = { id: '123', name: 'John' };
  mockRepository.findById.mockResolvedValue(mockUser);

  // Act
  const result = await service.getUser('123');

  // Assert
  expect(result).toEqual(mockUser);
  expect(mockRepository.findById).toHaveBeenCalledWith('123');
});
```

### Given-When-Then (BDD)

```typescript
describe('OrderService', () => {
  describe('given a valid order', () => {
    describe('when processing payment', () => {
      it('then should create order and charge customer', async () => {
        // ...
      });
    });
  });
});
```

## Mocking Strategies

### When to Mock

| Mock | Don't Mock |
| ---- | ---------- |
| External APIs | Core business logic |
| Database (unit tests) | Pure functions |
| File system | Data transformations |
| Time/Date | Value objects |
| Third-party services | Domain entities |

### Mock Implementation

```typescript
// Create mock
const mockUserRepository = {
  findById: jest.fn(),
  save: jest.fn(),
  delete: jest.fn(),
};

// Setup behavior
mockUserRepository.findById.mockResolvedValue({ id: '1', name: 'Test' });

// Verify calls
expect(mockUserRepository.findById).toHaveBeenCalledWith('1');
expect(mockUserRepository.findById).toHaveBeenCalledTimes(1);
```

## Test Data Management

### Fixtures

```typescript
// tests/fixtures/users.ts
export const testUsers = {
  admin: {
    id: 'admin-123',
    email: 'admin@example.com',
    role: 'admin',
  },
  regular: {
    id: 'user-456',
    email: 'user@example.com',
    role: 'user',
  },
};

export function createTestUser(overrides: Partial<User> = {}): User {
  return {
    id: randomUUID(),
    email: `test-${randomUUID()}@example.com`,
    name: 'Test User',
    createdAt: new Date(),
    ...overrides,
  };
}
```

### Database Seeding

```typescript
// tests/setup/database.ts
export async function seedDatabase() {
  await db.users.createMany({ data: testUsers });
}

export async function cleanDatabase() {
  await db.users.deleteMany();
}
```

## CI Integration

### Pipeline Configuration

```yaml
test:
  runs-on: ubuntu-latest
  steps:
    - name: Unit Tests
      run: pnpm test:unit --coverage

    - name: Integration Tests
      run: pnpm test:integration

    - name: E2E Tests
      run: pnpm test:e2e
      continue-on-error: true  # E2E can be flaky

    - name: Upload Coverage
      uses: codecov/codecov-action@v3

    - name: Check Coverage Thresholds
      run: pnpm test:coverage-check
```

### Parallel Execution

```typescript
// jest.config.js
module.exports = {
  maxWorkers: '50%',  // Use half of available CPUs
  testTimeout: 10000,
  // Shard tests across CI workers
  shard: process.env.CI ? `${process.env.SHARD_INDEX}/${process.env.SHARD_TOTAL}` : undefined,
};
```

## Flaky Test Policy

### Definition

A flaky test is one that produces different results (pass/fail) without code changes.

### Handling

1. **Quarantine immediately** - Move to separate test suite
2. **Investigate within 48 hours** - Find root cause
3. **Fix or delete** - No flaky tests in main suite
4. **Track metrics** - Monitor flakiness rate

### Common Causes

- Timing/race conditions
- Shared state between tests
- External service dependencies
- Date/time dependencies
- Order-dependent tests

## Checklist

### Project Setup

- [ ] Test framework configured (Jest, Vitest, xUnit)
- [ ] Coverage reporting enabled
- [ ] CI pipeline runs tests
- [ ] Test database available
- [ ] Fixtures/factories created

### Test Quality

- [ ] Unit tests for all business logic
- [ ] Integration tests for API endpoints
- [ ] E2E tests for critical paths
- [ ] No flaky tests in main suite
- [ ] Coverage thresholds enforced

### Process

- [ ] Tests run before merge
- [ ] Failed tests block deployment
- [ ] Coverage reported on PRs
- [ ] Flaky tests tracked and addressed

## References

- [Google Testing Blog](https://testing.googleblog.com/)
- [Test Pyramid by Martin Fowler](https://martinfowler.com/articles/practical-test-pyramid.html)
- [Jest Documentation](https://jestjs.io/docs/getting-started)
- [Playwright Documentation](https://playwright.dev/docs/intro)
- [Testing Library](https://testing-library.com/)
