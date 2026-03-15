# Coding Standards: Standalone Next.js + NestJS

> Authoritative coding standards for standalone applications with Next.js frontend and NestJS backend.

## Purpose

Establish consistent coding practices for a two-tier application architecture with clear separation between frontend (Next.js) and backend (NestJS) codebases.

## Core Principles

1. **Clear separation** - Frontend and backend are distinct projects with their own dependencies
2. **API contract first** - Define API contracts before implementation
3. **Type sharing** - Share TypeScript types between frontend and backend
4. **Consistent patterns** - Use the same patterns within each tier
5. **Environment isolation** - Separate configuration per environment
6. **Testability** - Design for unit and integration testing

## Project Structure

```text
project/
├── frontend/                 # Next.js application
│   ├── src/
│   │   ├── app/             # App Router pages
│   │   ├── components/      # React components
│   │   ├── hooks/           # Custom React hooks
│   │   ├── lib/             # Utilities and API client
│   │   └── types/           # TypeScript types
│   ├── public/
│   ├── package.json
│   └── tsconfig.json
├── backend/                  # NestJS application
│   ├── src/
│   │   ├── modules/         # Feature modules
│   │   ├── common/          # Shared utilities
│   │   ├── config/          # Configuration
│   │   └── main.ts
│   ├── test/
│   ├── package.json
│   └── tsconfig.json
├── shared/                   # Shared types (optional)
│   └── types/
└── package.json              # Root package for scripts
```

## Standards Reference Table

| Category | Frontend (Next.js) | Backend (NestJS) |
| -------- | ------------------ | ---------------- |
| File naming | PascalCase for components | kebab-case for modules |
| State management | React hooks / Zustand | Service classes |
| Styling | Tailwind / CSS Modules | N/A |
| Testing | Jest + RTL | Jest + Supertest |
| API calls | Fetch / Axios | Controllers |

## TypeScript Configuration

### Frontend (tsconfig.json)

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "ES2022"],
    "jsx": "preserve",
    "module": "esnext",
    "moduleResolution": "bundler",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

### Backend (tsconfig.json)

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "target": "ES2022",
    "module": "commonjs",
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "declaration": true,
    "outDir": "./dist",
    "baseUrl": "./",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

## Frontend Standards

### Component Structure

```typescript
// src/components/UserProfile/UserProfile.tsx
'use client';

import { useState, useEffect } from 'react';
import type { User } from '@/types';
import { fetchUser } from '@/lib/api';

interface UserProfileProps {
  userId: string;
  onUpdate?: (user: User) => void;
}

export function UserProfile({ userId, onUpdate }: UserProfileProps): JSX.Element {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    async function loadUser(): Promise<void> {
      try {
        const data = await fetchUser(userId);
        setUser(data);
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Failed to load user');
      } finally {
        setLoading(false);
      }
    }
    loadUser();
  }, [userId]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!user) return <div>User not found</div>;

  return (
    <div className="user-profile">
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}
```

### API Client

```typescript
// src/lib/api.ts
const API_BASE = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3001';

interface ApiError {
  statusCode: number;
  message: string;
  error: string;
}

async function handleResponse<T>(response: Response): Promise<T> {
  if (!response.ok) {
    const error: ApiError = await response.json();
    throw new Error(error.message || `HTTP ${response.status}`);
  }
  return response.json();
}

export async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`${API_BASE}/api/v1/users/${id}`);
  const data = await handleResponse<{ user: User }>(response);
  return data.user;
}

export async function fetchUsers(params?: {
  page?: number;
  limit?: number;
}): Promise<{ users: User[]; pagination: Pagination }> {
  const searchParams = new URLSearchParams();
  if (params?.page) searchParams.set('page', params.page.toString());
  if (params?.limit) searchParams.set('limit', params.limit.toString());

  const response = await fetch(`${API_BASE}/api/v1/users?${searchParams}`);
  return handleResponse(response);
}
```

### NextAuth.js Configuration

When using NextAuth.js for authentication, always configure unique cookie names to prevent conflicts when running multiple apps on the same domain:

```typescript
// lib/auth.ts
import { NextAuthOptions } from 'next-auth';

export const authOptions: NextAuthOptions = {
  // REQUIRED: Unique cookie names per application
  cookies: {
    sessionToken: {
      name: `${APP_NAME}.session-token`,
      options: {
        httpOnly: true,
        sameSite: 'lax',
        path: '/',
        secure: process.env.NODE_ENV === 'production',
      },
    },
    callbackUrl: {
      name: `${APP_NAME}.callback-url`,
      options: {
        sameSite: 'lax',
        path: '/',
        secure: process.env.NODE_ENV === 'production',
      },
    },
    csrfToken: {
      name: `${APP_NAME}.csrf-token`,
      options: {
        httpOnly: true,
        sameSite: 'lax',
        path: '/',
        secure: process.env.NODE_ENV === 'production',
      },
    },
  },
  providers: [...],
  session: {
    strategy: 'jwt',
    maxAge: 24 * 60 * 60, // 24 hours
  },
  pages: {
    signIn: '/login',
    error: '/login',
  },
};
```

**Important**: When using custom cookie names, also update `getToken()` in middleware:

```typescript
// middleware.ts
const token = await getToken({
  req: request,
  secret: process.env.NEXTAUTH_SECRET,
  cookieName: `${APP_NAME}.session-token`,  // Must match custom cookie name
});
```

See [Security Guidelines - NextAuth.js Cookie Configuration](/standards/governance/security-guidelines#nextauthjs-cookie-configuration) for detailed explanation.

## Backend Standards

### Module Structure

```typescript
// src/modules/users/users.module.ts
import { Module } from '@nestjs/common';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

### Controller Pattern

```typescript
// src/modules/users/users.controller.ts
import {
  Controller,
  Get,
  Post,
  Put,
  Delete,
  Body,
  Param,
  Query,
  HttpCode,
  HttpStatus,
} from '@nestjs/common';
import { UsersService } from './users.service';
import { CreateUserDto, UpdateUserDto, QueryUsersDto } from './dto';

@Controller('api/v1/users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  async findAll(@Query() query: QueryUsersDto) {
    const { users, total } = await this.usersService.findAll(query);
    return {
      users,
      pagination: {
        page: query.page || 1,
        limit: query.limit || 20,
        total,
        totalPages: Math.ceil(total / (query.limit || 20)),
      },
    };
  }

  @Get(':id')
  async findOne(@Param('id') id: string) {
    const user = await this.usersService.findOne(id);
    return { user };
  }

  @Post()
  @HttpCode(HttpStatus.CREATED)
  async create(@Body() dto: CreateUserDto) {
    const user = await this.usersService.create(dto);
    return { user };
  }

  @Put(':id')
  async update(@Param('id') id: string, @Body() dto: UpdateUserDto) {
    const user = await this.usersService.update(id, dto);
    return { user };
  }

  @Delete(':id')
  async remove(@Param('id') id: string) {
    await this.usersService.remove(id);
    return { message: 'User deleted successfully' };
  }
}
```

### Service Pattern

```typescript
// src/modules/users/users.service.ts
import { Injectable, NotFoundException, ConflictException } from '@nestjs/common';
import { PrismaService } from '@/common/prisma/prisma.service';
import { CreateUserDto, UpdateUserDto, QueryUsersDto } from './dto';

@Injectable()
export class UsersService {
  constructor(private readonly prisma: PrismaService) {}

  async findAll(query: QueryUsersDto): Promise<{ users: User[]; total: number }> {
    const { page = 1, limit = 20, search } = query;
    const skip = (page - 1) * limit;

    const where = search
      ? {
          OR: [
            { name: { contains: search, mode: 'insensitive' } },
            { email: { contains: search, mode: 'insensitive' } },
          ],
        }
      : {};

    const [users, total] = await Promise.all([
      this.prisma.user.findMany({ where, skip, take: limit }),
      this.prisma.user.count({ where }),
    ]);

    return { users, total };
  }

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

  async update(id: string, dto: UpdateUserDto): Promise<User> {
    await this.findOne(id); // Throws if not found
    return this.prisma.user.update({ where: { id }, data: dto });
  }

  async remove(id: string): Promise<void> {
    await this.findOne(id); // Throws if not found
    await this.prisma.user.delete({ where: { id } });
  }
}
```

## Shared Types

```typescript
// shared/types/user.ts
export interface User {
  id: string;
  email: string;
  name: string;
  createdAt: string;
  updatedAt: string;
}

export interface Pagination {
  page: number;
  limit: number;
  total: number;
  totalPages: number;
}
```

## Test Coverage Requirements

| Layer | Minimum Coverage | Critical Path Coverage |
| ----- | ---------------- | ---------------------- |
| Backend Services | 70% | 90% |
| Backend Controllers | 50% | 80% |
| Frontend Components | 40% | 70% |
| Frontend Hooks | 60% | 80% |
| Shared Types | N/A (type-only) | N/A |

## Checklist

### Project Setup

- [ ] TypeScript strict mode enabled in both projects
- [ ] Shared types directory configured
- [ ] ESLint and Prettier configured
- [ ] Environment variables documented

### Frontend

- [ ] Components follow naming convention
- [ ] API client handles errors consistently
- [ ] Loading and error states handled
- [ ] Type imports use `type` keyword

### Backend

- [ ] Modules follow NestJS conventions
- [ ] DTOs use class-validator decorators
- [ ] Services handle all business logic
- [ ] Controllers are thin (delegation only)

### Integration

- [ ] API contract documented
- [ ] CORS configured correctly
- [ ] Environment-specific API URLs

## References

- [Next.js Documentation](https://nextjs.org/docs)
- [NestJS Documentation](https://docs.nestjs.com/)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/)
- [React Testing Library](https://testing-library.com/docs/react-testing-library/intro/)
