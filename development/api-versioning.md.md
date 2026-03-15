# API Versioning Standards

> Authoritative API versioning standards for managing breaking changes, deprecation, and lifecycle across all services.

## Purpose

Define how APIs are versioned, how breaking changes are communicated, and how consumers are migrated — ensuring stability for clients while allowing services to evolve.

## Core Principles

1. **Stability over convenience** — Existing clients must not break without explicit migration support
2. **Explicit versioning** — Every API endpoint has a clear, discoverable version
3. **Graceful deprecation** — Sunset periods with clear timelines and migration paths
4. **Additive by default** — Prefer backward-compatible changes over new versions
5. **Document everything** — Every version change has a changelog entry and migration guide

## Versioning Strategy

### URI Path Versioning (Default)

URI path versioning is the default strategy for all REST APIs:

```
GET /api/v1/users
GET /api/v2/users
```

This is the most widely adopted strategy because it is:
- Immediately visible in logs, documentation, and debugging tools
- Easy to route at the load balancer or gateway level
- Simple for consumers to understand and use

### When to Use Alternatives

| Strategy | Use When | Example |
| -------- | -------- | ------- |
| URI path (`/api/v1/`) | Default for all REST APIs | `GET /api/v1/users` |
| Header-based | Internal microservices with API gateway | `API-Version: 2` |
| Content negotiation | Strictly RESTful APIs with media type versioning | `Accept: application/vnd.myapp.v2+json` |
| Query parameter | Never for new APIs (legacy compatibility only) | `GET /api/users?version=2` |

### Implementation

#### NestJS / Express

```typescript
// Controller-level versioning
@Controller({ path: 'users', version: '1' })
export class UsersV1Controller {
  @Get()
  findAll() {
    return this.usersService.findAllV1();
  }
}

@Controller({ path: 'users', version: '2' })
export class UsersV2Controller {
  @Get()
  findAll() {
    return this.usersService.findAllV2();
  }
}

// App configuration
const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.URI,
  prefix: 'api/v',
});
```

#### ASP.NET Core

```csharp
// Program.cs
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = new UrlSegmentApiVersionReader();
})
.AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

// Controller
[ApiController]
[Route("api/v{version:apiVersion}/users")]
[ApiVersion("1.0")]
public class UsersV1Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() => Ok(/* v1 response */);
}

[ApiController]
[Route("api/v{version:apiVersion}/users")]
[ApiVersion("2.0")]
public class UsersV2Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() => Ok(/* v2 response */);
}
```

#### FastAPI (Python)

```python
from fastapi import APIRouter, FastAPI

app = FastAPI()

v1_router = APIRouter(prefix="/api/v1")
v2_router = APIRouter(prefix="/api/v2")

@v1_router.get("/users")
async def get_users_v1():
    return {"users": [...], "format": "v1"}

@v2_router.get("/users")
async def get_users_v2():
    return {"data": [...], "meta": {"total": 100}, "format": "v2"}

app.include_router(v1_router)
app.include_router(v2_router)
```

## Breaking vs Non-Breaking Changes

### Non-Breaking Changes (No Version Bump)

These changes are safe to make without a new API version:

- Adding a new optional field to a response
- Adding a new optional query parameter
- Adding a new endpoint
- Adding a new enum value (if clients handle unknown values)
- Relaxing a validation constraint (e.g., increasing max length)
- Adding support for a new media type

### Breaking Changes (Requires New Version)

These changes require a new API version:

- Removing or renaming a field from a response
- Removing or renaming a query parameter
- Changing a field's type (e.g., `string` → `number`)
- Changing the structure of a response (e.g., flat → nested)
- Tightening a validation constraint
- Changing authentication requirements
- Changing error response format
- Removing an endpoint
- Changing the meaning of an existing field

### Grey Areas

| Change | Guidance |
| ------ | -------- |
| Adding a required request field | Breaking — existing clients will fail |
| Changing default sort order | Breaking — clients may depend on order |
| Adding pagination to previously unpaginated endpoint | Breaking — response shape changes |
| Changing rate limits | Non-breaking if documented, but notify consumers |
| Adding a new error code | Non-breaking if clients handle unknown codes |

## Version Lifecycle

### Lifecycle Stages

```
Alpha → Beta → Stable → Deprecated → Sunset
```

| Stage | Stability | SLA | Duration |
| ----- | --------- | --- | -------- |
| **Alpha** | Unstable, may change without notice | None | Variable |
| **Beta** | Mostly stable, breaking changes possible with notice | Best effort | 1-3 months |
| **Stable** | Fully stable, only non-breaking changes | Full SLA | Until deprecated |
| **Deprecated** | Stable but no new features, fix critical bugs only | Maintenance SLA | 6-12 months minimum |
| **Sunset** | Removed, returns 410 Gone | None | — |

### Deprecation Process

1. **Announce** — Add `Deprecation` and `Sunset` headers to responses
2. **Document** — Publish migration guide with before/after examples
3. **Notify** — Email/notify API consumers directly
4. **Monitor** — Track usage of deprecated version
5. **Remind** — Send periodic reminders as sunset approaches
6. **Sunset** — Return `410 Gone` with migration link

### Deprecation Headers

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 01 Mar 2026 00:00:00 GMT
Link: <https://docs.example.com/api/migration/v1-to-v2>; rel="successor-version"
```

#### Implementation

```typescript
// NestJS middleware for deprecated endpoints
@Injectable()
export class DeprecationMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    res.setHeader('Deprecation', 'true');
    res.setHeader('Sunset', 'Sat, 01 Mar 2026 00:00:00 GMT');
    res.setHeader(
      'Link',
      '<https://docs.example.com/api/migration/v1-to-v2>; rel="successor-version"'
    );
    next();
  }
}
```

## Version Support Policy

### Concurrent Versions

- **Maximum 2 stable versions** supported simultaneously (current + previous)
- When `v3` is released, `v1` enters deprecation immediately
- When `v4` is released, `v1` is sunset

### Support Timeline

```
v1 Stable ─────────────────── Deprecated ──── Sunset
v2          Stable ─────────────────── Deprecated ──── Sunset
v3                    Stable ─────────────────── ...
                              ↑
                        v1 deprecated when
                        v3 becomes stable
```

### Minimum Deprecation Period

| Consumer Type | Minimum Notice | Minimum Sunset Period |
| ------------- | -------------- | --------------------- |
| Internal services | 30 days | 90 days |
| External partners | 90 days | 180 days |
| Public API | 180 days | 12 months |

## API Changelog

Every API version must maintain a changelog. Format:

```markdown
# API Changelog

## v2.0.0 (2026-03-15)

### Breaking Changes
- `GET /users` now returns paginated response `{ data: [], meta: { total, page, pageSize } }`
  - **Migration**: Update client to read `response.data` instead of `response` directly
- Removed `GET /users/search` — use `GET /users?q=term` instead

### New Features
- Added `GET /users/:id/activity` endpoint
- Added `filter` query parameter to `GET /users`

### Deprecations
- `GET /users/:id/posts` deprecated — use `GET /posts?userId=:id` instead (sunset: 2026-09-15)

## v1.3.0 (2026-02-01)

### New Features
- Added optional `metadata` field to user response
- Added `sort` query parameter to `GET /users`
```

## Response Envelope

Use a consistent response envelope across all versions:

```typescript
// Standard success response
interface ApiResponse<T> {
  data: T;
  meta?: {
    total?: number;
    page?: number;
    pageSize?: number;
    version?: string;
  };
}

// Standard error response
interface ApiError {
  error: {
    code: string;        // Machine-readable (e.g., "VALIDATION_ERROR")
    message: string;     // Human-readable description
    details?: unknown[];  // Field-level errors
    requestId: string;   // For support/debugging
  };
}
```

### Versioned Response Example

```typescript
// v1 response (flat)
{
  "users": [
    { "id": "1", "name": "Alice", "email": "alice@example.com" }
  ]
}

// v2 response (envelope with pagination)
{
  "data": [
    { "id": "1", "name": "Alice", "email": "alice@example.com", "createdAt": "2026-01-15T10:00:00Z" }
  ],
  "meta": {
    "total": 150,
    "page": 1,
    "pageSize": 20
  }
}
```

## Checklist

### Before Creating a New Version

- [ ] Change is confirmed as breaking (see breaking changes list)
- [ ] Non-breaking alternatives explored and ruled out
- [ ] Migration guide written with before/after examples
- [ ] API changelog updated
- [ ] Deprecation headers added to previous version
- [ ] Consumer notification plan in place
- [ ] Sunset date set (minimum period per consumer type)

### Version Maintenance

- [ ] Maximum 2 stable versions concurrently
- [ ] Deprecated versions monitored for usage
- [ ] Sunset reminders sent at 90, 30, and 7 days before removal
- [ ] Sunset returns 410 Gone with migration link
- [ ] Old version code cleaned up after sunset

### Documentation

- [ ] OpenAPI/Swagger spec updated for new version
- [ ] Version-specific documentation published
- [ ] Migration guide linked from deprecation headers
- [ ] Changelog entry added

## References

- [Semantic Versioning](https://semver.org/)
- [RFC 8594 — The Sunset HTTP Header](https://www.rfc-editor.org/rfc/rfc8594)
- [RFC 8288 — Web Linking](https://www.rfc-editor.org/rfc/rfc8288)
- [Microsoft REST API Guidelines — Versioning](https://github.com/microsoft/api-guidelines/blob/vNext/Guidelines.md#12-versioning)
- [Stripe API Versioning](https://stripe.com/docs/api/versioning)
