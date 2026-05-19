# PostHog Product Analytics Standards

> Authoritative configuration and instrumentation standards for PostHog across all projects. Complements [observability-standards](./observability-standards.md) — that doc covers server-side telemetry; this one covers user-facing product analytics.

## Purpose

Provide a reusable, low-noise PostHog setup that captures **meaningful product actions** without drowning teams in autocapture noise, accidentally polluting production data, or leaking PII into telemetry.

## Core Principles

1. **Manual instrumentation over autocapture.** A small set of well-named, deliberate events beats thousands of `clicked p with text "X"` rows every time.
2. **Production-only capture by default.** Staging and local dev send nothing to PostHog unless explicitly opted-in.
3. **One identity, two surfaces.** Frontend and backend agree on the same `distinct_id` — typically the user's primary key.
4. **Groups for organisations.** Tenants/organisations are tracked as PostHog `groups` so dashboards can be sliced by customer.
5. **No PII in event props.** IDs, counts, enums, durations — never free-text input, emails, or names that aren't already person properties on `identify`.
6. **Safety nets on by default.** Key-mismatch detection, env-tagged events, and dedupe sentinels are mandatory.

---

## Environment & Region

### Pick a region and commit to it

PostHog projects are **region-locked at creation**. A project on EU Cloud cannot be moved to US Cloud (and vice versa) — only recreated.

**Default to EU Cloud.** Our customers and operations are UK/EU-based, so EU Cloud keeps data residency aligned with GDPR and lowers ingest latency for the majority of users. US Cloud is only justified when a project's customer base is predominantly North American.

| Region | Dashboard host | Ingest host | Asset host |
|---|---|---|---|
| **EU Cloud (default)** | `https://eu.posthog.com` | `https://eu.i.posthog.com` | `https://eu-assets.i.posthog.com` |
| US Cloud | `https://us.posthog.com` | `https://us.i.posthog.com` | `https://us-assets.i.posthog.com` |

> **The ingest host always contains `i.`** — the dashboard host (`eu.posthog.com`, `us.posthog.com`) is *not* the ingest endpoint. Pointing the SDK at the dashboard host produces silent 4xx failures and no captured data.

### Production-only deployment posture

| Environment | Capture | How |
|---|---|---|
| Local dev | **OFF** by default | Env var unset or `false`; no API key |
| Staging | **OFF** | CI workflow sets `KEY=''` and `ENABLED=false`, backend `Enabled=false` |
| Production | **ON** | CI injects the prod key + `ENABLED=true`; backend App Settings enable capture |

Frontend gates capture on **both** an explicit opt-in flag *and* a non-empty key, so removing either kills capture without code changes.

### Required environment variables

**Frontend (Next.js / Vite / similar):**

| Var | Purpose |
|---|---|
| `NEXT_PUBLIC_POSTHOG_KEY` (or framework equivalent) | Project API key |
| `NEXT_PUBLIC_POSTHOG_HOST` | Ingest host (default `https://eu.i.posthog.com`) |
| `NEXT_PUBLIC_POSTHOG_ENABLED` | Master kill-switch (`true`/`false`) |
| `NEXT_PUBLIC_POSTHOG_ENV` | Environment tag attached to every event |
| `NEXT_PUBLIC_APP_VERSION` | Release version super-property (commonly the git SHA) |

> `NEXT_PUBLIC_*` (and equivalents like `VITE_`) are **inlined at build time**. Changing a CI secret does not affect deployed bundles until a fresh build runs.

**Backend (.NET / Node / Python):**

```json
{
  "PostHog": {
    "ProjectApiKey": "",
    "HostUrl": "https://eu.i.posthog.com",
    "Enabled": false,
    "Environment": "development",
    "AppVersion": ""
  }
}
```

Backend reads at runtime from environment config, so changes take effect on next process restart.

---

## Client SDK Configuration

The following configuration is the **standard baseline**. Deviate only with documented justification.

```typescript
posthog.init(apiKey, {
  api_host: ingestHost,

  // ── Identity & persistence ──────────────────────────────────
  persistence: 'localStorage',
  person_profiles: 'identified_only', // Anonymous users get no person profile
  disable_persistence: false,
  disable_cookie: false,

  // ── Pageviews ───────────────────────────────────────────────
  // Capture manually in your router/provider so you can attach
  // your own props (pathname, search params, viewport).
  capture_pageview: false,
  capture_pageleave: true,

  // ── Autocapture: OFF ────────────────────────────────────────
  // Generates massive volume (clicked svg / clicked p with text X /
  // Rageclick / input changes / form submits) that drowns the signal.
  // Manual events cover everything that matters.
  autocapture: false,

  // ── Feature flags: OFF unless used ──────────────────────────
  // Stops the SDK from polling /flags on load. Re-enable only when
  // adopting PostHog feature flags.
  advanced_disable_feature_flags: true,

  // ── Web vitals / performance: OFF unless used ───────────────
  // No `Web vitals` events unless you've built dashboards for them.
  capture_performance: false,

  // ── Session recording (optional but recommended) ────────────
  session_recording: {
    maskAllInputs: true,
    maskTextSelector: '[data-private]',
  },
  // Sample rate is configured in the PostHog dashboard, not here.

  loaded: (ph) => {
    // Register super-properties tagged on every event
    ph.register({
      environment: env,     // 'development' | 'staging' | 'production'
      app_version: version, // git SHA or semver
    });
  },
});
```

### Why each `false`

| Setting | Default behaviour | Why we override |
|---|---|---|
| `autocapture` | Captures every click, form submit, input change | Produces 30-50+ events per page load. Dashboards become unreadable. Manual events are clearer and more stable. |
| `capture_pageview` | Captures on init + history changes | We capture in our own provider with custom props (pathname, search params, viewport). |
| `advanced_disable_feature_flags` | SDK polls `/flags` on every load | Unused endpoints can 4xx and pollute the console. Re-enable when adopting flags. |
| `capture_performance` | Captures `Web vitals` events | We don't have dashboards keyed on these. Re-enable when needed. |

---

## Server SDK Configuration

The backend SDK wrapper should expose a single `IAnalytics` (or framework-equivalent) interface that:

1. **No-ops cleanly** when `Enabled=false` or no key is configured.
2. **Auto-injects super-properties** (`environment`, `app_version`, `server_side: true`) on every event so they're symmetrical with the frontend.
3. **Accepts a `distinctId`** matching the frontend's identity scheme.
4. **Never throws** — telemetry failures are logged at `warn` and swallowed.

```csharp
public interface IAnalytics
{
    Task CaptureAsync(string? distinctId, string eventName, Dictionary<string, object?>? props);
    Task IdentifyAsync(string distinctId, Dictionary<string, object?>? properties);
    bool IsEnabled { get; }
}
```

System-initiated events (cron jobs, webhook handlers, background workers) where no user can be attributed should use a stable sentinel distinct_id like `system:<job-type>` so the event still lands in PostHog.

---

## Identity & Groups

### Distinct IDs

| Surface | Distinct ID value |
|---|---|
| Frontend after login | User's primary key from your identity system (typically a string GUID) |
| Frontend before login | PostHog-generated anonymous UUID (auto, no action needed) |
| Backend | Same user primary key as frontend, propagated via the request's authenticated principal |
| Backend system events | `system:<job-type>` (e.g. `system:import-job`, `system:wordpress-sync`) |

The user's primary key should be the **authentication system's user ID**, never an email or username (those can change; IDs shouldn't).

> PostHog's UI may display the user's `email` property in the "Distinct ID" column for readability. The underlying distinct_id is still the GUID — this is UI rendering, not the actual identity.

### Group analytics for tenants / organisations

Every authenticated event should be **grouped by the organisation/tenant** so dashboards can be sliced by customer:

```typescript
posthog.identify(userId, { email, name, role, ... });

if (organizationId) {
  posthog.group('organization', String(organizationId), {
    name: organizationName,
    plan: subscriptionTier,
  });
}
```

**Both calls must be made from a single source of truth** — typically an auth hook/provider. If multiple components each call `useAuth()` and re-fire identify/group on mount, you'll see 10+ `$groupidentify` events per page load.

### Mandatory dedupe at the SDK wrapper level

```typescript
let lastIdentifyKey = '';
let lastGroupKey = '';

export function identify(distinctId: string, props: IdentifyPayload): void {
  if (!isEnabled() || !distinctId) return;
  const key = `${distinctId}::${JSON.stringify(props)}`;
  if (key === lastIdentifyKey) return;
  posthog.identify(distinctId, props);
  lastIdentifyKey = key;
}

export function group(type: string, key: string, props?: GroupPayload): void {
  if (!isEnabled() || !key) return;
  const sentinel = `${type}::${key}::${JSON.stringify(props ?? {})}`;
  if (sentinel === lastGroupKey) return;
  posthog.group(type, key, props);
  lastGroupKey = sentinel;
}

export function resetIdentity(): void {
  posthog.reset();
  lastIdentifyKey = '';
  lastGroupKey = '';
}
```

Sentinels reset on `resetIdentity()` at logout, and naturally reset on page reload (module re-evaluation).

> `posthog.identify()` is internally idempotent on distinct_id alone, but `posthog.group()` is **not** — every call sends a `$groupidentify` over the wire. Dedupe is mandatory.

---

## Event Taxonomy

### Naming conventions

- **`snake_case`** event names. PostHog's default is fine — match it.
- **Object-verb** ordering: `entity_viewed`, `report_exported`, `dashboard_pinned` (not `view_entity`).
- **Past tense** for completed actions (`record_saved`), **present tense** for state changes (`tab_changed`).
- **Prefix by feature area** when ambiguous: `data_grid_filter_changed` (not just `filter_changed`).

### Typed `EVENTS` constant

Never call `posthog.capture('some_event', ...)` directly. Define every event name in a constant and provide a typed helper:

```typescript
export const EVENTS = {
  SEARCH_EXECUTED: 'search_executed',
  ENTITY_VIEWED: 'entity_viewed',
  CUSTOM_REPORT_CREATED: 'custom_report_created',
  // ...
} as const;

export type EventName = (typeof EVENTS)[keyof typeof EVENTS];

export type AnalyticsProps = Record<
  string,
  string | number | boolean | string[] | number[] | null | undefined
>;

export function track(event: EventName, props?: AnalyticsProps): void {
  if (!isEnabled()) return;
  try {
    posthog.capture(event, props);
  } catch (error) {
    logger.error(`PostHog capture failed for "${event}":`, error);
  }
}
```

Why:

- Compiler catches typos and renames across the codebase.
- New events are visible in one place (PR diffs make them easy to review).
- Refactoring an event name is a single change.

### Event property hygiene (PII)

| ✅ Safe to capture | ❌ Never put in props |
|---|---|
| IDs (`entity_id`, `report_id`, `organization_id`) | Email, phone, address |
| Counts (`result_count`, `section_count`) | Free-text the user typed (search queries, comments) |
| Enums (`format: 'csv' \| 'xlsx'`, `mode: 'create' \| 'edit'`) | Names (people, files with personal context) |
| Durations (`duration_ms`) | Tokens, secrets, IP addresses |
| Booleans (`with_snapshots`, `is_admin`) | Health, financial, or other regulated data |
| Field names (`fields: ['name', 'country']`) | The values inside those fields |
| Lengths instead of content (`query_length: 12`) | The query itself |

Email and name belong on the **person profile** (set via `identify`), not on every event.

### Properties to register as super-properties

Properties true for every event in a session should be registered with `posthog.register()` in the SDK's `loaded` callback, not added to every `capture()` call:

- `environment` (`development` / `staging` / `production` / `e2e`)
- `app_version` (git SHA or semver)
- `server_side: true` (backend only)

---

## Filtering Internal Users

PostHog's **Internal & test users** filter (Project Settings → Internal & test users) is the only mechanism for excluding staff QA traffic from dashboards in a single-project setup. Don't skip it.

### Critical clarification

The filter applies to **Insights and Dashboards only**. It does **not** filter Live Events — Live Events always shows everything so you can verify capture is working during QA.

If you're testing and see your own activity in Live Events, that's correct. Open any dashboard insight to confirm the filter is excluding you there.

### Standard filter rules

| Match type | Value | Applies to |
|---|---|---|
| `email contains` | `@yourcompany.com` | Staff |
| `email contains` | `@vendor.com` | External contractors with access |

Document the staff email list in a project-level file (e.g. `docs/posthog/internal-users.md`) and review quarterly.

---

## Multi-Environment Strategy

### Single project (free tier, recommended default)

- One PostHog project. Capture from production only.
- Staging and local hard-disabled at build time.
- Internal Users filter handles staff dashboards exclusion.
- **Cost**: free up to PostHog's monthly event quota.

### Multi-project (paid plans, when justified)

When upgrading to a paid plan and separate projects per environment is desirable:

1. Create separate projects (`<app>-dev`, `<app>-staging`, `<app>-prod`).
2. Wire each environment's CI to inject its own key.
3. Set a `ProdKeySentinel` env var on non-prod environments — the SDK wrapper detects when the configured key matches the sentinel and refuses to initialise (prevents accidentally polluting prod from a misconfigured staging deploy).
4. The Internal Users filter still applies per-project — configure it on prod, optionally on staging.

### Mandatory: key-mismatch safety net

Whether single- or multi-project, every project should ship a safety net that refuses to initialise if the configured key matches a known prod key but the environment isn't production:

```typescript
const keyMismatchDetected =
  prodKeySentinel.length > 0 &&
  apiKey === prodKeySentinel &&
  environment !== 'production';

if (keyMismatchDetected) {
  console.warn(
    '[PostHog] Production key detected outside production environment. ' +
    'Capture disabled to protect prod analytics.',
  );
}

export const enabled =
  explicitlyEnabled && apiKey.length > 0 && !keyMismatchDetected;
```

This is dormant in a single-project setup (the sentinel is unset) but becomes active automatically if multi-project is adopted later.

---

## Required Documentation per Project

Every project using PostHog must maintain:

| File | Contents |
|---|---|
| `docs/posthog/README.md` | Region, project topology, env vars, event catalogue, validation runbook, troubleshooting |
| `docs/posthog/setup.md` | Step-by-step new-project setup (PostHog UI config, GitHub secrets, App Service settings) |
| `docs/posthog/internal-users.md` | Staff email list applied to the Internal Users filter |

The event catalogue in `README.md` is **the source of truth** for what's captured and where. Every new `track()` call must update the catalogue in the same commit.

---

## Common Pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| 4xx errors hitting `eu.posthog.com` or `us.posthog.com` for ingest | Missing the `i.` subdomain — that's the dashboard host, not the ingest host | Change host to `eu.i.posthog.com` / `us.i.posthog.com` |
| 401 on `/flags`, 400 on `/e/`, 404 on `*-assets` | Region mismatch — sending US project key to EU endpoints or vice versa | Match SDK `api_host` to the project's actual region |
| `Group identify` firing 10+ times per page load | `useAuth` consumed by multiple components, each calling `group()` on mount | Implement SDK wrapper dedupe (see Identity section) |
| Dashboards drowning in `clicked svg` / `Rageclick` / `Web vitals` rows | Autocapture and `capture_performance` are on by default | Set `autocapture: false` and `capture_performance: false`; rely on manual events |
| `/flags` returning 400 every page load | Feature flags fetching with no flags configured | `advanced_disable_feature_flags: true` until flags are adopted |
| Events from staging appearing in prod project | Staging build picked up prod env vars | Implement key-mismatch safety net |
| `Distinct ID` column shows email — is identity broken? | PostHog UI displays the person's `email` property as their label | Not broken — actual distinct_id is still the GUID |
| Internal user filter not excluding my activity in Live Events | The filter only applies to Insights/Dashboards | Open an Insight to confirm filtering works |
| `NEXT_PUBLIC_POSTHOG_HOST` updated in CI but old host still used | `NEXT_PUBLIC_*` vars are inlined at build time | Trigger a fresh CI build to bake in the new value |
| All events captured but no person profile | `person_profiles: 'identified_only'` and `identify()` never called | Either call `identify()` post-login or change to `'always'` |

---

## Validation Checklist (per project)

Before declaring PostHog "live" in production:

- [ ] Region confirmed: project is on the documented region (US or EU); SDK `api_host` matches with `i.` subdomain
- [ ] Capture is gated on both opt-in flag **and** non-empty key
- [ ] Staging / local explicitly disabled in CI workflows
- [ ] Frontend SDK config matches the standard baseline (autocapture off, flags off, performance off)
- [ ] Server SDK no-ops cleanly when disabled, auto-injects super-properties
- [ ] `distinct_id` is the user's primary key on both surfaces
- [ ] Group analytics fires once per session per organisation (verify in Live Events)
- [ ] SDK wrapper dedupes identify and group calls
- [ ] No PII in any event prop (audit the EVENTS catalogue)
- [ ] Internal Users filter configured for all staff domains
- [ ] Key-mismatch safety net code is in place (even if dormant)
- [ ] Event catalogue documented in `docs/posthog/README.md`
- [ ] Production smoke test passed: log in, click through, see events in Live Events, confirm dashboards exclude internal users
