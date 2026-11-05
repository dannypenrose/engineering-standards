# PHP API Design Standards

> Authoritative API design standards for PHP backends: Laravel REST APIs and the WordPress REST API.

## Purpose

Establish consistent HTTP API patterns for PHP services, so that a Laravel API and a WordPress REST route look the same to the client that calls them. These rules implement the cross-stack conventions in [API Versioning](/standards/development/api-versioning) and [Security Guidelines](/standards/governance/security-guidelines).

## Core Principles

1. **Thin controllers**: a controller authorises, validates, calls one action and returns a resource
2. **Validation at the boundary**: every write endpoint has a dedicated request class or schema; nothing reads raw input
3. **Resources shape responses**: never return an Eloquent model or `WP_Post` directly
4. **Authorisation is explicit**: every route declares who may call it; "public" is a decision, not a default
5. **Versioned from day one**: every route lives under a version prefix
6. **One error shape**: every failure, including validation, returns the same JSON envelope

## URL Conventions

| Rule | Example |
|------|---------|
| Version prefix | `/api/v1/...` (Laravel), `/wp-json/{vendor}/v1/...` (WordPress) |
| Plural nouns for collections | `/api/v1/invoices` |
| kebab-case paths | `/api/v1/payment-methods` |
| Nesting at most one level | `/api/v1/customers/{customer}/invoices` |
| Actions that are not CRUD use a sub-resource verb | `POST /api/v1/invoices/{invoice}/void` |
| snake_case JSON keys | `created_at`, `line_items` |
| Query parameters | `?page=2&limit=25&sort=-created_at&filter[status]=paid` |

## Response Format

### Success

```json
// GET /api/v1/invoices/42
{
  "data": {
    "id": 42,
    "status": "issued",
    "total": { "amount": 12500, "currency": "GBP" },
    "created_at": "2026-09-01T10:15:00Z"
  }
}
```

```json
// GET /api/v1/invoices?page=2&limit=25
{
  "data": [ { "id": 42, "status": "issued" } ],
  "pagination": {
    "page": 2,
    "limit": 25,
    "total": 240,
    "total_pages": 10,
    "has_next": true,
    "has_prev": true
  }
}
```

- Timestamps are ISO 8601 in UTC
- Money is integer minor units plus a currency code
- `limit` has a documented maximum (default 25, maximum 100); requests above it are clamped, not rejected
- Use cursor pagination (`cursorPaginate()`) for large or frequently appended tables

### Errors

```json
// 422 Unprocessable Content
{
  "error": "The given data was invalid.",
  "code": "VALIDATION_FAILED",
  "details": {
    "email": ["The email field must be a valid email address."]
  }
}
```

| Status | When | `code` |
|--------|------|--------|
| 400 | Malformed request (bad JSON, wrong content type) | `BAD_REQUEST` |
| 401 | No or invalid credentials | `UNAUTHENTICATED` |
| 403 | Authenticated but not allowed | `FORBIDDEN` |
| 404 | Not found, **or** exists but the caller may not know it exists | `NOT_FOUND` |
| 409 | State conflict (already paid, duplicate) | `CONFLICT` |
| 422 | Validation failed | `VALIDATION_FAILED` |
| 429 | Rate limited (include `Retry-After`) | `RATE_LIMITED` |
| 500 | Unexpected failure: generic message, details logged not returned | `INTERNAL_ERROR` |

Never return a stack trace, SQL, file path or exception class in a response body. `APP_DEBUG` is `false` in every non-local environment.

## Laravel

### Routes

```php
// routes/api.php
use App\Http\Controllers\Api\V1\InvoiceController;
use App\Http\Controllers\Api\V1\VoidInvoiceController;

Route::prefix('v1')
    ->middleware(['auth:sanctum', 'throttle:api'])
    ->group(function (): void {
        Route::apiResource('invoices', InvoiceController::class);
        Route::post('invoices/{invoice}/void', VoidInvoiceController::class);
    });
```

- Use `apiResource()` for CRUD and single-action invokable controllers for everything else
- Use route model binding, and scoped bindings (`->scopeBindings()`) for nested resources so `/customers/1/invoices/99` cannot return another customer's invoice
- Name the rate limiter in `AppServiceProvider` and apply it to every group

### Controller

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers\Api\V1;

use App\Actions\Invoices\CreateInvoice;
use App\Http\Requests\StoreInvoiceRequest;
use App\Http\Resources\InvoiceResource;
use App\Models\Invoice;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;
use Illuminate\Support\Facades\Gate;
use Symfony\Component\HttpFoundation\Response;

final class InvoiceController
{
    public function index(Request $request): AnonymousResourceCollection
    {
        Gate::authorize('viewAny', Invoice::class);

        $invoices = $request->user()
            ->invoices()
            ->with('lineItems')
            ->latest()
            ->paginate(min($request->integer('limit', 25), 100));

        return InvoiceResource::collection($invoices);
    }

    public function store(StoreInvoiceRequest $request, CreateInvoice $createInvoice): JsonResponse
    {
        $invoice = $createInvoice->handle($request->user(), $request->toData());

        return InvoiceResource::make($invoice)
            ->response()
            ->setStatusCode(Response::HTTP_CREATED);
    }

    public function show(Invoice $invoice): InvoiceResource
    {
        Gate::authorize('view', $invoice);

        return InvoiceResource::make($invoice->load('lineItems'));
    }
}
```

### Validation: Form Requests

```php
final class StoreInvoiceRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create', Invoice::class);
    }

    /** @return array<string, list<mixed>> */
    public function rules(): array
    {
        return [
            'customer_id' => ['required', 'integer', Rule::exists('customers', 'id')->where('team_id', $this->user()->team_id)],
            'currency' => ['required', Rule::enum(Currency::class)],
            'line_items' => ['required', 'array', 'min:1', 'max:200'],
            'line_items.*.description' => ['required', 'string', 'max:255'],
            'line_items.*.amount' => ['required', 'integer', 'min:0'],
        ];
    }

    public function toData(): NewInvoiceData
    {
        return NewInvoiceData::fromArray($this->validated());
    }
}
```

| Rule | Why |
|------|-----|
| Use `$request->validated()`, never `$request->all()` | `all()` passes unvalidated keys straight into the model |
| Scope `exists` rules to the caller's tenant | An unscoped `exists:customers,id` lets a user attach another tenant's record |
| Bound every array (`max:`) and string length | Unbounded input is a denial-of-service vector |
| Models use `$fillable`, never `$guarded = []` | Mass assignment protection is the last line of defence |

### Responses: API Resources

```php
/** @mixin \App\Models\Invoice */
final class InvoiceResource extends JsonResource
{
    /** @return array<string, mixed> */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'status' => $this->status->value,
            'total' => [
                'amount' => $this->total_minor,
                'currency' => $this->currency->value,
            ],
            'line_items' => LineItemResource::collection($this->whenLoaded('lineItems')),
            'created_at' => $this->created_at->toIso8601ZuluString(),
        ];
    }
}
```

Use `whenLoaded()` for relationships so a resource never triggers a lazy query. Override the paginated response in a base collection so list endpoints emit the `pagination` envelope above rather than Laravel's default `links`/`meta`.

### Error Rendering

Render every exception to the standard shape in `bootstrap/app.php`:

```php
->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->shouldRenderJsonWhen(fn (Request $request) => $request->is('api/*'));

    $exceptions->render(function (ValidationException $e, Request $request) {
        return response()->json([
            'error' => 'The given data was invalid.',
            'code' => 'VALIDATION_FAILED',
            'details' => $e->errors(),
        ], 422);
    });

    $exceptions->render(function (DomainException $e, Request $request) {
        return response()->json([
            'error' => $e->getMessage(),
            'code' => $e->errorCode(),
            'details' => (object) [],
        ], $e->status());
    });
})
```

### Authentication and Authorisation

- **Sanctum** for first-party SPAs (cookie-based, with CSRF) and simple API tokens; **Passport** only when you genuinely need to be an OAuth2 server
- Token abilities (`$user->createToken('ci', ['invoices:read'])`) restrict what a token can do
- A **Policy** per model; every controller method calls `Gate::authorize()` or relies on a Form Request's `authorize()`
- Return 404 rather than 403 when revealing existence would leak information (another tenant's record)

### Documentation

Generate an OpenAPI document from the code (for example with Scramble or `l5-swagger`) and publish it with the API. A route that is not in the OpenAPI document is not part of the contract.

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

In Laravel, a global scope on every tenant-owned model applies the tenant from the authenticated session; a local `visibleTo($user)` scope applies record visibility, and every list, count, search, export and bulk query starts from `Model::visibleTo($request->user())`. A Policy per model decides each action against the stored record (`Gate::authorize('update', $contact)` after loading `$contact` through `visibleTo()`, so a hidden record is a `404`), and `$this->deny('...')` gives the readable `403` reason. Any query that calls `withoutGlobalScopes()` (for example to reach soft-deleted rows) must re-apply the tenant condition explicitly, with a comment saying why.

## WordPress REST API

### Registering Routes

```php
add_action('rest_api_init', static function (): void {
    register_rest_route('acme/v1', '/bookings', [
        [
            'methods' => WP_REST_Server::READABLE,
            'callback' => [BookingController::class, 'index'],
            'permission_callback' => static fn (): bool => current_user_can('read_bookings'),
            'args' => [
                'page' => ['type' => 'integer', 'default' => 1, 'minimum' => 1],
                'limit' => ['type' => 'integer', 'default' => 25, 'minimum' => 1, 'maximum' => 100],
            ],
        ],
        [
            'methods' => WP_REST_Server::CREATABLE,
            'callback' => [BookingController::class, 'store'],
            'permission_callback' => static fn (): bool => current_user_can('create_bookings'),
            'args' => [
                'email' => [
                    'type' => 'string',
                    'format' => 'email',
                    'required' => true,
                    'sanitize_callback' => 'sanitize_email',
                ],
                'date' => ['type' => 'string', 'format' => 'date', 'required' => true],
            ],
        ],
    ]);
});
```

| Rule | Why |
|------|-----|
| Namespace is `{vendor}/v{n}`, never `wp/v2` | `wp/v2` belongs to core; collisions break core routes |
| `permission_callback` is always present | Without it WordPress logs a `_doing_it_wrong` notice and the route is public |
| `__return_true` only for genuinely public, read-only routes, with a comment saying so | Makes "public" a visible, reviewable decision |
| Check a capability, not a role | Roles are configurable; capabilities are the contract |
| Every argument has a JSON Schema `type`; strings have `sanitize_callback` | Core validates schema-typed args before the callback runs |

### Responses and Errors

```php
final class BookingController
{
    public static function store(WP_REST_Request $request): WP_REST_Response|WP_Error
    {
        $booking = BookingService::create(
            email: $request->get_param('email'),
            date: $request->get_param('date'),
        );

        if ($booking instanceof WP_Error) {
            return $booking;
        }

        return new WP_REST_Response(['data' => BookingResource::toArray($booking)], 201);
    }
}

// Errors: code, message, and the HTTP status in data
return new WP_Error('booking_conflict', __('That slot is already booked.', 'acme'), ['status' => 409]);
```

WordPress serialises `WP_Error` as `{ "code", "message", "data" }`. Keep that native shape for WordPress routes (clients and core tooling expect it), but use the same machine-readable `code` values as the table above, lower-cased (`validation_failed`, `conflict`).

### Authentication

- Logged-in browser requests: cookie auth plus the `X-WP-Nonce` header (`wp_create_nonce('wp_rest')`); without the nonce the request runs as a logged-out user
- Server-to-server: **Application Passwords** over HTTPS only, one per integration so each can be revoked alone
- Never build a custom token scheme in a plugin when Application Passwords or an established auth plugin will do

## Rate Limiting

| Endpoint type | Limit |
|---------------|-------|
| Authentication (login, token, password reset) | 5 per minute per IP and per account |
| Authenticated reads | 120 per minute per user |
| Authenticated writes | 30 per minute per user |
| Public reads | 60 per minute per IP |

Laravel: `RateLimiter::for()` plus the `throttle` middleware. WordPress: enforce at the reverse proxy or with a transient/object-cache counter in the `permission_callback`; WordPress has no built-in REST rate limiter.

## Checklist

### API Design

- [ ] Every route is under a version prefix
- [ ] Plural, kebab-case resource paths; snake_case JSON keys
- [ ] Responses go through an API Resource (Laravel) or resource mapper (WordPress)
- [ ] List endpoints paginate, with a maximum `limit`
- [ ] One error envelope across the API

### Security

- [ ] Every route authenticates or is explicitly, visibly public
- [ ] Every action authorises (Policy / capability check)
- [ ] Writes validate with a Form Request or schema-typed `args`
- [ ] `validated()` only; `$fillable` on every model
- [ ] Nested and `exists` lookups scoped to the caller's tenant
- [ ] Rate limits on every group, stricter on auth endpoints
- [ ] `APP_DEBUG=false` / `WP_DEBUG_DISPLAY=false` outside local

### Documentation

- [ ] OpenAPI document generated from code and published
- [ ] Breaking changes follow [API Versioning](/standards/development/api-versioning)

### Authorisation and Data Scope

- [ ] Every endpoint applies the shared permission policy (lists, details, related lists, counts, search, exports, reports, bulk actions)
- [ ] Updates and deletes check the stored record's owners through the scoped set
- [ ] No tenant or user id taken from input; nested ids resolved through the scoped set
- [ ] Hidden records `404`, refused actions `403` with a reason, never `401`
- [ ] Export routes gated on the export permission
- [ ] Two-tenant isolation tests generated from the route list, run against the real database

## References

- [Laravel: API Resources](https://laravel.com/docs/eloquent-resources)
- [Laravel: Validation](https://laravel.com/docs/validation)
- [Laravel: Sanctum](https://laravel.com/docs/sanctum)
- [WordPress REST API Handbook](https://developer.wordpress.org/rest-api/)
- [WordPress: Adding custom endpoints](https://developer.wordpress.org/rest-api/extending-the-rest-api/adding-custom-endpoints/)
- [OWASP API Security Top 10](https://owasp.org/API-Security/)
