# PHP Standards

Implementation standards for PHP projects, including Laravel applications, WordPress sites, themes and plugins, and framework-free PHP.

## Purpose

These standards provide concrete implementations of universal engineering principles using PHP-specific patterns, tools and best practices aligned with the PHP-FIG PSRs and PER Coding Style.

## Standards Index

| Standard | Description |
| -------- | ----------- |
| [Coding Standards](/standards/php/coding-standards) | Modern PHP conventions: strict types, static analysis, project structure, patterns, testing |
| [API Design](/standards/php/api-design) | Laravel and WordPress REST API design, validation, resources, errors, authentication |
| [WordPress](/standards/php/wordpress) | WordPress themes, plugins and sites: layout, WPCS, security, data, performance, deployment |

## Which Standard Applies

| Project | Read |
|---------|------|
| Laravel application or API | Coding Standards, then API Design for HTTP work |
| WordPress theme, plugin or site | WordPress, then Coding Standards for namespaced `src/` code, then API Design for REST routes |
| Framework-free PHP or library | Coding Standards |

## Tech Stack Coverage

### Frameworks and Platforms

- **Laravel** - Full-stack framework and API backends
- **WordPress** - Block and classic themes, plugins, mu-plugins
- **Framework-free PHP** - Libraries and small services on PSR components

### Language and Runtime

- **PHP 8.3+** - Strict types, enums, readonly classes
- **PHP-FPM** - Process manager behind Caddy or Nginx
- **OPcache** - Bytecode cache, reset on deploy

### Tooling

- **Composer** - Dependency management and scripts
- **PHPStan** - Static analysis (Larastan, phpstan-wordpress)
- **Laravel Pint / PHP CS Fixer** - Formatting to PER Coding Style
- **PHP_CodeSniffer + WPCS** - WordPress coding standards
- **Rector** - Automated upgrades
- **Pest / PHPUnit** - Testing
- **WP-CLI / wp-env** - WordPress tooling and local environments

## Universal Standards Reference

These PHP implementations build upon:

- [Security Guidelines](/standards/governance/security-guidelines) - Authentication, authorisation, input validation
- [Database Standards](/standards/governance/database-standards) - Schema design, migrations, indexing, naming conventions
- [Testing Strategy](/standards/quality/testing-strategy) - Unit, integration, E2E testing patterns
- [Observability Standards](/standards/reliability/observability-standards) - Logging, metrics, tracing
- [Performance Budgets](/standards/quality/performance-budgets) - Page and API response times
- [CI/CD Pipelines](/standards/development/ci-cd-pipelines) - Deploy workflow templates, including WordPress

## Quick Reference

### Every PHP File

```php
<?php

declare(strict_types=1);

namespace App\Billing;
```

### Preflight

```bash
composer validate --strict
composer lint       # Pint / PHP CS Fixer / PHPCS
composer analyse    # PHPStan
composer test       # Pest / PHPUnit, bounded workers
composer audit
```

### WordPress Security in Four Lines

```php
$title = sanitize_text_field( wp_unslash( $_POST['title'] ?? '' ) );   // Sanitise input
check_admin_referer( 'acme_save_' . $post_id, 'acme_nonce' );          // Verify intent
if ( ! current_user_can( 'edit_post', $post_id ) ) { wp_die( '', 403 ); } // Verify permission
echo esc_html( $title );                                                // Escape output
```
