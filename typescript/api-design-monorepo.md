# API Design: Turborepo Monorepo

> Authoritative API design standards for Turborepo monorepo projects with shared conventions across multiple applications.

## Purpose

Establish consistent API design patterns across all applications in a monorepo, ensuring uniform client experiences regardless of which app or service handles the request.

## Core Principles

1. **Consistency first** - All apps follow identical response formats and conventions
2. **Shared types** - API types live in shared packages
3. **Versioned APIs** - All endpoints include version prefix
4. **Self-documenting** - Responses include enough context to be useful
5. **Pagination by default** - List endpoints always support pagination
6. **Predictable errors** - Error responses follow a standard format

## Response Format Standards

### Collection Endpoints (GET /resources)

```json
{
  "users": [
    { "id": "...", "email": "...", "name": "..." }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8,
    "hasNext": true,
    "hasPrev": false
  }
}
```

### Single Resource (GET /resources/:id)

```json
{
  "user": {
    "id": "abc123",
    "email": "user@example.com",
    "name": "John Doe",
    "createdAt": "2024-01-15T10:30:00.000Z",
    "updatedAt": "2024-01-20T14:00:00.000Z"
  }
}
```

### Create Response (POST /resources)

Status: `201 Created`

```json
{
  "user": {
    "id": "new-id",
    "email": "newuser@example.com",
    "name": "Jane Doe",
    "createdAt": "2024-01-23T09:00:00.000Z",
    "updatedAt": "2024-01-23T09:00:00.000Z"
  }
}
```

### Update Response (PUT/PATCH /resources/:id)

Status: `200 OK`

```json
{
  "user": {
    "id": "abc123",
    "email": "user@example.com",
    "name": "Updated Name",
    "createdAt": "2024-01-15T10:30:00.000Z",
    "updatedAt": "2024-01-23T09:15:00.000Z"
  }
}
```

### Delete Response (DELETE /resources/:id)

Status: `200 OK`

```json
{
  "message": "User deleted successfully"
}
```

## URL Structure

### Pattern

```
/api/v{version}/{resource}
/api/v{version}/{resource}/{id}
/api/v{version}/{resource}/{id}/{sub-resource}
```

### Conventions

| Rule | Example |
| ---- | ------- |
| Use plural nouns | `/users`, `/blog-posts` |
| Use kebab-case for multi-word | `/blog-posts`, `/api-keys` |
| Max 2 levels of nesting | `/users/{id}/posts` (OK), `/users/{id}/posts/{id}/comments` (Avoid) |
| Version prefix always | `/api/v1/users` |
| No trailing slashes | `/api/v1/users` (not `/api/v1/users/`) |

## Query Parameters

### Pagination

| Parameter | Type | Default | Max | Description |
| --------- | ---- | ------- | --- | ----------- |
| `page` | number | 1 | - | Page number (1-indexed) |
| `limit` | number | 20 | 100 | Items per page |

### Sorting

| Parameter | Type | Example | Description |
| --------- | ---- | ------- | ----------- |
| `sort` | string | `createdAt` | Field to sort by |
| `order` | string | `asc` or `desc` | Sort direction |

### Filtering

| Parameter | Type | Example | Description |
| --------- | ---- | ------- | ----------- |
| `search` | string | `john` | Full-text search |
| `{field}` | varies | `status=active` | Field-specific filter |

### Example

```
GET /api/v1/users?page=2&limit=10&sort=createdAt&order=desc&search=john&status=active
```

## Error Response Format

All errors follow this structure:

```json
{
  "statusCode": 400,
  "message": ["email must be a valid email", "password must be at least 8 characters"],
  "error": "Bad Request",
  "timestamp": "2024-01-23T09:00:00.000Z",
  "path": "/api/v1/users"
}
```

### HTTP Status Codes

| Code | Usage |
| ---- | ----- |
| 200 | Success (GET, PUT, PATCH, DELETE) |
| 201 | Created (POST) |
| 204 | No Content (DELETE with no body) |
| 400 | Bad Request (validation errors) |
| 401 | Unauthorized (missing/invalid auth) |
| 403 | Forbidden (insufficient permissions) |
| 404 | Not Found (resource doesn't exist) |
| 409 | Conflict (duplicate resource) |
| 422 | Unprocessable Entity (business logic error) |
| 429 | Too Many Requests (rate limited) |
| 500 | Internal Server Error |

## Shared Types Package

### Package Structure

```
packages/api-types/
├── src/
│   ├── index.ts
│   ├── common.ts        # Pagination, errors
│   ├── users.ts         # User-related types
│   ├── posts.ts         # Post-related types
│   └── responses.ts     # Response wrappers
└── package.json
```

### Type Definitions

```typescript
// packages/api-types/src/common.ts
export interface Pagination {
  page: number;
  limit: number;
  total: number;
  totalPages: number;
  hasNext: boolean;
  hasPrev: boolean;
}

export interface ApiError {
  statusCode: number;
  message: string[];
  error: string;
  timestamp: string;
  path: string;
}

export interface ListResponse<T> {
  pagination: Pagination;
}

export interface SingleResponse<T> {
  [key: string]: T;
}

// packages/api-types/src/users.ts
export interface User {
  id: string;
  email: string;
  name: string;
  createdAt: string;
  updatedAt: string;
}

export interface UsersListResponse extends ListResponse<User> {
  users: User[];
}

export interface UserResponse {
  user: User;
}

export interface CreateUserRequest {
  email: string;
  name: string;
  password: string;
}

export interface UpdateUserRequest {
  name?: string;
  email?: string;
}
```

## Field Naming Conventions

| Concept | Field Name |
| ------- | ---------- |
| Primary identifier | `id` |
| Foreign key reference | `{entity}Id` (e.g., `userId`) |
| Creation timestamp | `createdAt` |
| Update timestamp | `updatedAt` |
| Deletion timestamp | `deletedAt` |
| Boolean fields | `is{Adjective}` or `has{Noun}` (e.g., `isActive`, `hasEmail`) |
| Count fields | `{noun}Count` (e.g., `postCount`) |

## Date/Time Format

Always use ISO 8601 format in UTC:

```
"2024-01-23T09:30:00.000Z"
```

## Authentication

### Header

```
Authorization: Bearer <token>
```

### Error Responses

```json
// 401 Unauthorized - Missing or invalid token
{
  "statusCode": 401,
  "message": ["Invalid or expired token"],
  "error": "Unauthorized",
  "timestamp": "...",
  "path": "/api/v1/users"
}

// 403 Forbidden - Valid token but insufficient permissions
{
  "statusCode": 403,
  "message": ["You do not have permission to access this resource"],
  "error": "Forbidden",
  "timestamp": "...",
  "path": "/api/v1/admin/users"
}
```

## Authorisation and Data Scope

Authentication says who the caller is; authorisation decides what they may see and change. Apply the [Authorisation and Data Scope](/standards/governance/authorisation-and-data-scope) standard to every endpoint. The rules most often missed:

- **One shared permission policy** evaluates every setting (tenant, record visibility, per-action levels, account-wide flags). Endpoints call it; none re-implement a rule.
- **Scope the query, not the response.** Lists, single records, related lists, counts, search, exports, reports and bulk actions all start from the policy's scoped set. Checking only the detail endpoint leaves the same records exposed through the list, the search box and the export.
- **Updates and deletes check the stored record's owners**, loaded through the scoped set, not the owners sent in the request.
- **Ownership matches every identity the user has** (for example a legacy id and a current id, or a primary and secondary owner).
- **Never take the tenant or acting user from input.** Derive them from the authenticated session. Resolve every id in the request, including ids nested in payloads, through the scoped set; narrow bulk actions to the ids the caller may act on.
- **Bypassing global query filters re-applies the tenant condition explicitly.**
- **Deny by default.** Unset permissions are the most restrictive value; a user whose record cannot be loaded gets nothing.
- **Status codes:** a hidden record is `404`; a refused action is `403` with a readable reason naming the setting; never `401`, which clients treat as signed out.
- **Exports need the export permission**, settings exports included.
- **Identifiers from configuration used in SQL** (column names) are validated against a strict allow-list; every value is parameterised.
- **Enforced by tests:** policy tests, enforcement tests with a restricted user, a two-tenant isolation suite generated from the route list and run against the real database engine, and architecture tests that fail the build when a route lacks its gate.

In a monorepo the permission policy lives in a shared package (with the scoped-query helpers and the isolation test harness), so every app and every backend imports the same rule and no app copies a check.

## Rate Limiting

### Headers

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1706000000
```

### 429 Response

```json
{
  "statusCode": 429,
  "message": ["Too many requests, please try again later"],
  "error": "Too Many Requests",
  "timestamp": "...",
  "path": "/api/v1/users",
  "retryAfter": 60
}
```

## Checklist

### URL Design

- [ ] All endpoints use `/api/v1/` prefix
- [ ] Resource names are plural and kebab-case
- [ ] Maximum 2 levels of nesting
- [ ] No trailing slashes

### Response Format

- [ ] Collection endpoints return `{ items: [], pagination: {} }`
- [ ] Single resource endpoints return `{ resource: {} }`
- [ ] Create returns 201 with created resource
- [ ] Delete returns 200 with message

### Types

- [ ] Shared types package exists (`@org/api-types`)
- [ ] Request/response types for each endpoint
- [ ] Pagination type is reusable
- [ ] Error type is standardized

### Documentation

- [ ] OpenAPI/Swagger spec maintained
- [ ] All endpoints documented
- [ ] Request/response examples provided

### Authorisation and Data Scope

- [ ] Every endpoint applies the shared permission policy (lists, details, related lists, counts, search, exports, reports, bulk actions)
- [ ] Updates and deletes check the stored record's owners through the scoped set
- [ ] No tenant or user id taken from input; nested ids resolved through the scoped set
- [ ] Hidden records `404`, refused actions `403` with a reason, never `401`
- [ ] Export routes gated on the export permission
- [ ] Two-tenant isolation tests generated from the route list, run against the real database

## References

- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)
- [Google API Design Guide](https://cloud.google.com/apis/design)
- [JSON:API Specification](https://jsonapi.org/)
- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
