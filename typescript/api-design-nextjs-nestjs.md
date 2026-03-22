# API Design: Standalone Next.js + NestJS

> Authoritative API design standards for standalone applications with Next.js frontend and NestJS backend.

## Purpose

Establish consistent API design patterns between the frontend and backend of a standalone application, ensuring type safety and developer productivity.

## Core Principles

1. **Type-safe communication** - Share types between frontend and backend
2. **RESTful design** - Follow REST conventions for predictability
3. **Error-first design** - Handle errors gracefully at every level
4. **Validation at boundaries** - Validate on both client and server
5. **Stateless APIs** - No server-side session state
6. **CORS-aware** - Configure cross-origin requests properly

## Project Communication Flow

```
Frontend (Next.js)              Backend (NestJS)
    │                                │
    │  HTTP Request                  │
    │  ─────────────────────────────>│
    │  (Authorization: Bearer token) │
    │                                │
    │                          ┌─────┴─────┐
    │                          │ Controller │
    │                          └─────┬─────┘
    │                                │
    │                          ┌─────┴─────┐
    │                          │  Service  │
    │                          └─────┬─────┘
    │                                │
    │                          ┌─────┴─────┐
    │                          │ Database  │
    │                          └───────────┘
    │                                │
    │  HTTP Response (JSON)          │
    │  <─────────────────────────────│
    │                                │
```

## Response Format Standards

### Success Responses

```typescript
// Collection endpoint
interface UsersListResponse {
  users: User[];
  pagination: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
    hasNext: boolean;
    hasPrev: boolean;
  };
}

// Single resource
interface UserResponse {
  user: User;
}

// Create/Update
interface CreateUserResponse {
  user: User;
}

// Delete
interface DeleteResponse {
  message: string;
}
```

### Error Responses

```typescript
interface ApiError {
  statusCode: number;
  message: string | string[];
  error: string;
  timestamp: string;
  path: string;
}
```

## Backend Implementation (NestJS)

### Controller Structure

```typescript
// backend/src/users/users.controller.ts
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
  ParseUUIDPipe,
} from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse } from '@nestjs/swagger';

@ApiTags('users')
@Controller('api/v1/users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  @ApiOperation({ summary: 'Get all users' })
  @ApiResponse({ status: 200, description: 'Returns paginated users' })
  async findAll(@Query() query: QueryUsersDto): Promise<UsersListResponse> {
    return this.usersService.findAll(query);
  }

  @Get(':id')
  @ApiOperation({ summary: 'Get user by ID' })
  @ApiResponse({ status: 200, description: 'Returns user' })
  @ApiResponse({ status: 404, description: 'User not found' })
  async findOne(
    @Param('id', ParseUUIDPipe) id: string,
  ): Promise<UserResponse> {
    const user = await this.usersService.findOne(id);
    return { user };
  }

  @Post()
  @HttpCode(HttpStatus.CREATED)
  @ApiOperation({ summary: 'Create new user' })
  @ApiResponse({ status: 201, description: 'User created' })
  @ApiResponse({ status: 400, description: 'Validation error' })
  @ApiResponse({ status: 409, description: 'Email already exists' })
  async create(@Body() dto: CreateUserDto): Promise<UserResponse> {
    const user = await this.usersService.create(dto);
    return { user };
  }

  @Put(':id')
  @ApiOperation({ summary: 'Update user' })
  @ApiResponse({ status: 200, description: 'User updated' })
  @ApiResponse({ status: 404, description: 'User not found' })
  async update(
    @Param('id', ParseUUIDPipe) id: string,
    @Body() dto: UpdateUserDto,
  ): Promise<UserResponse> {
    const user = await this.usersService.update(id, dto);
    return { user };
  }

  @Delete(':id')
  @ApiOperation({ summary: 'Delete user' })
  @ApiResponse({ status: 200, description: 'User deleted' })
  @ApiResponse({ status: 404, description: 'User not found' })
  async remove(
    @Param('id', ParseUUIDPipe) id: string,
  ): Promise<DeleteResponse> {
    await this.usersService.remove(id);
    return { message: 'User deleted successfully' };
  }
}
```

### DTO Validation

```typescript
// backend/src/users/dto/create-user.dto.ts
import { IsEmail, IsString, MinLength, MaxLength, Transform } from 'class-validator';
import { ApiProperty } from '@nestjs/swagger';

export class CreateUserDto {
  @ApiProperty({ example: 'user@example.com' })
  @IsEmail()
  @Transform(({ value }) => value.toLowerCase().trim())
  email: string;

  @ApiProperty({ example: 'John Doe' })
  @IsString()
  @MinLength(2)
  @MaxLength(100)
  @Transform(({ value }) => value.trim())
  name: string;

  @ApiProperty({ example: 'SecurePass123!' })
  @IsString()
  @MinLength(8)
  password: string;
}

// backend/src/users/dto/query-users.dto.ts
import { IsOptional, IsInt, Min, Max, IsString, IsIn } from 'class-validator';
import { Type } from 'class-transformer';

export class QueryUsersDto {
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page?: number = 1;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  limit?: number = 20;

  @IsOptional()
  @IsString()
  search?: string;

  @IsOptional()
  @IsString()
  sort?: string = 'createdAt';

  @IsOptional()
  @IsIn(['asc', 'desc'])
  order?: 'asc' | 'desc' = 'desc';
}
```

### Global Exception Filter

```typescript
// backend/src/common/filters/http-exception.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let message: string | string[] = 'Internal server error';
    let error = 'Internal Server Error';

    if (exception instanceof HttpException) {
      status = exception.getStatus();
      const exceptionResponse = exception.getResponse();

      if (typeof exceptionResponse === 'object') {
        message = (exceptionResponse as any).message || exception.message;
        error = (exceptionResponse as any).error || exception.name;
      } else {
        message = exceptionResponse;
      }
    }

    response.status(status).json({
      statusCode: status,
      message: Array.isArray(message) ? message : [message],
      error,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}
```

## Frontend Implementation (Next.js)

### API Client

```typescript
// frontend/src/lib/api/client.ts
const API_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3001';

interface RequestOptions extends RequestInit {
  params?: Record<string, string | number | boolean | undefined>;
}

class ApiClient {
  private baseUrl: string;

  constructor(baseUrl: string) {
    this.baseUrl = baseUrl;
  }

  private async request<T>(
    endpoint: string,
    options: RequestOptions = {},
  ): Promise<T> {
    const { params, ...init } = options;

    let url = `${this.baseUrl}${endpoint}`;
    if (params) {
      const searchParams = new URLSearchParams();
      Object.entries(params).forEach(([key, value]) => {
        if (value !== undefined) {
          searchParams.set(key, String(value));
        }
      });
      url += `?${searchParams.toString()}`;
    }

    const response = await fetch(url, {
      ...init,
      headers: {
        'Content-Type': 'application/json',
        ...init.headers,
      },
    });

    if (!response.ok) {
      const error = await response.json();
      throw new ApiError(error);
    }

    return response.json();
  }

  get<T>(endpoint: string, options?: RequestOptions): Promise<T> {
    return this.request<T>(endpoint, { ...options, method: 'GET' });
  }

  post<T>(endpoint: string, data: unknown, options?: RequestOptions): Promise<T> {
    return this.request<T>(endpoint, {
      ...options,
      method: 'POST',
      body: JSON.stringify(data),
    });
  }

  put<T>(endpoint: string, data: unknown, options?: RequestOptions): Promise<T> {
    return this.request<T>(endpoint, {
      ...options,
      method: 'PUT',
      body: JSON.stringify(data),
    });
  }

  delete<T>(endpoint: string, options?: RequestOptions): Promise<T> {
    return this.request<T>(endpoint, { ...options, method: 'DELETE' });
  }
}

export const api = new ApiClient(API_URL);

// Custom error class
export class ApiError extends Error {
  statusCode: number;
  errors: string[];

  constructor(error: { statusCode: number; message: string[] }) {
    super(error.message.join(', '));
    this.statusCode = error.statusCode;
    this.errors = error.message;
  }
}
```

### Typed API Functions

```typescript
// frontend/src/lib/api/users.ts
import { api } from './client';
import type {
  User,
  UsersListResponse,
  UserResponse,
  CreateUserRequest,
  UpdateUserRequest,
  DeleteResponse,
} from '@/types';

export interface GetUsersParams {
  page?: number;
  limit?: number;
  search?: string;
  sort?: string;
  order?: 'asc' | 'desc';
}

export const usersApi = {
  getAll: (params?: GetUsersParams): Promise<UsersListResponse> =>
    api.get('/api/v1/users', { params }),

  getById: (id: string): Promise<UserResponse> =>
    api.get(`/api/v1/users/${id}`),

  create: (data: CreateUserRequest): Promise<UserResponse> =>
    api.post('/api/v1/users', data),

  update: (id: string, data: UpdateUserRequest): Promise<UserResponse> =>
    api.put(`/api/v1/users/${id}`, data),

  delete: (id: string): Promise<DeleteResponse> =>
    api.delete(`/api/v1/users/${id}`),
};
```

### React Query Integration

```typescript
// frontend/src/hooks/useUsers.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { usersApi, GetUsersParams } from '@/lib/api/users';

export function useUsers(params?: GetUsersParams) {
  return useQuery({
    queryKey: ['users', params],
    queryFn: () => usersApi.getAll(params),
  });
}

export function useUser(id: string) {
  return useQuery({
    queryKey: ['users', id],
    queryFn: () => usersApi.getById(id),
    enabled: !!id,
  });
}

export function useCreateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: usersApi.create,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });
}

export function useUpdateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: UpdateUserRequest }) =>
      usersApi.update(id, data),
    onSuccess: (_, { id }) => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
      queryClient.invalidateQueries({ queryKey: ['users', id] });
    },
  });
}

export function useDeleteUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: usersApi.delete,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });
}
```

## CORS Configuration

```typescript
// backend/src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.enableCors({
    origin: process.env.FRONTEND_URL || 'http://forge-hub-dev.axiomstudio.io',
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
    allowedHeaders: ['Content-Type', 'Authorization'],
    credentials: true,
  });

  await app.listen(3001);
}
bootstrap();
```

## Checklist

### Backend

- [ ] Controllers follow REST conventions
- [ ] DTOs validate all input
- [ ] Global exception filter configured
- [ ] Swagger/OpenAPI documentation
- [ ] CORS properly configured

### Frontend

- [ ] Type-safe API client
- [ ] Error handling in all API calls
- [ ] Loading states handled
- [ ] React Query (or similar) for caching

### Integration

- [ ] Shared types between frontend/backend
- [ ] Environment variables for API URL
- [ ] Authentication token handling

## References

- [NestJS Documentation](https://docs.nestjs.com/)
- [Next.js Data Fetching](https://nextjs.org/docs/app/building-your-application/data-fetching)
- [TanStack Query](https://tanstack.com/query/latest)
- [class-validator](https://github.com/typestack/class-validator)
