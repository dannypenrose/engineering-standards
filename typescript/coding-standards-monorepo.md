# Coding Standards: Turborepo Monorepo

> Authoritative coding standards for Turborepo monorepo projects with Next.js frontend and NestJS backend.

## Purpose

Establish consistent coding practices across all packages and applications in a monorepo environment, ensuring maintainability, scalability, and developer productivity.

## Core Principles

1. **Package-first architecture** - Shared code lives in packages, apps consume packages
2. **Strict TypeScript** - Enable strict mode in all packages and apps
3. **Consistent naming** - Follow established conventions across the entire monorepo
4. **Import boundaries** - Respect package boundaries, no circular dependencies
5. **Single responsibility** - Each package has a clear, focused purpose
6. **Test coverage** - Maintain minimum coverage thresholds per package

## Standards Reference Table

| Category | Standard | Example |
| -------- | -------- | ------- |
| File naming (React) | PascalCase | `UserProfile.tsx` |
| File naming (NestJS) | kebab-case | `user-profile.controller.ts` |
| File naming (utilities) | camelCase | `formatDate.ts` |
| Constants | SCREAMING_SNAKE_CASE | `API_ENDPOINTS` |
| Package naming | `@org/package-name` | `@org/backend-core` |
| Test files | `*.spec.ts` or `*.test.ts` | `user.service.spec.ts` |

## TypeScript Configuration

### Base Configuration (packages/tsconfig/base.json)

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "forceConsistentCasingInFileNames": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  }
}
```

### Return Types

Always define explicit return types for public functions:

```typescript
// Correct
export async function getUser(id: string): Promise<User> {
  return this.prisma.user.findUnique({ where: { id } });
}

// Incorrect - implicit return type
export async function getUser(id: string) {
  return this.prisma.user.findUnique({ where: { id } });
}
```

## Import Order

Maintain consistent import ordering across all files:

```typescript
// 1. Node.js built-in modules
import path from 'path';

// 2. External packages (npm)
import { Injectable } from '@nestjs/common';
import React from 'react';

// 3. Monorepo packages (@org/*)
import { User } from '@org/types';
import { formatDate } from '@org/utils';

// 4. Relative imports - parent directories
import { AppConfig } from '../../config';

// 5. Relative imports - same directory
import { UserDto } from './dto';
```

## Package Boundaries

### Correct: Import from packages

```typescript
// In apps/my-app/frontend/src/components/UserList.tsx
import { DataTable, Button } from '@org/ui';
import { formatDate } from '@org/utils';
import { User } from '@org/types';
```

### Incorrect: Cross-app imports or duplication

```typescript
// NEVER import from other apps
import { UserCard } from '../../../other-app/components/UserCard';

// NEVER duplicate shared code in apps
// apps/my-app/frontend/src/utils/formatDate.ts <- Should be in @org/utils
```

## Error Handling

### Backend (NestJS)

```typescript
import { NotFoundException, ConflictException, BadRequestException } from '@nestjs/common';

@Injectable()
export class UserService {
  async findOne(id: string): Promise<User> {
    const user = await this.prisma.user.findUnique({ where: { id } });
    if (!user) {
      throw new NotFoundException(`User with ID ${id} not found`);
    }
    return user;
  }

  async create(dto: CreateUserDto): Promise<User> {
    const existing = await this.prisma.user.findUnique({
      where: { email: dto.email },
    });
    if (existing) {
      throw new ConflictException('Email already in use');
    }
    return this.prisma.user.create({ data: dto });
  }
}
```

### Frontend (React/Next.js)

```typescript
import { isAxiosError } from 'axios';

export async function fetchUser(id: string): Promise<User> {
  try {
    const response = await api.get(`/users/${id}`);
    return response.data.user;
  } catch (error) {
    if (isAxiosError(error)) {
      if (error.response?.status === 404) {
        throw new Error('User not found');
      }
      throw new Error(error.response?.data?.message || 'Failed to fetch user');
    }
    throw error;
  }
}
```

## Async Patterns

### Always use async/await

```typescript
// Correct
async function getUsers(): Promise<User[]> {
  const users = await this.prisma.user.findMany();
  return users;
}

// Incorrect - .then() chains
function getUsers(): Promise<User[]> {
  return this.prisma.user.findMany().then((users) => users);
}
```

### Parallel operations

```typescript
const [user, posts, notifications] = await Promise.all([
  this.userService.findOne(userId),
  this.postService.findByUser(userId),
  this.notificationService.findUnread(userId),
]);
```

## Security Standards

### Input Validation (NestJS DTOs)

```typescript
import { IsEmail, IsString, MinLength, Transform } from 'class-validator';

export class CreateUserDto {
  @IsEmail()
  @Transform(({ value }) => value.toLowerCase().trim())
  email: string;

  @IsString()
  @MinLength(8)
  password: string;
}
```

### Never Log Sensitive Data

```typescript
// Correct
logger.info('User login', { userId: user.id });

// Incorrect - may expose sensitive data
logger.info('User login', { user }); // Could include password hash
logger.debug('Request body', { body }); // Could include credentials
```

## Test Coverage Requirements

| Package Type | Minimum Coverage | Critical Path Coverage |
| ------------ | ---------------- | ---------------------- |
| Shared packages (`@org/*`) | 60% | 80% |
| Backend apps | 50% | 80% |
| Frontend apps | 40% | 70% |
| Utilities | 80% | 90% |

## Turbo Pipeline Configuration

```json
{
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**"]
    },
    "type-check": {
      "dependsOn": ["^build"]
    },
    "lint": {},
    "test": {
      "dependsOn": ["^build"]
    }
  }
}
```

## Checklist

### Project Setup

- [ ] Base tsconfig extended by all packages
- [ ] ESLint configuration shared across monorepo
- [ ] Prettier configuration at root level
- [ ] Turbo pipeline configured for all tasks
- [ ] Package naming convention documented

### Code Quality

- [ ] Strict TypeScript enabled everywhere
- [ ] Explicit return types on public functions
- [ ] Import order enforced via ESLint
- [ ] No cross-app imports
- [ ] Error handling follows patterns

### Testing

- [ ] Coverage thresholds configured per package
- [ ] Test files follow naming convention
- [ ] Shared test utilities in `@org/testing` package

## References

- [Turborepo Documentation](https://turbo.build/repo/docs)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/)
- [NestJS Documentation](https://docs.nestjs.com/)
- [Next.js Documentation](https://nextjs.org/docs)
- [Google TypeScript Style Guide](https://google.github.io/styleguide/tsguide.html)
