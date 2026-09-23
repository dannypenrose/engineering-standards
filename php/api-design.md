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

## References

- [Laravel: API Resources](https://laravel.com/docs/eloquent-resources)
- [Laravel: Validation](https://laravel.com/docs/validation)
- [Laravel: Sanctum](https://laravel.com/docs/sanctum)
- [WordPress REST API Handbook](https://developer.wordpress.org/rest-api/)
- [WordPress: Adding custom endpoints](https://developer.wordpress.org/rest-api/extending-the-rest-api/adding-custom-endpoints/)
- [OWASP API Security Top 10](https://owasp.org/API-Security/)
