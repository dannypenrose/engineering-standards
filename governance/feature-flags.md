# Feature Flags & Progressive Rollout

> Authoritative feature flag standards for progressive rollout, canary deployments, and kill switches.

## Purpose

Establish patterns for safely releasing features, enabling experimentation, and providing operational control over application behavior without deployments.

## Core Principles

1. **Separate deployment from release** - Deploy anytime, release when ready
2. **Gradual rollout** - Start small, increase incrementally
3. **Kill switch ready** - Every feature must be instantly disableable
4. **Clean up technical debt** - Remove flags after full rollout
5. **Audit everything** - Track who changed what and when
6. **Test both states** - Every flag path must be tested

## Flag Types

### Classification

| Type | Duration | Use Case | Example |
| ---- | -------- | -------- | ------- |
| **Release** | Temporary | New feature rollout | `new_checkout_flow` |
| **Experiment** | Temporary | A/B testing | `pricing_experiment_v2` |
| **Ops** | Permanent | Circuit breaker | `disable_external_api` |
| **Permission** | Permanent | Entitlement | `premium_features` |
| **Kill Switch** | Permanent | Emergency disable | `kill_payments` |

### Naming Conventions

```typescript
// Format: {type}_{domain}_{feature}_{version}

// Release flags
const RELEASE_FLAGS = {
  'release_checkout_new_flow_v1': boolean,
  'release_dashboard_widgets_v2': boolean,
};

// Experiment flags
const EXPERIMENT_FLAGS = {
  'exp_pricing_annual_discount': boolean,
  'exp_onboarding_simplified': boolean,
};

// Ops flags
const OPS_FLAGS = {
  'ops_payments_circuit_breaker': boolean,
  'ops_search_fallback_enabled': boolean,
};

// Permission flags
const PERMISSION_FLAGS = {
  'perm_premium_analytics': boolean,
  'perm_api_rate_limit_increased': boolean,
};
```

## App-Specific Domain Flags (Monorepo)

In a monorepo with multiple applications, feature flags operate at two levels:

### Shared Infrastructure Flags

Flags for shared modules that every app can consume (auth, billing, email, etc.):

```typescript
// Shared module flags (defined in @org/feature-flags)
'perm_auth_core_v1'          // Authentication
'perm_billing_stripe_v1'     // Stripe billing
'perm_content_blog_v1'       // Blog CMS
'perm_comms_notifications_v1' // Notifications
```

### App-Specific Domain Flags

Flags for features unique to a single application. These use domain names that reflect the app's feature areas:

**App A** (e-commerce):

```typescript
'perm_catalog_products_v1'           // Product catalog CRUD
'perm_catalog_search_v1'             // Product search
'perm_orders_checkout_v1'            // Checkout flow
'perm_orders_tracking_v1'            // Order tracking
'perm_analytics_dashboard_v1'        // Sales analytics
'perm_social_reviews_v1'             // Customer reviews
```

**App B** (project management):

```typescript
'perm_projects_core_v1'              // Project CRUD
'perm_projects_timeline_v1'          // Timeline/Gantt view
'perm_tasks_assignments_v1'          // Task assignment
'perm_tasks_automation_v1'           // Workflow automation
'perm_reporting_exports_v1'          // Report exports
```

**App C** (content platform):

```typescript
'perm_content_editor_v1'             // Rich text editor
'perm_content_publishing_v1'         // Publishing workflow
'perm_media_uploads_v1'              // Media library
```

### Guidelines

1. **Document app-specific flags** in each app's `docs/feature-flags-brief.md`
2. **Group by domain** in admin UI for clarity
3. **Map to both** environment variables (startup module loading) and runtime flag evaluation
4. **Keep domains unique** across the monorepo to avoid flag name collisions
5. **Map app-specific flags** to their `ENABLE_*` env var via the admin module's `forRoot({ moduleFlags: { ... } })` configuration
6. **Use per-app briefs** as the source of truth for flag inventories
7. **One `ENABLE_*` per feature**: Each feature flag should map to its own env var for granular control. Avoid mapping multiple flags to a single env var — this prevents independent toggling and makes the admin UI misleading

### Module-Loaded Awareness & App-Specific Flag Registration

The admin module's `forRoot()` method accepts a `moduleFlags` mapping of flag name to `ENABLE_*` environment variable. This mapping serves **two purposes**:

1. **Registration**: App-specific flags that have no `LEGACY_FLAG_METADATA` entry only appear in the admin UI when registered via `moduleFlags`. The admin service synthesizes flag entries for any `moduleFlags` key that is not already present in the flag storage layer.
2. **Module-loaded indicator**: When the mapped env var is `false` or missing, the admin UI shows that the underlying module is not loaded, preventing users from enabling flags whose backend module was never started.

```typescript
// apps/my-app/backend/src/modules/enterprise/enterprise.module.ts
// Best practice: one ENABLE_* env var per feature flag for granular control
FeatureFlagAdminModule.forRoot({
  moduleFlags: {
    'perm_catalog_products_v1':      'ENABLE_PRODUCTS',
    'perm_catalog_search_v1':        'ENABLE_SEARCH',
    'perm_orders_checkout_v1':       'ENABLE_CHECKOUT',
    'perm_orders_tracking_v1':       'ENABLE_TRACKING',
    'perm_analytics_dashboard_v1':   'ENABLE_ANALYTICS',
  },
})
```

**Behaviour by flag type:**

| Flag Category | Registration | Admin UI visibility |
| ------------- | ------------ | ------------------- |
| Platform flags (`perm_content_blog_v1`, etc.) | Automatic — built-in `PLATFORM_MODULE_FLAGS` | Always visible |
| App-specific flags via `forRoot({ moduleFlags })` | Required — `moduleFlags` entry | Visible only when registered |

**Important**: If an app-specific flag is not listed in `moduleFlags`, it will **not** appear in the admin UI even if the flag name is used in frontend `FeatureGate` components. Always register app-specific flags in the enterprise module's `FeatureFlagAdminModule.forRoot()` call.

## Frontend Gating Patterns (Required)

### Page-Level Gating

Every feature page **must** be wrapped with a `FeatureGate` component so that directly navigating to the URL respects flag state. Hiding sidebar links alone is insufficient — users can access pages via bookmarks or direct URLs.

**Admin pages** use `fallback={null}` (the existing role-check redirect handles the user experience):

```tsx
import { FeatureGate } from '@org/feature-flags/react';

export default function BlogAdminPage() {
  // ... existing admin role check with redirect ...
  return (
    <FeatureGate flag="perm_content_blog_v1" fallback={null}>
      <div className="container mx-auto p-4">
        <BlogManagement />
      </div>
    </FeatureGate>
  );
}
```

**App-specific pages** use `fallback={<FeatureUnavailable />}` to show a friendly message:

```tsx
import { FeatureGate } from '@org/feature-flags/react';
import { FeatureUnavailable } from '@org/ui';

export default function TasksPage() {
  return (
    <FeatureGate flag="perm_productivity_tasks_v1" fallback={<FeatureUnavailable />}>
      {/* page content */}
    </FeatureGate>
  );
}
```

### Sidebar Gating

Menu items declare an optional `featureFlag` field. The layout component filters items based on flag state before rendering the navigation:

```tsx
import { useMultipleFeatureFlags } from '@org/feature-flags/react';

const menuItems = [
  { text: 'Dashboard', path: '/dashboard', icon: <DashboardIcon /> },
  { text: 'Tasks', path: '/tasks', icon: <TasksIcon />, featureFlag: 'perm_productivity_tasks_v1' },
];

// Filter by flag state
const sidebarFlags = useMultipleFeatureFlags(['perm_productivity_tasks_v1']);
const filteredItems = menuItems.filter((item) => !item.featureFlag || sidebarFlags[item.featureFlag]);
```

### Admin UI Visibility

Not all flags should appear in the admin UI. Infrastructure and ops flags that cannot be meaningfully toggled at runtime are hidden via the `adminVisible` field in `LEGACY_FLAG_METADATA`. This prevents admin users from toggling flags that have no runtime effect (e.g., database, CORS, Docker, caching).

The admin UI separates visible flags into:
- **Platform Features**: Shared flags (blog, changelog, email, AI, billing, etc.)
- **App Features**: App-specific flags registered via `FeatureFlagAdminModule.forRoot({ moduleFlags })`

## Implementation Patterns

### Flag Service Interface

```typescript
// lib/feature-flags/types.ts
interface FeatureFlagService {
  isEnabled(flag: string, context?: FlagContext): Promise<boolean>;
  getVariant<T>(flag: string, context?: FlagContext): Promise<T>;
  getAllFlags(context?: FlagContext): Promise<Record<string, boolean>>;
}

interface FlagContext {
  userId?: string;
  email?: string;
  organizationId?: string;
  environment?: string;
  percentile?: number; // For sticky rollouts
  attributes?: Record<string, string | number | boolean>;
}
```

### React Implementation

```typescript
// hooks/useFeatureFlag.ts
import { useQuery } from '@tanstack/react-query';
import { featureFlagService } from '@/lib/feature-flags';

export function useFeatureFlag(flag: string, defaultValue = false) {
  const { data, isLoading } = useQuery({
    queryKey: ['feature-flag', flag],
    queryFn: () => featureFlagService.isEnabled(flag),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });

  return {
    enabled: data ?? defaultValue,
    isLoading,
  };
}

// Usage in component
function CheckoutPage() {
  const { enabled: newCheckout, isLoading } = useFeatureFlag('release_checkout_new_flow_v1');

  if (isLoading) return <Loading />;

  return newCheckout ? <NewCheckoutFlow /> : <LegacyCheckoutFlow />;
}
```

### Server-Side Implementation (NestJS)

```typescript
// feature-flags/feature-flags.service.ts
@Injectable()
export class FeatureFlagsService {
  constructor(
    private readonly configService: ConfigService,
    private readonly cacheService: CacheService,
  ) {}

  async isEnabled(flag: string, context?: FlagContext): Promise<boolean> {
    // Check cache first
    const cacheKey = this.getCacheKey(flag, context);
    const cached = await this.cacheService.get(cacheKey);
    if (cached !== undefined) return cached;

    // Get flag configuration
    const flagConfig = await this.getFlagConfig(flag);
    if (!flagConfig) return false;

    // Evaluate targeting rules
    const enabled = this.evaluateFlag(flagConfig, context);

    // Cache result
    await this.cacheService.set(cacheKey, enabled, 300); // 5 min TTL

    return enabled;
  }

  private evaluateFlag(config: FlagConfig, context?: FlagContext): boolean {
    // Kill switch check
    if (config.killed) return false;

    // Environment check
    if (config.environments && !config.environments.includes(process.env.NODE_ENV)) {
      return false;
    }

    // Percentage rollout
    if (config.percentage !== undefined && config.percentage < 100) {
      const percentile = context?.percentile ?? this.getUserPercentile(context?.userId);
      if (percentile > config.percentage) return false;
    }

    // Targeting rules
    if (config.targetingRules) {
      return this.evaluateTargetingRules(config.targetingRules, context);
    }

    return config.enabled;
  }

  private getUserPercentile(userId?: string): number {
    if (!userId) return Math.random() * 100;
    // Consistent hashing for sticky rollouts
    const hash = this.hashString(userId);
    return hash % 100;
  }
}

// feature-flags/feature-flag.decorator.ts
export function FeatureFlag(flag: string, fallback?: () => any) {
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;

    descriptor.value = async function (...args: any[]) {
      const flagService = this.featureFlagsService;
      const request = args.find(arg => arg?.user); // Find request with user context

      const enabled = await flagService.isEnabled(flag, {
        userId: request?.user?.id,
        organizationId: request?.user?.organizationId,
      });

      if (!enabled) {
        if (fallback) return fallback();
        throw new NotFoundException('Feature not available');
      }

      return originalMethod.apply(this, args);
    };

    return descriptor;
  };
}

// Usage
@Controller('api/v1/dashboard')
export class DashboardController {
  @Get('widgets')
  @FeatureFlag('release_dashboard_widgets_v2')
  async getNewWidgets() {
    return this.dashboardService.getNewWidgets();
  }
}
```

### .NET Implementation

```csharp
// Services/FeatureFlagService.cs
public interface IFeatureFlagService
{
    Task<bool> IsEnabledAsync(string flag, FlagContext? context = null);
    Task<T?> GetVariantAsync<T>(string flag, FlagContext? context = null);
}

public class FeatureFlagService : IFeatureFlagService
{
    private readonly IConfiguration _configuration;
    private readonly IMemoryCache _cache;

    public async Task<bool> IsEnabledAsync(string flag, FlagContext? context = null)
    {
        var cacheKey = $"ff:{flag}:{context?.UserId}";

        return await _cache.GetOrCreateAsync(cacheKey, async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);

            var flagConfig = await GetFlagConfigAsync(flag);
            if (flagConfig == null) return false;

            return EvaluateFlag(flagConfig, context);
        });
    }

    private bool EvaluateFlag(FlagConfig config, FlagContext? context)
    {
        if (config.Killed) return false;

        if (config.Percentage.HasValue && config.Percentage < 100)
        {
            var percentile = GetUserPercentile(context?.UserId);
            if (percentile > config.Percentage) return false;
        }

        return config.Enabled;
    }
}

// Middleware/FeatureFlagMiddleware.cs
public class FeatureFlagAttribute : ActionFilterAttribute
{
    private readonly string _flag;

    public FeatureFlagAttribute(string flag)
    {
        _flag = flag;
    }

    public override async Task OnActionExecutionAsync(
        ActionExecutingContext context,
        ActionExecutionDelegate next)
    {
        var featureFlags = context.HttpContext.RequestServices
            .GetRequiredService<IFeatureFlagService>();

        var userId = context.HttpContext.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;

        var enabled = await featureFlags.IsEnabledAsync(_flag, new FlagContext
        {
            UserId = userId
        });

        if (!enabled)
        {
            context.Result = new NotFoundResult();
            return;
        }

        await next();
    }
}

// Usage
[ApiController]
[Route("api/v1/dashboard")]
public class DashboardController : ControllerBase
{
    [HttpGet("widgets")]
    [FeatureFlag("release_dashboard_widgets_v2")]
    public async Task<IActionResult> GetNewWidgets()
    {
        return Ok(await _dashboardService.GetNewWidgets());
    }
}
```

## Progressive Rollout Strategy

### Rollout Phases

```
Phase 1: Internal       Phase 2: Beta        Phase 3: GA
┌─────────────┐        ┌─────────────┐      ┌─────────────┐
│   0.1%      │───────▶│   5-10%     │─────▶│   100%      │
│  (team)     │        │  (opt-in)   │      │  (everyone) │
└─────────────┘        └─────────────┘      └─────────────┘
    │                       │                    │
    ▼                       ▼                    ▼
  Monitor              Monitor              Monitor
  1-2 days             1 week              Ongoing
```

### Rollout Configuration

```typescript
// Flag configuration schema
interface FlagConfig {
  name: string;
  type: 'release' | 'experiment' | 'ops' | 'permission';
  enabled: boolean;
  killed: boolean; // Emergency kill switch

  // Targeting
  environments?: ('development' | 'staging' | 'production')[];
  percentage?: number; // 0-100

  // Rollout schedule
  rollout?: {
    phases: RolloutPhase[];
    currentPhase: number;
  };

  // Targeting rules
  targetingRules?: TargetingRule[];

  // Metadata
  owner: string;
  description: string;
  createdAt: string;
  expiresAt?: string; // For temporary flags
}

interface RolloutPhase {
  name: string;
  percentage: number;
  duration: string; // e.g., "7d"
  criteria: string[];
}

// Example configuration
const newCheckoutFlag: FlagConfig = {
  name: 'release_checkout_new_flow_v1',
  type: 'release',
  enabled: true,
  killed: false,
  environments: ['production'],
  percentage: 10,
  rollout: {
    currentPhase: 2,
    phases: [
      { name: 'internal', percentage: 1, duration: '2d', criteria: ['@company.com emails'] },
      { name: 'beta', percentage: 10, duration: '7d', criteria: ['Opted-in users'] },
      { name: 'ga', percentage: 100, duration: 'indefinite', criteria: ['All users'] },
    ],
  },
  owner: 'checkout-team',
  description: 'New streamlined checkout flow',
  createdAt: '2024-01-15',
  expiresAt: '2024-04-15', // Clean up by this date
};
```

### Canary Deployment Pattern

```typescript
// Canary rollout with automatic progression
class CanaryRollout {
  private readonly flag: string;
  private readonly healthCheck: () => Promise<HealthStatus>;

  async progressRollout(): Promise<void> {
    const config = await this.getConfig();
    const currentPercentage = config.percentage ?? 0;

    // Check health metrics
    const health = await this.healthCheck();
    if (!health.healthy) {
      await this.rollback();
      throw new Error(`Canary failed health check: ${health.reason}`);
    }

    // Progress to next percentage
    const nextPercentage = this.getNextPercentage(currentPercentage);
    await this.setPercentage(nextPercentage);

    console.log(`Rolled out ${this.flag} to ${nextPercentage}%`);
  }

  private getNextPercentage(current: number): number {
    const stages = [1, 5, 10, 25, 50, 75, 100];
    const nextIndex = stages.findIndex(p => p > current);
    return stages[nextIndex] ?? 100;
  }

  async rollback(): Promise<void> {
    await this.setPercentage(0);
    await this.alertTeam(`Canary rollback triggered for ${this.flag}`);
  }
}
```

## Testing Feature Flags

### Unit Testing

```typescript
// __tests__/checkout.test.ts
describe('Checkout', () => {
  const mockFeatureFlags = {
    isEnabled: jest.fn(),
  };

  beforeEach(() => {
    jest.clearAllMocks();
  });

  describe('when new checkout is enabled', () => {
    beforeEach(() => {
      mockFeatureFlags.isEnabled.mockResolvedValue(true);
    });

    it('renders new checkout flow', async () => {
      render(<CheckoutPage featureFlags={mockFeatureFlags} />);
      expect(await screen.findByTestId('new-checkout')).toBeInTheDocument();
    });
  });

  describe('when new checkout is disabled', () => {
    beforeEach(() => {
      mockFeatureFlags.isEnabled.mockResolvedValue(false);
    });

    it('renders legacy checkout flow', async () => {
      render(<CheckoutPage featureFlags={mockFeatureFlags} />);
      expect(await screen.findByTestId('legacy-checkout')).toBeInTheDocument();
    });
  });
});
```

### E2E Testing

```typescript
// e2e/checkout.spec.ts
test.describe('Checkout Feature Flag', () => {
  test('new checkout flow when flag enabled', async ({ page }) => {
    // Override feature flags for test
    await page.route('**/api/feature-flags', async route => {
      await route.fulfill({
        json: { 'release_checkout_new_flow_v1': true },
      });
    });

    await page.goto('/checkout');
    await expect(page.getByTestId('new-checkout')).toBeVisible();
  });

  test('legacy checkout flow when flag disabled', async ({ page }) => {
    await page.route('**/api/feature-flags', async route => {
      await route.fulfill({
        json: { 'release_checkout_new_flow_v1': false },
      });
    });

    await page.goto('/checkout');
    await expect(page.getByTestId('legacy-checkout')).toBeVisible();
  });
});
```

## Flag Lifecycle Management

### Flag Creation

```markdown
## New Flag Checklist

- [ ] Flag follows naming convention
- [ ] Type is correctly categorized
- [ ] Owner is assigned
- [ ] Description is clear
- [ ] Expiration date set (for temporary flags)
- [ ] Monitoring/alerts configured
- [ ] Rollback plan documented
- [ ] Both code paths tested
```

### Flag Cleanup

```typescript
// scripts/audit-feature-flags.ts
async function auditFlags() {
  const flags = await getAllFlags();
  const now = new Date();

  const issues: FlagIssue[] = [];

  for (const flag of flags) {
    // Check for expired flags
    if (flag.expiresAt && new Date(flag.expiresAt) < now) {
      issues.push({
        flag: flag.name,
        issue: 'expired',
        message: `Flag expired on ${flag.expiresAt}`,
        severity: 'high',
      });
    }

    // Check for stale flags (100% for > 30 days)
    if (flag.percentage === 100 && flag.type === 'release') {
      const daysSinceFullRollout = getDaysSince(flag.updatedAt);
      if (daysSinceFullRollout > 30) {
        issues.push({
          flag: flag.name,
          issue: 'stale',
          message: `Flag at 100% for ${daysSinceFullRollout} days`,
          severity: 'medium',
        });
      }
    }

    // Check for flags without owners
    if (!flag.owner) {
      issues.push({
        flag: flag.name,
        issue: 'no-owner',
        message: 'Flag has no assigned owner',
        severity: 'low',
      });
    }
  }

  return issues;
}
```

### Cleanup Process

```markdown
## Flag Removal Process

1. **Verify full rollout**
   - Confirm flag is at 100% for > 2 weeks
   - No incidents related to feature
   - Metrics are healthy

2. **Remove flag checks from code**
   - Search codebase for flag name
   - Remove conditional logic
   - Delete fallback/legacy code

3. **Update tests**
   - Remove flag-specific test cases
   - Update integration tests

4. **Remove flag configuration**
   - Delete flag from flag service
   - Update documentation

5. **Clean up monitoring**
   - Remove flag-specific dashboards
   - Update alerts
```

## Monitoring & Observability

### Flag Metrics

```typescript
// Track flag evaluations
const flagMetrics = {
  evaluations: new Counter({
    name: 'feature_flag_evaluations_total',
    help: 'Total feature flag evaluations',
    labelNames: ['flag', 'result', 'source'],
  }),

  latency: new Histogram({
    name: 'feature_flag_evaluation_duration_seconds',
    help: 'Feature flag evaluation duration',
    labelNames: ['flag'],
    buckets: [0.001, 0.005, 0.01, 0.05, 0.1],
  }),
};

// Usage
async function evaluateFlag(flag: string, context: FlagContext) {
  const timer = flagMetrics.latency.startTimer({ flag });

  try {
    const result = await doEvaluate(flag, context);
    flagMetrics.evaluations.inc({ flag, result: String(result), source: 'cache' });
    return result;
  } finally {
    timer();
  }
}
```

### Dashboard Requirements

```yaml
# Dashboard panels for feature flags
panels:
  - title: "Flag Evaluations"
    query: sum(rate(feature_flag_evaluations_total[5m])) by (flag, result)

  - title: "Rollout Progress"
    query: feature_flag_percentage{type="release"}

  - title: "Error Rate by Flag"
    query: |
      sum(rate(http_errors_total[5m])) by (feature_flag) /
      sum(rate(http_requests_total[5m])) by (feature_flag)

  - title: "Latency by Flag"
    query: |
      histogram_quantile(0.95,
        sum(rate(http_request_duration_seconds_bucket[5m])) by (le, feature_flag)
      )
```

## Checklist

### Setup

- [ ] Feature flag service/library chosen
- [ ] SDK integrated in frontend and backend
- [ ] Flag configuration storage set up
- [ ] Caching strategy implemented
- [ ] Audit logging configured

### Process

- [ ] Naming convention documented
- [ ] Flag creation process defined
- [ ] Rollout phases documented
- [ ] Cleanup process established
- [ ] Regular flag audits scheduled

### Operations

- [ ] Kill switches available for critical flags
- [ ] Monitoring dashboards created
- [ ] Alerts configured for flag issues
- [ ] Runbooks for flag-related incidents

## References

- [LaunchDarkly Feature Flags](https://launchdarkly.com/blog/feature-flag-best-practices/)
- [Martin Fowler - Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [Netflix Feature Flags](https://netflixtechblog.com/feature-flags-at-netflix-9b43b64c2b0)
- [Meta Gatekeeper](https://engineering.fb.com/)
- [GitLab Feature Flags](https://docs.gitlab.com/ee/operations/feature_flags.html)
