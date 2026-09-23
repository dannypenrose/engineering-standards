# PHP Coding Standards

> Authoritative PHP coding standards aligned with PER Coding Style and the PSR family, for modern, strictly typed PHP across Laravel, WordPress and framework-free projects.

## Purpose

Establish consistent PHP coding practices that produce code which is typed, statically analysable, testable and safe by default. These rules apply to every PHP project. WordPress projects also follow the [WordPress Standards](/standards/php/wordpress), which override this document where WordPress core conventions differ.

## Core Principles

1. **Strict types everywhere**: every file declares `strict_types=1`, so type juggling bugs surface as errors instead of silent coercions
2. **Types over docblocks**: native parameter, return and property types come first; docblocks only add what the type system cannot express (generics, array shapes)
3. **Static analysis is a gate**: PHPStan runs in the preflight and a new error blocks the push
4. **Immutable by default**: `readonly` properties and classes unless mutation is the point
5. **Composition over inheritance**: small final classes wired by constructor injection
6. **Fail loudly at the boundary**: validate input where it enters the system, then trust your types inside

## Supported Versions

| Item | Standard |
|------|----------|
| Minimum PHP | 8.3 for new projects |
| Target PHP | The newest release with active support |
| End-of-life PHP | Never deploy to a PHP version past its security support date |
| Declared in | `composer.json` `require.php` and `config.platform.php` |

Pin the platform version so Composer resolves dependencies for production's PHP, not the developer's laptop:

```json
{
  "require": {
    "php": "^8.3"
  },
  "config": {
    "platform": {
      "php": "8.3.0"
    },
    "sort-packages": true,
    "optimize-autoloader": true
  }
}
```

## Style Guide

### Formatting

Follow [PER Coding Style 2.0](https://www.php-fig.org/per/coding-style/) (the successor to PSR-12). Enforce it with a tool, never by review comment:

| Project type | Formatter |
|--------------|-----------|
| Laravel | **Laravel Pint** (`pint.json`, `per` preset) |
| Framework-free / Symfony | **PHP CS Fixer** (`@PER-CS` rule set) |
| WordPress | **PHP_CodeSniffer** with WPCS (see [WordPress Standards](/standards/php/wordpress)) |

```json
// pint.json
{
  "preset": "per",
  "rules": {
    "declare_strict_types": true,
    "final_class": true,
    "ordered_imports": { "sort_algorithm": "alpha" },
    "no_unused_imports": true,
    "strict_comparison": true,
    "void_return": true
  }
}
```

### File Header

Every PHP file starts the same way:

```php
<?php

declare(strict_types=1);

namespace App\Billing;

use App\Billing\Exceptions\PaymentFailedException;
use Psr\Log\LoggerInterface;
```

- One class, interface, trait or enum per file
- No closing `?>` tag in PHP-only files (trailing whitespace after it leaks into the response)
- Files that declare symbols do not also cause side effects (PSR-1)

### Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Namespaces | PascalCase, PSR-4 mapped | `App\Billing\Invoices` |
| Classes / interfaces / traits / enums | PascalCase | `InvoiceService`, `PaymentGateway` |
| Interfaces | Noun, no `I` prefix; `Interface` suffix only if a same-named class exists | `PaymentGateway` |
| Methods / functions | camelCase | `calculateTotal()` |
| Properties / variables | camelCase | `$lineItems` |
| Constants / enum cases | UPPER_SNAKE for constants, PascalCase for enum cases | `MAX_RETRIES`, `Status::Paid` |
| Files | Match the class name exactly | `InvoiceService.php` |
| Config keys / DB columns | snake_case | `created_at` |

### Comparisons

Always compare strictly. Loose comparison is the source of a whole class of PHP bugs (`"abc" == 0` was `true` before PHP 8, and `in_array()` is still loose by default):

```php
// ✅ Good
if ($status === Status::Paid) { ... }
if (in_array($role, $allowedRoles, true)) { ... }

// ❌ Bad
if ($status == 'paid') { ... }
if (in_array($role, $allowedRoles)) { ... }
```

## Type System

### Native Types First

```php
// ✅ Good: fully typed, readonly, constructor promotion
final readonly class Money
{
    public function __construct(
        public int $amountInMinorUnits,
        public Currency $currency,
    ) {}

    public function add(self $other): self
    {
        if ($other->currency !== $this->currency) {
            throw new CurrencyMismatchException($this->currency, $other->currency);
        }

        return new self($this->amountInMinorUnits + $other->amountInMinorUnits, $this->currency);
    }
}

// ❌ Bad: untyped, mutable, float for money
class Money
{
    public $amount;
    public $currency;
}
```

### Rules

- Store money as integer minor units (pence, cents), never `float`
- Use `?Type` or `Type|null` rather than untyped parameters; avoid `mixed` unless the value really is anything
- Return `void` or `never` explicitly where applicable
- Use backed enums instead of class constants or magic strings for a fixed set of values
- Use `#[\Override]` on methods that override a parent, so a renamed parent method fails loudly
- Use docblock generics and array shapes for what native types cannot say:

```php
/**
 * @param list<LineItem> $items
 * @return array{subtotal: int, tax: int, total: int}
 */
public function summarise(array $items): array
```

Prefer a typed collection or value object over a long array shape; if the shape has more than three or four keys, it wants to be a class.

### Enums

```php
enum InvoiceStatus: string
{
    case Draft = 'draft';
    case Issued = 'issued';
    case Paid = 'paid';
    case Void = 'void';

    public function isFinal(): bool
    {
        return match ($this) {
            self::Paid, self::Void => true,
            self::Draft, self::Issued => false,
        };
    }
}
```

Use `match` over `switch`: it compares strictly, returns a value and throws `UnhandledMatchError` when a case is missing.

## Static Analysis

**PHPStan** is mandatory. New projects start at level 8 or higher; legacy projects adopt a baseline and never grow it.

```neon
# phpstan.neon.dist
includes:
    - phpstan-baseline.neon
    # Laravel: - vendor/larastan/larastan/extension.neon
    # WordPress: - vendor/szepeviktor/phpstan-wordpress/extension.neon

parameters:
    level: 8
    paths:
        - src
        - tests
    treatPhpDocTypesAsCertain: false
    reportUnmatchedIgnoredErrors: true
```

| Rule | Why |
|------|-----|
| Baseline shrinks, never grows | A growing baseline hides new bugs behind old ones |
| `@phpstan-ignore` needs an identifier and a reason | Blanket ignores silence errors you have not read |
| Framework extension installed | Larastan and phpstan-wordpress teach PHPStan about magic the framework does |

Use **Rector** for automated upgrades (PHP version bumps, framework majors). Run it as a reviewed, one-off change on its own branch, never mixed with feature work.

## Project Structure

### Framework-Free / Library Layout

```text
project/
├── src/                      # PSR-4 root: App\
│   ├── Domain/               # Entities, value objects, domain services
│   ├── Application/          # Use cases / actions
│   ├── Infrastructure/       # Database, HTTP clients, queues
│   └── Http/                 # Controllers, middleware, requests
├── tests/
│   ├── Unit/
│   ├── Integration/
│   └── Feature/
├── config/
├── public/                   # Web root: index.php and assets only
├── composer.json
├── composer.lock             # Always committed for applications
├── phpstan.neon.dist
└── phpunit.xml.dist
```

The web server's document root is `public/`, never the project root. Nothing outside `public/` is reachable over HTTP.

### Laravel Layout

Follow the default Laravel skeleton and add these conventions:

```text
app/
├── Actions/                  # Single-purpose use cases: CreateInvoice, RefundPayment
├── Enums/
├── Http/
│   ├── Controllers/          # Thin: validate, call an action, return a resource
│   ├── Requests/             # FormRequest per write endpoint
│   └── Resources/            # API resources (response shape)
├── Models/                   # Eloquent models: relationships, casts, scopes only
├── Policies/                 # Authorisation per model
├── Services/                 # Integrations with external systems
└── ValueObjects/
```

| Rule | Why |
|------|-----|
| Controllers do not contain business logic | Logic in controllers cannot be reused from jobs, commands or tests |
| Models do not call external services | Model events that send email or hit APIs make every save a side effect |
| One action class, one `handle()` / `__invoke()` method | Keeps use cases findable and independently testable |
| No `env()` outside `config/` | `env()` returns `null` once config is cached in production |
| No facades inside domain classes | Inject the contract instead, so the class is testable without the container |

## Code Patterns

### Dependency Injection

```php
// ✅ Good: dependencies injected through the constructor, against interfaces
final readonly class IssueInvoice
{
    public function __construct(
        private InvoiceRepository $invoices,
        private PdfRenderer $pdf,
        private LoggerInterface $logger,
    ) {}

    public function handle(InvoiceId $id): Invoice
    {
        $invoice = $this->invoices->get($id)->issue();
        $this->invoices->save($invoice);
        $this->pdf->render($invoice);

        $this->logger->info('Invoice issued', ['invoice_id' => $id->value]);

        return $invoice;
    }
}

// ❌ Bad: hidden dependencies, untestable
class IssueInvoice
{
    public function handle($id)
    {
        $invoice = DB::table('invoices')->find($id);
        Mail::to($invoice->email)->send(new InvoiceMail($invoice));
    }
}
```

Mark classes `final` by default. Remove `final` deliberately when a class is designed for extension, not to make mocking easier (mock the interface instead).

### Exception Handling

```php
// ✅ Good: a domain exception hierarchy
abstract class DomainException extends \RuntimeException {}

final class InvoiceNotFoundException extends DomainException
{
    public static function withId(InvoiceId $id): self
    {
        return new self("Invoice {$id->value} was not found.");
    }
}

// ✅ Good: catch what you can handle, keep the chain
try {
    $this->gateway->charge($payment);
} catch (GatewayTimeoutException $e) {
    throw new PaymentFailedException('Payment gateway timed out.', previous: $e);
}

// ❌ Bad: swallowing everything
try {
    $this->gateway->charge($payment);
} catch (\Exception $e) {
    // ignore
}
```

- Never use `@` to suppress errors; handle the failure explicitly
- Convert warnings to exceptions in framework-free apps (`set_error_handler`), so a failed `file_get_contents()` cannot pass silently
- Never show stack traces to users: `display_errors=Off` in production, log instead

### Database Access

- Use the framework's query builder or ORM, or PDO prepared statements; never concatenate input into SQL
- Configure PDO with `PDO::ERRMODE_EXCEPTION` and `PDO::ATTR_EMULATE_PREPARES => false`
- Eager-load relationships to avoid N+1 queries (`with()` in Eloquent); enable `Model::preventLazyLoading()` outside production
- Wrap multi-step writes in a transaction

```php
// ✅ Good
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = :email');
$stmt->execute(['email' => $email]);

// ❌ Bad: SQL injection
$pdo->query("SELECT * FROM users WHERE email = '$email'");
```

Schema, naming and migration rules come from [Database Standards](/standards/governance/database-standards).

### Output and Escaping

Escape at the point of output, for the context you are writing into. Blade's `{{ }}` escapes HTML; `{!! !!}` does not and needs a comment justifying it. In plain PHP templates use `htmlspecialchars($value, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8')`. WordPress has its own escaping functions (see [WordPress Standards](/standards/php/wordpress)).

### Things Never to Use

| Construct | Why | Use instead |
|-----------|-----|-------------|
| `eval()`, `create_function()` | Arbitrary code execution | Closures, strategy classes |
| `extract()` | Invents variables from input, defeats static analysis | Explicit array access |
| `unserialize()` on untrusted data | Object injection | `json_decode()` with `JSON_THROW_ON_ERROR` |
| `md5()` / `sha1()` for passwords | Trivially cracked | `password_hash()` / `password_verify()` |
| `rand()` / `mt_rand()` for tokens | Predictable | `random_bytes()`, `random_int()`, `bin2hex(random_bytes(32))` |
| `global` variables | Hidden coupling | Constructor injection |
| `$_GET` / `$_POST` directly in business logic | Unvalidated input deep in the system | Request objects validated at the boundary |
| `exit` / `die` in library code | Kills the process mid-request, untestable | Throw an exception |

## Dependency Management

- **Composer** only; never commit `vendor/`
- Commit `composer.lock` for applications; libraries do not commit it
- Production installs use `composer install --no-dev --optimize-autoloader --classmap-authoritative`
- Run `composer audit` in the preflight and in scheduled CI
- Run `composer validate --strict` in the preflight
- Use `composer bump` deliberately, not as a habit; follow [Dependency Management](/standards/development/dependency-management)

```json
{
  "scripts": {
    "lint": "pint --test",
    "analyse": "phpstan analyse --memory-limit=1G",
    "test": "pest --parallel --processes=2",
    "preflight": [
      "@composer validate --strict",
      "@lint",
      "@analyse",
      "@test",
      "@composer audit"
    ]
  }
}
```

Bound parallel test workers (`--processes=2`) as the testing standard requires; an unbounded run starves the machine and reports false failures.

## Testing

Use **Pest** for new projects (it runs on PHPUnit, so PHPUnit knowledge and tooling carry over). Existing PHPUnit suites stay on PHPUnit; do not mix styles in one suite.

### Test Structure

```php
// tests/Unit/Billing/MoneyTest.php
use App\Billing\Currency;
use App\Billing\Exceptions\CurrencyMismatchException;
use App\Billing\Money;

describe('Money', function () {
    it('adds amounts in the same currency', function () {
        $total = (new Money(1_000, Currency::GBP))->add(new Money(250, Currency::GBP));

        expect($total->amountInMinorUnits)->toBe(1_250);
    });

    it('refuses to add different currencies', function () {
        (new Money(1_000, Currency::GBP))->add(new Money(250, Currency::EUR));
    })->throws(CurrencyMismatchException::class);
});
```

### Laravel Feature Tests

```php
use App\Models\User;

it('lets an owner view their invoice', function () {
    $user = User::factory()->hasInvoices(1)->create();

    $this->actingAs($user)
        ->getJson("/api/v1/invoices/{$user->invoices->first()->id}")
        ->assertOk()
        ->assertJsonPath('data.status', 'draft');
});

it('hides another user\'s invoice', function () {
    $invoice = Invoice::factory()->create();

    $this->actingAs(User::factory()->create())
        ->getJson("/api/v1/invoices/{$invoice->id}")
        ->assertNotFound();
});
```

- Use `RefreshDatabase` (or `LazilyRefreshDatabase`) against the same database engine as production, not SQLite, when queries use engine-specific features
- Use `Http::fake()`, `Queue::fake()`, `Mail::fake()` rather than hand-rolled mocks for framework services
- Add architecture tests (`arch()->expect('App\Models')->not->toUse('Illuminate\Support\Facades\Http')`) to hold the layering rules above
- Coverage targets and test levels come from [Testing Strategy](/standards/quality/testing-strategy)

## Runtime Configuration

| Setting | Production value | Why |
|---------|------------------|-----|
| `display_errors` | `Off` | Errors must not leak paths or data to users |
| `log_errors` | `On` | Errors still need to be seen |
| `expose_php` | `Off` | Do not advertise the version in headers |
| `opcache.enable` | `1` | Order-of-magnitude speed-up |
| `opcache.validate_timestamps` | `0` (reset OPcache on deploy) | No `stat()` per request; deploys must reload PHP-FPM or reset OPcache |
| `session.cookie_secure` / `httponly` / `samesite` | `1` / `1` / `Lax` | Session cookie hardening |
| `memory_limit` | Set explicitly per pool | A runaway request should fail, not take the host down |

Run PHP under **PHP-FPM** with a pool per application and a dedicated system user, following [Self-Hosting Standards](/standards/governance/self-hosting-standards).

## Checklist

### Project Setup

- [ ] `composer.json` pins `require.php` and `config.platform.php`
- [ ] `composer.lock` committed (applications)
- [ ] Formatter configured (Pint, PHP CS Fixer or PHPCS) and run in the preflight
- [ ] PHPStan at level 8+ (or a shrinking baseline) with the framework extension
- [ ] `composer preflight` script covers validate, lint, analyse, test and audit
- [ ] Document root is `public/`

### Code Quality

- [ ] Every file has `declare(strict_types=1);`
- [ ] Native types on every parameter, return and property
- [ ] Strict comparisons (`===`, `in_array(..., true)`)
- [ ] Classes `final` and `readonly` unless designed otherwise
- [ ] No `eval`, `extract`, `@` suppression, or `unserialize` on untrusted data
- [ ] No `env()` outside config files (Laravel)
- [ ] Money stored as integer minor units

### Testing

- [ ] Pest (new) or PHPUnit (existing) with bounded parallel workers
- [ ] Feature tests cover authorisation failures, not just the happy path
- [ ] Architecture tests enforce layering
- [ ] External services faked, never called

## References

- [PER Coding Style](https://www.php-fig.org/per/coding-style/)
- [PHP-FIG PSRs](https://www.php-fig.org/psr/)
- [PHP supported versions](https://www.php.net/supported-versions.php)
- [PHPStan](https://phpstan.org/)
- [Rector](https://getrector.com/)
- [Pest](https://pestphp.com/)
- [Laravel documentation](https://laravel.com/docs)
- [OWASP PHP Configuration Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/PHP_Configuration_Cheat_Sheet.html)
