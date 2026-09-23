# WordPress Standards

> Authoritative standards for WordPress sites, themes and plugins: project layout, coding style, security, data access, performance, testing and deployment.

## Purpose

WordPress runs a large share of the web and is the most attacked PHP application there is. Most WordPress compromises come from custom or third-party theme and plugin code, not from core. These standards make custom WordPress code as safe, fast and maintainable as the rest of the estate.

They build on the [PHP Coding Standards](/standards/php/coding-standards). Where WordPress conventions differ (formatting, naming, escaping), this document wins for WordPress code.

## Core Principles

1. **Never edit core or third-party code**: extend through hooks, child themes and your own plugins; anything edited in place is lost on the next update
2. **Sanitise on input, escape on output, late**: every value that enters is sanitised and validated; every value that leaves is escaped at the moment it is printed
3. **Check intent and permission**: every state-changing request verifies a nonce (intent) and a capability (permission)
4. **Code in git, content in the database**: themes, plugins and configuration are versioned and deployed; posts, media and settings are not
5. **Fewer plugins, better plugins**: every plugin is attack surface and maintenance debt
6. **Treat WordPress as a framework**: use its APIs (Options, Settings, REST, HTTP, Filesystem, Cron) rather than reinventing them

## Project Types

| Type | Lives in | Use for |
|------|----------|---------|
| **mu-plugin** | `wp-content/mu-plugins/` | Site-specific behaviour that must always run and must not be deactivated from the admin (custom post types, integrations, hardening) |
| **Plugin** | `wp-content/plugins/{slug}/` | Reusable or optional functionality; anything distributed |
| **Block theme** | `wp-content/themes/{slug}/` | New sites: presentation via `theme.json`, templates and block patterns |
| **Classic / child theme** | `wp-content/themes/{slug}/` | Existing sites; child themes to modify a third-party parent theme |

**Rule**: themes contain presentation only. Custom post types, taxonomies, shortcodes, REST routes and integrations belong in a plugin or mu-plugin, so switching theme does not lose data or functionality.

## Repository Layout

Keep core and third-party plugins out of the repository; commit only the code you own.

```text
site/
├── wp-content/
│   ├── mu-plugins/
│   │   ├── acme-loader.php          # Loads the mu-plugin directories below
│   │   └── acme-core/               # Site behaviour: CPTs, hooks, integrations
│   │       ├── src/                 # PSR-4: Acme\Core\
│   │       └── acme-core.php
│   ├── plugins/
│   │   └── acme-bookings/           # Our own plugin(s) only
│   └── themes/
│       └── acme/                    # Our theme (or child theme)
├── composer.json                    # Dev tooling; optionally core and plugins via wpackagist
├── composer.lock
├── phpcs.xml.dist
├── phpstan.neon.dist
├── .wp-env.json                     # Local environment
└── .gitignore                       # Ignores core, uploads, third-party plugins, vendor
```

```gitignore
# .gitignore
/wp-admin/
/wp-includes/
/wp-*.php
/index.php
/xmlrpc.php
/wp-config.php
/wp-content/uploads/
/wp-content/upgrade/
/wp-content/cache/
/wp-content/plugins/*
!/wp-content/plugins/acme-*/
vendor/
node_modules/
```

For sites with several third-party plugins, manage them with Composer through [WPackagist](https://wpackagist.org/) so versions are pinned in `composer.lock` and reproducible. Premium plugins without a Composer source are vendored into a private Composer repository, not committed as loose files.

## Coding Style

WordPress code follows the [WordPress Coding Standards](https://developer.wordpress.org/coding-standards/wordpress-coding-standards/php/) (WPCS), enforced by PHP_CodeSniffer:

```xml
<?xml version="1.0"?>
<!-- phpcs.xml.dist -->
<ruleset name="Acme">
    <file>wp-content/mu-plugins/acme-core</file>
    <file>wp-content/plugins/acme-bookings</file>
    <file>wp-content/themes/acme</file>
    <exclude-pattern>*/vendor/*</exclude-pattern>
    <exclude-pattern>*/node_modules/*</exclude-pattern>
    <exclude-pattern>*/build/*</exclude-pattern>

    <arg name="extensions" value="php"/>
    <arg name="parallel" value="2"/>

    <rule ref="WordPress-Extra"/>
    <rule ref="WordPress-Docs"/>
    <rule ref="PHPCompatibilityWP"/>

    <config name="testVersion" value="8.3-"/>
    <config name="minimum_wp_version" value="6.6"/>

    <rule ref="WordPress.WP.I18n">
        <properties>
            <property name="text_domain" type="array">
                <element value="acme"/>
            </property>
        </properties>
    </rule>
    <rule ref="WordPress.NamingConventions.PrefixAllGlobals">
        <properties>
            <property name="prefixes" type="array">
                <element value="acme"/>
                <element value="Acme"/>
            </property>
        </properties>
    </rule>
</ruleset>
```

Set `minimum_wp_version` to the oldest WordPress release the code supports.

| Rule | Detail |
|------|--------|
| Naming | WPCS: `snake_case` functions, variables and hook names; `Class_Name` for classes in the global namespace |
| Namespaced OOP code | Allowed and preferred for anything beyond a few functions; PSR-4 autoload via Composer under `src/` |
| Prefix everything global | Functions, classes, constants, options, meta keys, hooks, transients, cron events, script handles: `acme_` / `Acme\` |
| `declare(strict_types=1)` | Required in namespaced `src/` files (strictness applies to calls your file makes, so core calling your hooks is unaffected) |
| Direct access guard | Entry files that run code on load start with `defined( 'ABSPATH' ) \|\| exit;` |
| Static analysis | PHPStan with `szepeviktor/phpstan-wordpress`, level 6 minimum, rising to 8 for new code |
| PHP version | Our own sites run a supported PHP (see the PHP standard); distributed plugins declare `Requires PHP` honestly |

## Hooks

```php
namespace Acme\Bookings;

final class Plugin
{
    public function register(): void
    {
        add_action( 'init', [ $this, 'register_post_types' ] );
        add_filter( 'the_content', [ $this, 'append_booking_form' ], 20 );
        add_action( 'acme_booking_confirmed', [ Notifications::class, 'send_confirmation' ], 10, 2 );
    }
}
```

| Rule | Why |
|------|-----|
| Register hooks in a `register()` method, not a constructor | Constructing an object for a test should not wire it into WordPress |
| Filters always return a value, of the type they received | A filter that returns nothing blanks the content for every later callback |
| Specify `$accepted_args` when a callback takes more than one argument | Otherwise only the first is passed |
| Use a priority other than 10 only with a comment saying why | Priority races are a common source of "works on staging" bugs |
| Expose your own extension points with prefixed `do_action` / `apply_filters` | Other code extends yours the same way you extend core |
| Avoid anonymous closures for hooks others may need to remove | A closure cannot be passed to `remove_action()` |
| Do work on the right hook: CPTs on `init`, assets on `wp_enqueue_scripts`, REST on `rest_api_init` | Running early (at file load) breaks translations, capabilities and conditional tags |

## Security

WordPress security failures are almost always one of four things: unescaped output (XSS), missing capability checks, missing nonces (CSRF) or unprepared SQL. Every review checks all four.

### Sanitise and Validate Input

```php
// ✅ Good: unslash, then sanitise for the expected type, then validate
$email = isset( $_POST['email'] ) ? sanitize_email( wp_unslash( $_POST['email'] ) ) : '';
if ( ! is_email( $email ) ) {
    wp_send_json_error( [ 'message' => __( 'Enter a valid email address.', 'acme' ) ], 422 );
}

$count  = absint( $_GET['count'] ?? 0 );
$status = sanitize_key( wp_unslash( $_GET['status'] ?? '' ) );
if ( ! in_array( $status, [ 'pending', 'confirmed' ], true ) ) {
    $status = 'pending';
}

// ❌ Bad: raw superglobal used directly
update_option( 'acme_email', $_POST['email'] );
```

| Input | Sanitiser |
|-------|-----------|
| Single-line text | `sanitize_text_field()` |
| Multi-line text | `sanitize_textarea_field()` |
| Email | `sanitize_email()` then `is_email()` |
| Integer ID | `absint()` |
| Key / slug | `sanitize_key()`, `sanitize_title()` |
| URL | `esc_url_raw()` / `sanitize_url()` |
| Limited HTML | `wp_kses_post()` or `wp_kses()` with an explicit allow-list |
| File name | `sanitize_file_name()` |
| Fixed set of values | Allow-list with `in_array( ..., true )` |

Always `wp_unslash()` superglobals before sanitising; WordPress adds slashes to them.

### Escape Output, Late

Escape at the point of printing, for the context, even if the value was sanitised on the way in:

```php
<a href="<?php echo esc_url( $booking_url ); ?>" class="<?php echo esc_attr( $class ); ?>">
    <?php echo esc_html( $title ); ?>
</a>
<script>
    const acmeConfig = <?php echo wp_json_encode( $config ); ?>;
</script>
<?php echo wp_kses_post( $trusted_rich_text ); ?>
<?php esc_html_e( 'Book now', 'acme' ); ?>
```

| Context | Function |
|---------|----------|
| HTML text | `esc_html()`, `esc_html__()`, `esc_html_e()` |
| HTML attribute | `esc_attr()`, `esc_attr__()` |
| URL (`href`, `src`) | `esc_url()` |
| Inside `<textarea>` | `esc_textarea()` |
| JavaScript data | `wp_json_encode()` or `wp_add_inline_script()` / `wp_localize_script()` |
| Rich HTML | `wp_kses_post()` |

Never escape early and store the escaped value; never `echo` a variable without an escaping function. PHPCS `WordPress.Security.EscapeOutput` enforces this; do not suppress it without a reason in the comment.

### Nonces and Capabilities

```php
// Form
wp_nonce_field( 'acme_cancel_booking_' . $booking_id, 'acme_nonce' );

// Handler
add_action( 'admin_post_acme_cancel_booking', static function (): void {
    $booking_id = absint( $_POST['booking_id'] ?? 0 );

    check_admin_referer( 'acme_cancel_booking_' . $booking_id, 'acme_nonce' );

    if ( ! current_user_can( 'edit_post', $booking_id ) ) {
        wp_die( esc_html__( 'You are not allowed to cancel this booking.', 'acme' ), 403 );
    }

    Bookings::cancel( $booking_id );
    wp_safe_redirect( admin_url( 'admin.php?page=acme-bookings&cancelled=1' ) );
    exit;
} );
```

| Rule | Why |
|------|-----|
| A nonce proves intent, not permission | Always pair `check_admin_referer()` / `check_ajax_referer()` with `current_user_can()` |
| Nonce action names include the object ID | A nonce for booking 12 cannot be replayed against booking 13 |
| Check capabilities, never roles | `current_user_can( 'manage_options' )`, not `in_array( 'administrator', $user->roles )` |
| Use meta capabilities with an object | `current_user_can( 'edit_post', $id )` respects ownership; `edit_posts` does not |
| Register custom capabilities for custom features | Map them to roles on activation; do not piggyback on `manage_options` for everything |
| `is_admin()` is not an auth check | It is `true` for any request to `/wp-admin/`, including `admin-ajax.php` from logged-out visitors |
| Redirect with `wp_safe_redirect()` | Prevents open redirects |

AJAX handlers registered with `wp_ajax_nopriv_` are public endpoints and are held to public-endpoint rules (validation, rate limiting). Prefer the REST API for new endpoints (see [PHP API Design](/standards/php/api-design)).

### Database Queries

```php
global $wpdb;

// ✅ Good: prepared, with placeholders and table name from $wpdb
$rows = $wpdb->get_results(
    $wpdb->prepare(
        "SELECT id, slot FROM {$wpdb->prefix}acme_bookings WHERE user_id = %d AND status = %s",
        $user_id,
        $status
    )
);

// ✅ Good: identifiers via %i
$wpdb->prepare( 'SELECT * FROM %i WHERE id = %d', $table, $id );

// ✅ Good: LIKE with escaping
$wpdb->prepare( "SELECT * FROM {$wpdb->posts} WHERE post_title LIKE %s", '%' . $wpdb->esc_like( $term ) . '%' );

// ❌ Bad: SQL injection
$wpdb->get_results( "SELECT * FROM {$wpdb->prefix}acme_bookings WHERE status = '$status'" );
```

Prefer core APIs (`WP_Query`, `get_posts()`, `get_post_meta()`, `WP_User_Query`) to raw SQL; they handle caching and escaping. Use `$wpdb` directly only for custom tables or measured performance needs.

### Files and Uploads

- Upload through `wp_handle_upload()` / `media_handle_upload()` and restrict types with the `upload_mimes` filter
- Validate the real type with `wp_check_filetype_and_ext()`; never trust the extension or `$_FILES['type']`
- Never `include` or `require` a path built from user input
- Write files through `WP_Filesystem`, never with `file_put_contents()` into the plugin directory
- Block PHP execution in `wp-content/uploads/` at the web server

### Hardened `wp-config.php`

```php
// Secrets come from the environment, never from the repository
define( 'DB_PASSWORD', getenv( 'WP_DB_PASSWORD' ) );
define( 'AUTH_KEY', getenv( 'WP_AUTH_KEY' ) );
// ... all eight keys and salts

define( 'WP_ENVIRONMENT_TYPE', getenv( 'WP_ENVIRONMENT_TYPE' ) ?: 'production' );

define( 'DISALLOW_FILE_EDIT', true );      // No theme/plugin editor in the admin
define( 'DISALLOW_FILE_MODS', true );      // No installs or updates from the admin: code arrives via deploy
define( 'FORCE_SSL_ADMIN', true );
define( 'WP_AUTO_UPDATE_CORE', 'minor' );  // Security releases apply automatically

define( 'WP_DEBUG', false );
define( 'WP_DEBUG_DISPLAY', false );
define( 'WP_DEBUG_LOG', '/var/log/wordpress/acme-debug.log' ); // Outside the web root

define( 'DISABLE_WP_CRON', true );         // Cron runs from the system scheduler instead
```

Also:

- Disable XML-RPC (`add_filter( 'xmlrpc_enabled', '__return_false' );` and block `/xmlrpc.php` at the proxy) unless a client needs it
- Stop user enumeration: block `?author=N` scans and restrict the `/wp/v2/users` endpoint for logged-out requests
- Enforce two-factor authentication for every account with `edit_posts` or above
- Rate-limit `wp-login.php` at the reverse proxy
- Follow [Security Guidelines](/standards/governance/security-guidelines) and [Infrastructure Security](/standards/governance/infrastructure-security) for everything else

## Data Storage

| Data | Store in | Notes |
|------|----------|-------|
| Site settings | Options API | Group related settings into one array option; set `autoload` to `false` for anything not needed on every request |
| Per-post data | Post meta | `register_post_meta()` with `type`, `single`, `sanitize_callback`, `auth_callback`, and `show_in_rest` if the editor needs it |
| Per-user data | User meta | Same rules as post meta |
| Content-like records | Custom post type | Gets revisions, REST, admin UI and caching for free |
| High-volume or relational records (bookings, logs, orders) | Custom table | Created with `dbDelta()` in an activation or upgrade routine, versioned with a schema option |
| Cached computed values | Transients / object cache | Always set an expiry; code must work when the cache is empty |

- Every autoloaded option is loaded on every request; keep the total autoloaded size under 800 KB and audit it with `wp option list --autoload=on`
- Do not query on `meta_value` for large post counts; `postmeta` is not indexed for it. Use a taxonomy or a custom table
- Custom table schema and naming follow [Database Standards](/standards/governance/database-standards)
- Every plugin that creates data ships an `uninstall.php` that removes its options, tables, cron events and capabilities

## Performance

```php
// ✅ Good: bounded, lean query
$query = new WP_Query( [
    'post_type'              => 'acme_booking',
    'posts_per_page'         => 20,
    'no_found_rows'          => true,   // Skip SQL_CALC_FOUND_ROWS when not paginating
    'update_post_term_cache' => false,  // Skip term priming when terms are not used
    'fields'                 => 'ids',  // When only IDs are needed
] );

// ❌ Bad: unbounded
get_posts( [ 'post_type' => 'acme_booking', 'posts_per_page' => -1 ] );
```

| Rule | Why |
|------|-----|
| Never `posts_per_page => -1` on a front-end request | Grows without limit as content grows |
| No remote HTTP calls during page render | Cache the result in a transient, refreshed by cron |
| Remote calls use `wp_remote_get()` with an explicit `timeout` | The default can hold a PHP worker for seconds |
| Persistent object cache (Redis) on every production site | Without it, transients and the object cache live in the database |
| Full-page cache for logged-out traffic | At the proxy or a caching plugin; purge on publish |
| Load assets only where they are used | Conditional `wp_enqueue_script()`; `strategy => 'defer'` for scripts |
| Images through core's responsive image functions | `wp_get_attachment_image()` emits `srcset`, `sizes` and lazy loading |

Performance targets come from [Performance Budgets](/standards/quality/performance-budgets).

## Assets and Blocks

- Register and enqueue every script and style; never hard-code `<script>` or `<link>` tags
- Version assets with the build hash from `@wordpress/scripts` (`*.asset.php`), not `time()`
- Build blocks with `@wordpress/scripts` and `block.json`; register with `register_block_type( __DIR__ . '/build/blocks/booking-form' )`
- Block themes define colours, typography and spacing in `theme.json`; do not duplicate them in CSS
- Prefer dynamic blocks (server-rendered with `render.php`) for anything that shows live data, so markup changes do not invalidate saved content

## Internationalisation

- Every user-facing string goes through a translation function with the project text domain: `__( 'Book now', 'acme' )`
- Use `sprintf()` with placeholders, never concatenation: `sprintf( __( '%d bookings', 'acme' ), $count )`, and `_n()` for plurals
- Add a translator comment for every placeholder: `/* translators: %d: number of bookings */`
- Translate in JavaScript with `@wordpress/i18n` and `wp_set_script_translations()`
- Load translations on `init` or later, never at file load
- Wider rules in [Internationalisation](/standards/development/internationalization)

## Scheduled Tasks

- Set `DISABLE_WP_CRON` and run `wp cron event run --due-now` from the system scheduler every minute; WP-Cron otherwise only fires on page visits
- Schedule on activation with a `wp_next_scheduled()` guard; unschedule on deactivation
- Use **Action Scheduler** for queues, retries or high volume, rather than stacking WP-Cron events
- Long jobs are idempotent and chunked; a job that dies half-way must be safe to run again

## Local Development and Tooling

| Tool | Use |
|------|-----|
| `@wordpress/env` (`wp-env`) | Docker-based local site, configured in `.wp-env.json` and committed |
| WP-CLI | Scripted admin: search-replace, cache flush, user and option management, data migrations |
| Query Monitor | Local and staging only: slow queries, hooks, HTTP calls |
| `@wordpress/scripts` | Build, lint and test JavaScript and blocks |

Data migrations are WP-CLI commands or versioned upgrade routines keyed on a stored schema version, never a one-off script run in the browser. Use `wp search-replace` (never a raw SQL replace) when moving a database between domains; it handles serialised data.

## Testing

| Level | Tool | Covers |
|-------|------|--------|
| Unit | PHPUnit or Pest with **Brain Monkey** / **WP_Mock** | Pure logic in `src/` without loading WordPress |
| Integration | PHPUnit with the WordPress test suite (`wp-env run tests-cli`, `WP_UnitTestCase`) | Hooks, queries, REST routes, capabilities against a real WordPress |
| End-to-end | Playwright with `@wordpress/e2e-test-utils-playwright` | Editor flows, forms, checkout |

Integration tests for every REST route and form handler cover: logged out, logged in without the capability, invalid nonce, invalid input and the happy path. Coverage targets follow [Testing Strategy](/standards/quality/testing-strategy).

## Preflight

```json
{
  "scripts": {
    "lint": "phpcs",
    "lint:fix": "phpcbf",
    "analyse": "phpstan analyse --memory-limit=1G",
    "test": "phpunit",
    "preflight": ["@composer validate --strict", "@lint", "@analyse", "@test", "@composer audit"]
  }
}
```

Run `npm run lint:js` and `npm run lint:css` from `@wordpress/scripts` alongside it when the project has a front-end build. The preflight is wired to the pre-push hook as described in [CI Preflight Strategy](/standards/quality/ci-preflight-strategy).

## Deployment and Updates

- Deploy themes and our own plugins by rsync from CI (see the WordPress section of [CI/CD Pipelines](/standards/development/ci-cd-pipelines)); never edit files on the server or through the admin
- Build assets in CI; deploy the `build/` output, not `node_modules/`
- Reset OPcache or reload PHP-FPM after each deploy when `opcache.validate_timestamps=0`
- Take a database backup before any deploy that runs an upgrade routine (see [Backup and Disaster Recovery](/standards/reliability/backup-disaster-recovery))
- Core minor releases apply automatically; core majors and plugin updates go through staging first, on a regular cadence, with a changelog review
- Remove plugins and themes that are deactivated; inactive code is still reachable and still exploitable
- Subscribe to vulnerability feeds (WPScan, Patchstack or Wordfence Intelligence) for every installed plugin and theme

### Choosing Third-Party Plugins

Before installing a plugin, confirm:

- [ ] Updated within the last six months and tested with the current WordPress major
- [ ] No unpatched entries in a vulnerability database
- [ ] Active installs and support threads show a maintained project
- [ ] It does one job; we are not installing a suite for one feature
- [ ] We could not do it in under a day of our own code, with less risk

## Checklist

### Structure

- [ ] Core, uploads and third-party plugins are not in git
- [ ] Behaviour lives in plugins / mu-plugins; the theme is presentation only
- [ ] Everything global is prefixed; namespaced code autoloads from `src/`
- [ ] Every data-creating plugin has `uninstall.php`

### Security

- [ ] Every superglobal read is unslashed, sanitised and validated
- [ ] Every echo is escaped for its context, at output time
- [ ] Every state change checks a nonce **and** a capability
- [ ] Every REST route has a real `permission_callback`
- [ ] Every `$wpdb` query with variables uses `prepare()`
- [ ] `wp-config.php` secrets from the environment; `DISALLOW_FILE_EDIT` and `DISALLOW_FILE_MODS` set
- [ ] Debug output off in production; debug log outside the web root
- [ ] XML-RPC disabled and login rate-limited
- [ ] 2FA for editors and above

### Performance

- [ ] No unbounded queries on front-end requests
- [ ] No uncached remote calls during render
- [ ] Autoloaded options under 800 KB
- [ ] Persistent object cache and page cache in production
- [ ] System cron instead of WP-Cron

### Quality

- [ ] PHPCS (WordPress-Extra, WordPress-Docs, PHPCompatibilityWP) passes
- [ ] PHPStan with the WordPress extension passes
- [ ] Integration tests cover permission and nonce failures
- [ ] All strings translatable with the project text domain

## References

- [WordPress Developer Resources](https://developer.wordpress.org/)
- [WordPress PHP Coding Standards](https://developer.wordpress.org/coding-standards/wordpress-coding-standards/php/)
- [Plugin Handbook: Security](https://developer.wordpress.org/plugins/security/)
- [Common APIs: Escaping](https://developer.wordpress.org/apis/security/escaping/)
- [Common APIs: Sanitising](https://developer.wordpress.org/apis/security/sanitizing/)
- [Theme Handbook: theme.json](https://developer.wordpress.org/themes/global-settings-and-styles/)
- [Block Editor Handbook](https://developer.wordpress.org/block-editor/)
- [WPCS on GitHub](https://github.com/WordPress/WordPress-Coding-Standards)
- [phpstan-wordpress](https://github.com/szepeviktor/phpstan-wordpress)
- [WP-CLI](https://wp-cli.org/)
- [OWASP WordPress Security Implementation Guideline](https://owasp.org/www-project-wordpress-security-implementation-guideline/)
