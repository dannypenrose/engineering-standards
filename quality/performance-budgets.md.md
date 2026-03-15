# Performance Budgets

> Authoritative performance standards for Core Web Vitals, API latency targets, and bundle size budgets.

## Purpose

Establish measurable performance targets that ensure applications deliver fast, responsive experiences across all devices and network conditions.

## Core Principles

1. **Measure what matters** - Focus on user-centric metrics, not vanity metrics
2. **Budget early** - Set budgets before development, not after
3. **Automate enforcement** - Budgets without automation are suggestions
4. **Progressive enhancement** - Core functionality works on slow connections
5. **Real user data** - Synthetic tests inform, RUM validates
6. **Continuous monitoring** - Performance is not a one-time achievement

## Core Web Vitals Targets

### Required Thresholds (Google Standards)

| Metric | Good | Needs Improvement | Poor | Target |
| ------ | ---- | ----------------- | ---- | ------ |
| **LCP** (Largest Contentful Paint) | ≤ 2.5s | 2.5s - 4.0s | > 4.0s | **≤ 2.0s** |
| **INP** (Interaction to Next Paint) | ≤ 200ms | 200ms - 500ms | > 500ms | **≤ 150ms** |
| **CLS** (Cumulative Layout Shift) | ≤ 0.1 | 0.1 - 0.25 | > 0.25 | **≤ 0.05** |

### Additional Metrics

| Metric | Target | Critical Path |
| ------ | ------ | ------------- |
| **TTFB** (Time to First Byte) | ≤ 200ms | ≤ 100ms |
| **FCP** (First Contentful Paint) | ≤ 1.8s | ≤ 1.0s |
| **TTI** (Time to Interactive) | ≤ 3.8s | ≤ 2.5s |
| **TBT** (Total Blocking Time) | ≤ 200ms | ≤ 100ms |

## Bundle Size Budgets

### JavaScript Budgets

```javascript
// next.config.js - Example bundle budget configuration
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

module.exports = withBundleAnalyzer({
  experimental: {
    // Warn when pages exceed budget
    largePageDataBytes: 128 * 1024, // 128KB
  },
});
```

### Size Limits

| Asset Type | Budget (Compressed) | Critical Path |
| ---------- | ------------------- | ------------- |
| **Initial JS** | ≤ 150KB | ≤ 100KB |
| **Initial CSS** | ≤ 50KB | ≤ 30KB |
| **Total Initial Load** | ≤ 300KB | ≤ 200KB |
| **Per-route JS** | ≤ 50KB | ≤ 30KB |
| **Images (hero)** | ≤ 200KB | ≤ 100KB |
| **Fonts (total)** | ≤ 100KB | ≤ 50KB |

### Enforcement with bundlesize

```json
// package.json
{
  "bundlesize": [
    {
      "path": ".next/static/chunks/main-*.js",
      "maxSize": "150 kB"
    },
    {
      "path": ".next/static/chunks/pages/_app-*.js",
      "maxSize": "50 kB"
    },
    {
      "path": ".next/static/css/*.css",
      "maxSize": "50 kB"
    }
  ]
}
```

## API Performance Budgets

### Response Time SLAs

| Endpoint Type | P50 | P95 | P99 | Timeout |
| ------------- | --- | --- | --- | ------- |
| **Health check** | ≤ 10ms | ≤ 50ms | ≤ 100ms | 1s |
| **Simple GET** | ≤ 50ms | ≤ 150ms | ≤ 300ms | 5s |
| **List/Search** | ≤ 100ms | ≤ 300ms | ≤ 500ms | 10s |
| **Write operations** | ≤ 100ms | ≤ 300ms | ≤ 500ms | 10s |
| **Complex queries** | ≤ 200ms | ≤ 500ms | ≤ 1s | 30s |
| **File upload** | ≤ 500ms | ≤ 2s | ≤ 5s | 60s |
| **Report generation** | ≤ 2s | ≤ 5s | ≤ 10s | 120s |

### Database Query Budgets

```typescript
// Query performance thresholds
const QUERY_BUDGETS = {
  simple: { warn: 10, error: 50 },      // ms
  withJoins: { warn: 50, error: 200 },  // ms
  aggregate: { warn: 100, error: 500 }, // ms
  report: { warn: 500, error: 2000 },   // ms
};

// Middleware to track query performance
export function queryPerformanceMiddleware(params, next) {
  const start = performance.now();
  const result = next(params);
  const duration = performance.now() - start;

  if (duration > QUERY_BUDGETS[params.type]?.error) {
    logger.error('Query exceeded budget', {
      query: params.query,
      duration,
      budget: QUERY_BUDGETS[params.type],
    });
  }

  return result;
}
```

## Frontend Performance Standards

### Image Optimization

```typescript
// Required image optimizations
const IMAGE_STANDARDS = {
  formats: ['avif', 'webp', 'jpg'], // Priority order
  quality: 80,
  lazyLoad: true, // Below the fold
  placeholder: 'blur', // Or 'empty' for small images

  // Size presets
  sizes: {
    thumbnail: { width: 150, height: 150 },
    card: { width: 400, height: 300 },
    hero: { width: 1200, height: 600 },
    full: { width: 1920, height: 1080 },
  },
};

// Next.js Image component usage
<Image
  src={src}
  alt={alt}
  width={400}
  height={300}
  placeholder="blur"
  blurDataURL={blurDataUrl}
  loading="lazy"
  sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
/>
```

### Font Loading Strategy

```css
/* Optimal font loading */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: swap; /* or 'optional' for non-critical */
  font-weight: 400;
  font-style: normal;
  unicode-range: U+0000-00FF; /* Subset if possible */
}
```

```typescript
// Next.js font optimization
import { Inter } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  preload: true,
  variable: '--font-inter',
});
```

### Code Splitting Requirements

```typescript
// Dynamic imports for non-critical components
import dynamic from 'next/dynamic';

// Heavy components loaded on demand
const HeavyChart = dynamic(() => import('@/components/HeavyChart'), {
  loading: () => <ChartSkeleton />,
  ssr: false, // Client-only if needed
});

// Route-based splitting (automatic in Next.js App Router)
// Each page in /app is automatically code-split
```

## Backend Performance Standards

### .NET Performance Budgets

```csharp
// Response time middleware
public class PerformanceMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<PerformanceMiddleware> _logger;

    private static readonly Dictionary<string, int> Budgets = new()
    {
        { "GET", 200 },      // ms
        { "POST", 300 },     // ms
        { "PUT", 300 },      // ms
        { "DELETE", 200 },   // ms
    };

    public async Task InvokeAsync(HttpContext context)
    {
        var sw = Stopwatch.StartNew();

        await _next(context);

        sw.Stop();
        var budget = Budgets.GetValueOrDefault(context.Request.Method, 500);

        if (sw.ElapsedMilliseconds > budget)
        {
            _logger.LogWarning(
                "Request exceeded budget: {Method} {Path} took {Duration}ms (budget: {Budget}ms)",
                context.Request.Method,
                context.Request.Path,
                sw.ElapsedMilliseconds,
                budget);
        }
    }
}
```

### NestJS Performance Budgets

```typescript
// performance.interceptor.ts
@Injectable()
export class PerformanceInterceptor implements NestInterceptor {
  private readonly budgets: Record<string, number> = {
    GET: 200,
    POST: 300,
    PUT: 300,
    DELETE: 200,
  };

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const start = Date.now();

    return next.handle().pipe(
      tap(() => {
        const duration = Date.now() - start;
        const budget = this.budgets[request.method] || 500;

        if (duration > budget) {
          this.logger.warn(`Request exceeded budget`, {
            method: request.method,
            path: request.url,
            duration,
            budget,
          });
        }
      }),
    );
  }
}
```

## Monitoring & Alerting

### Real User Monitoring (RUM)

```typescript
// Web Vitals reporting
import { onCLS, onINP, onLCP, onFCP, onTTFB } from 'web-vitals';

function sendToAnalytics(metric) {
  const body = JSON.stringify({
    name: metric.name,
    value: metric.value,
    rating: metric.rating,
    delta: metric.delta,
    id: metric.id,
    navigationType: metric.navigationType,
  });

  // Use sendBeacon for reliability
  if (navigator.sendBeacon) {
    navigator.sendBeacon('/api/analytics/vitals', body);
  } else {
    fetch('/api/analytics/vitals', { body, method: 'POST', keepalive: true });
  }
}

onCLS(sendToAnalytics);
onINP(sendToAnalytics);
onLCP(sendToAnalytics);
onFCP(sendToAnalytics);
onTTFB(sendToAnalytics);
```

### Performance Alerts

```yaml
# prometheus-alerts.yml
groups:
  - name: performance
    rules:
      - alert: HighP99Latency
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, path)
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency exceeds 1 second"
          description: "Path {{ $labels.path }} has P99 latency of {{ $value }}s"

      - alert: SlowDatabaseQueries
        expr: |
          histogram_quantile(0.95,
            sum(rate(db_query_duration_seconds_bucket[5m])) by (le, query_type)
          ) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Database queries exceeding budget"
```

## CI/CD Integration

### Lighthouse CI Configuration

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      url: ['http://localhost:3000/', 'http://localhost:3000/dashboard'],
      numberOfRuns: 3,
    },
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.9 }],
        'categories:accessibility': ['error', { minScore: 0.9 }],
        'first-contentful-paint': ['error', { maxNumericValue: 1800 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['error', { maxNumericValue: 200 }],
      },
    },
    upload: {
      target: 'temporary-public-storage',
    },
  },
};
```

### GitHub Actions Integration

```yaml
# .github/workflows/performance.yml
name: Performance Budget Check

on:
  pull_request:
    branches: [main]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: pnpm install

      - name: Build
        run: pnpm build

      - name: Run Lighthouse CI
        run: |
          npm install -g @lhci/cli
          lhci autorun

      - name: Check bundle size
        run: pnpm bundlesize
```

## Performance Testing Requirements

### Pre-deployment Checklist

- [ ] Lighthouse score ≥ 90 for Performance
- [ ] All Core Web Vitals in "Good" range
- [ ] Bundle size within budget
- [ ] No render-blocking resources
- [ ] Images optimized and lazy-loaded
- [ ] Fonts preloaded with display: swap

### Regression Testing

```typescript
// performance.spec.ts
describe('Performance Budgets', () => {
  it('should load homepage under 3 seconds', async () => {
    const start = Date.now();
    await page.goto('/');
    await page.waitForLoadState('networkidle');
    const loadTime = Date.now() - start;

    expect(loadTime).toBeLessThan(3000);
  });

  it('should have LCP under 2.5 seconds', async () => {
    const lcp = await page.evaluate(() => {
      return new Promise((resolve) => {
        new PerformanceObserver((list) => {
          const entries = list.getEntries();
          resolve(entries[entries.length - 1].startTime);
        }).observe({ type: 'largest-contentful-paint', buffered: true });
      });
    });

    expect(lcp).toBeLessThan(2500);
  });
});
```

## Device & Network Budgets

### Mobile-First Budgets

| Condition | Target Load Time | JS Budget |
| --------- | ---------------- | --------- |
| Fast 3G (1.6 Mbps) | ≤ 5s | ≤ 100KB |
| Slow 4G (4 Mbps) | ≤ 3s | ≤ 150KB |
| 4G (12 Mbps) | ≤ 2s | ≤ 200KB |
| WiFi (30+ Mbps) | ≤ 1s | ≤ 300KB |

### Testing on Constrained Networks

```typescript
// Playwright network throttling
await page.route('**/*', (route) => {
  route.continue();
});

// Simulate slow 3G
await context.route('**/*', async (route) => {
  await new Promise((r) => setTimeout(r, 100)); // Add latency
  route.continue();
});
```

## Checklist

### Project Setup

- [ ] Lighthouse CI configured
- [ ] Bundle analyzer installed
- [ ] RUM tracking implemented
- [ ] Performance monitoring dashboard
- [ ] Alerts configured for budget violations

### Development Process

- [ ] Performance budgets in CI pipeline
- [ ] Bundle size checks on PRs
- [ ] Core Web Vitals tracked
- [ ] Image optimization automated
- [ ] Code splitting implemented

### Monitoring

- [ ] RUM data collected
- [ ] P50/P95/P99 latencies tracked
- [ ] Database query times monitored
- [ ] Alert thresholds configured
- [ ] Weekly performance reviews

## References

- [Google Web Vitals](https://web.dev/vitals/)
- [Lighthouse CI](https://github.com/GoogleChrome/lighthouse-ci)
- [web-vitals Library](https://github.com/GoogleChrome/web-vitals)
- [Performance Budgets 101](https://web.dev/performance-budgets-101/)
- [Next.js Performance](https://nextjs.org/docs/app/building-your-application/optimizing)
- [Chrome DevTools Performance](https://developer.chrome.com/docs/devtools/performance/)
