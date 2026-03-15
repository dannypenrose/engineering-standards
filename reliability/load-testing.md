# Load Testing Standards

> Authoritative load testing standards for performance validation, baseline management, and capacity planning.

## Purpose

Establish consistent practices for performance testing to validate system capacity, identify bottlenecks, and ensure applications meet performance requirements under expected and peak loads.

## Core Principles

1. **Test early, test often** - Performance is a feature, not an afterthought
2. **Production-like environment** - Test conditions should mirror production
3. **Realistic scenarios** - Model actual user behavior, not synthetic patterns
4. **Baseline and compare** - Always measure against established baselines
5. **Automate and integrate** - Include in CI/CD pipeline
6. **Monitor everything** - Correlate test results with system metrics

## Test Types

### Load Testing Categories

| Type | Purpose | Load Pattern | Duration |
| ---- | ------- | ------------ | -------- |
| **Smoke** | Verify basic functionality | Minimal (1-5 users) | 1-5 minutes |
| **Load** | Validate performance at expected load | Normal load | 15-60 minutes |
| **Stress** | Find breaking point | Increasing to failure | Until failure |
| **Soak/Endurance** | Find memory leaks, degradation | Sustained load | 4-24 hours |
| **Spike** | Test sudden traffic increases | Rapid ramp up/down | 15-30 minutes |
| **Capacity** | Determine maximum throughput | Incremental increase | Until SLO breach |

### Load Patterns

```
Load Test               Stress Test              Spike Test
  │                        │                        │
  │    ┌────────────┐      │         ╱╲             │       ╱╲
Users │    │         │    Users     ╱  ╲           Users   ╱  ╲
  │    │         │      │         ╱    ╲             │     ╱    ╲
  │   ╱          │      │        ╱      ╲            │    ╱      ╲
  │  ╱           │      │       ╱        ╲           │   ╱        ╲
  └──────────────────   └─────────────────────      └────────────────
       Time                   Time                      Time

Soak Test                              Capacity Test
  │                                       │
  │   ┌─────────────────────────────┐     │          ╱
Users │                              │   Users      ╱
  │   │                              │     │       ╱
  │  ╱                               │     │      ╱
  │ ╱                                │     │     ╱
  └─────────────────────────────────────  └────────────
       Time (hours)                            Time
```

## Test Implementation

### k6 Load Testing Framework

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const apiLatency = new Trend('api_latency', true);

// Test configuration
export const options = {
  // Stages define the load pattern
  stages: [
    { duration: '2m', target: 50 },   // Ramp up to 50 users
    { duration: '5m', target: 50 },   // Stay at 50 users
    { duration: '2m', target: 100 },  // Ramp up to 100 users
    { duration: '5m', target: 100 },  // Stay at 100 users
    { duration: '2m', target: 0 },    // Ramp down
  ],

  // Performance thresholds
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    http_req_failed: ['rate<0.01'],
    errors: ['rate<0.01'],
  },

  // Cloud/distributed execution
  ext: {
    loadimpact: {
      projectID: 123456,
      name: 'API Load Test',
    },
  },
};

// Test scenarios
const BASE_URL = __ENV.BASE_URL || 'https://api.example.com';

export default function () {
  // Scenario: Browse products (60% of traffic)
  if (Math.random() < 0.6) {
    browseProducts();
  }
  // Scenario: Search (25% of traffic)
  else if (Math.random() < 0.85) {
    searchProducts();
  }
  // Scenario: Checkout (15% of traffic)
  else {
    checkout();
  }

  sleep(Math.random() * 3 + 1); // 1-4 second think time
}

function browseProducts() {
  const res = http.get(`${BASE_URL}/api/v1/products`, {
    tags: { name: 'GET /products' },
  });

  apiLatency.add(res.timings.duration);

  const success = check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
    'has products': (r) => JSON.parse(r.body).products.length > 0,
  });

  errorRate.add(!success);
}

function searchProducts() {
  const query = ['laptop', 'phone', 'tablet', 'headphones'][Math.floor(Math.random() * 4)];

  const res = http.get(`${BASE_URL}/api/v1/products/search?q=${query}`, {
    tags: { name: 'GET /products/search' },
  });

  apiLatency.add(res.timings.duration);

  const success = check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 1000ms': (r) => r.timings.duration < 1000,
  });

  errorRate.add(!success);
}

function checkout() {
  // Create order
  const orderRes = http.post(
    `${BASE_URL}/api/v1/orders`,
    JSON.stringify({
      items: [{ productId: 'prod_123', quantity: 1 }],
    }),
    {
      headers: { 'Content-Type': 'application/json' },
      tags: { name: 'POST /orders' },
    }
  );

  const success = check(orderRes, {
    'order created': (r) => r.status === 201,
    'response time < 2000ms': (r) => r.timings.duration < 2000,
  });

  errorRate.add(!success);
}
```

### Artillery Test Configuration

```yaml
# artillery.yml
config:
  target: 'https://api.example.com'
  phases:
    - duration: 120
      arrivalRate: 10
      name: "Warm up"
    - duration: 300
      arrivalRate: 50
      name: "Sustained load"
    - duration: 120
      arrivalRate: 100
      name: "Peak load"
    - duration: 60
      arrivalRate: 10
      name: "Cool down"

  defaults:
    headers:
      Content-Type: 'application/json'

  plugins:
    expect: {}
    metrics-by-endpoint: {}

  ensure:
    p95: 500
    p99: 1000
    maxErrorRate: 1

scenarios:
  - name: "Browse and search"
    weight: 60
    flow:
      - get:
          url: "/api/v1/products"
          capture:
            - json: "$.products[0].id"
              as: "productId"
          expect:
            - statusCode: 200
            - contentType: json
      - think: 2
      - get:
          url: "/api/v1/products/{{ productId }}"
          expect:
            - statusCode: 200

  - name: "Checkout flow"
    weight: 40
    flow:
      - post:
          url: "/api/v1/cart"
          json:
            productId: "prod_123"
            quantity: 1
          expect:
            - statusCode: 201
      - think: 1
      - post:
          url: "/api/v1/orders"
          json:
            cartId: "{{ cartId }}"
          expect:
            - statusCode: 201
```

### .NET Load Testing with NBomber

```csharp
// LoadTest.cs
using NBomber.CSharp;
using NBomber.Http.CSharp;

public class LoadTests
{
    public static void Run()
    {
        var httpClient = new HttpClient();

        var browseScenario = Scenario.Create("browse_products", async context =>
        {
            var request = Http.CreateRequest("GET", "https://api.example.com/api/v1/products");
            var response = await Http.Send(httpClient, request);

            return response.IsError
                ? Response.Fail()
                : Response.Ok(statusCode: response.StatusCode);
        })
        .WithLoadSimulations(
            Simulation.InjectPerSec(rate: 50, during: TimeSpan.FromMinutes(5)),
            Simulation.InjectPerSec(rate: 100, during: TimeSpan.FromMinutes(5))
        );

        var checkoutScenario = Scenario.Create("checkout", async context =>
        {
            var request = Http.CreateRequest("POST", "https://api.example.com/api/v1/orders")
                .WithHeader("Content-Type", "application/json")
                .WithBody(new StringContent("{\"items\": [{\"productId\": \"123\", \"quantity\": 1}]}"));

            var response = await Http.Send(httpClient, request);

            return response.IsError
                ? Response.Fail()
                : Response.Ok(statusCode: response.StatusCode);
        })
        .WithLoadSimulations(
            Simulation.InjectPerSec(rate: 10, during: TimeSpan.FromMinutes(5))
        );

        NBomberRunner
            .RegisterScenarios(browseScenario, checkoutScenario)
            .WithReportFolder("./reports")
            .WithReportFormats(ReportFormat.Html, ReportFormat.Csv)
            .Run();
    }
}
```

## Performance Thresholds

### API Performance Targets

| Endpoint Type | P50 | P95 | P99 | Error Rate |
| ------------- | --- | --- | --- | ---------- |
| Health check | 10ms | 50ms | 100ms | 0% |
| Read (single) | 50ms | 150ms | 300ms | 0.1% |
| Read (list) | 100ms | 300ms | 500ms | 0.1% |
| Write | 100ms | 300ms | 500ms | 0.5% |
| Search | 200ms | 500ms | 1000ms | 0.5% |
| File upload | 500ms | 2000ms | 5000ms | 1% |

### Throughput Targets

```typescript
// Target throughput based on expected load
const THROUGHPUT_TARGETS = {
  // Requests per second at different load levels
  normal: {
    api: 100,
    search: 50,
    checkout: 10,
  },
  peak: {
    api: 500,
    search: 200,
    checkout: 50,
  },
  stress: {
    api: 1000,
    search: 500,
    checkout: 100,
  },
};
```

## CI/CD Integration

### GitHub Actions Integration

```yaml
# .github/workflows/load-test.yml
name: Load Tests

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 2 * * *' # Nightly at 2 AM

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup k6
        uses: grafana/setup-k6-action@v1

      - name: Deploy to staging
        run: |
          # Deploy application to staging environment
          ./scripts/deploy-staging.sh

      - name: Wait for deployment
        run: sleep 60

      - name: Run smoke tests
        run: k6 run tests/load/smoke.js

      - name: Run load tests
        run: k6 run tests/load/load-test.js
        env:
          BASE_URL: ${{ secrets.STAGING_URL }}
          K6_CLOUD_TOKEN: ${{ secrets.K6_CLOUD_TOKEN }}

      - name: Upload results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: load-test-results
          path: |
            summary.json
            results.html

      - name: Check thresholds
        run: |
          # Fail if thresholds not met
          if [ $(jq '.metrics.http_req_failed.rate' summary.json) > 0.01 ]; then
            echo "Error rate threshold exceeded"
            exit 1
          fi
```

### Pre-Release Performance Gates

```yaml
# .github/workflows/release.yml
jobs:
  performance-gate:
    runs-on: ubuntu-latest
    needs: [build, deploy-staging]
    steps:
      - name: Run performance tests
        run: |
          k6 run tests/load/performance-gate.js \
            --out json=results.json \
            --env BASE_URL=${{ secrets.STAGING_URL }}

      - name: Validate performance
        run: |
          # Compare against baseline
          node scripts/compare-performance.js \
            --baseline baseline.json \
            --current results.json \
            --threshold 10  # 10% regression allowed
```

## Baseline Management

### Establishing Baselines

```typescript
// scripts/establish-baseline.ts
interface PerformanceBaseline {
  timestamp: string;
  version: string;
  metrics: {
    p50: number;
    p95: number;
    p99: number;
    errorRate: number;
    throughput: number;
  };
  conditions: {
    users: number;
    duration: string;
    environment: string;
  };
}

async function establishBaseline(): Promise<PerformanceBaseline> {
  // Run standardized test
  const results = await runK6Test('baseline-test.js');

  const baseline: PerformanceBaseline = {
    timestamp: new Date().toISOString(),
    version: process.env.VERSION || 'unknown',
    metrics: {
      p50: results.http_req_duration.p50,
      p95: results.http_req_duration.p95,
      p99: results.http_req_duration.p99,
      errorRate: results.http_req_failed.rate,
      throughput: results.iterations.rate,
    },
    conditions: {
      users: 100,
      duration: '10m',
      environment: 'staging',
    },
  };

  // Store baseline
  await saveBaseline(baseline);

  return baseline;
}
```

### Regression Detection

```typescript
// scripts/compare-performance.ts
interface ComparisonResult {
  passed: boolean;
  regressions: Regression[];
  improvements: Improvement[];
}

function compareToBaseline(
  baseline: PerformanceBaseline,
  current: PerformanceBaseline,
  threshold: number = 10
): ComparisonResult {
  const regressions: Regression[] = [];
  const improvements: Improvement[] = [];

  // Compare each metric
  const metrics = ['p50', 'p95', 'p99', 'errorRate'];

  for (const metric of metrics) {
    const baselineValue = baseline.metrics[metric];
    const currentValue = current.metrics[metric];
    const change = ((currentValue - baselineValue) / baselineValue) * 100;

    if (change > threshold) {
      regressions.push({
        metric,
        baseline: baselineValue,
        current: currentValue,
        change: `+${change.toFixed(1)}%`,
      });
    } else if (change < -threshold) {
      improvements.push({
        metric,
        baseline: baselineValue,
        current: currentValue,
        change: `${change.toFixed(1)}%`,
      });
    }
  }

  return {
    passed: regressions.length === 0,
    regressions,
    improvements,
  };
}
```

## Environment Requirements

### Test Environment Specifications

```markdown
## Test Environment Requirements

### Infrastructure
- Staging environment should mirror production:
  - Same instance types/sizes
  - Same database configuration
  - Same caching configuration
  - Same network topology

### Data
- Production-like data volumes
- Anonymized production data OR
- Generated data matching production patterns
- Database should be warmed (not empty)

### Isolation
- Dedicated test environment (not shared)
- No other traffic during tests
- Consistent state between test runs

### Monitoring
- Full observability stack enabled
- APM tracing active
- Database metrics visible
- Application logs accessible
```

### Test Data Management

```typescript
// Generate realistic test data
async function seedTestData() {
  // Create realistic user distribution
  const users = await generateUsers(10000);

  // Create realistic product catalog
  const products = await generateProducts(5000);

  // Create historical orders (for realistic DB size)
  const orders = await generateOrders(100000);

  // Warm caches
  await warmCaches();

  console.log('Test data seeded successfully');
}
```

## Reporting

### Load Test Report Template

```markdown
# Load Test Report

## Summary
- **Test Date:** 2024-01-15
- **Application Version:** 2.3.4
- **Environment:** Staging
- **Duration:** 30 minutes
- **Result:** ✅ PASSED

## Test Configuration
- **Scenario:** Standard load test
- **Virtual Users:** 100 (peak)
- **Ramp-up:** 5 minutes
- **Steady state:** 20 minutes
- **Ramp-down:** 5 minutes

## Results

### Response Times
| Metric | Target | Actual | Status |
| ------ | ------ | ------ | ------ |
| P50 | < 100ms | 85ms | ✅ |
| P95 | < 300ms | 245ms | ✅ |
| P99 | < 500ms | 420ms | ✅ |

### Throughput
| Metric | Target | Actual | Status |
| ------ | ------ | ------ | ------ |
| Requests/sec | > 500 | 623 | ✅ |
| Error Rate | < 1% | 0.3% | ✅ |

### Endpoint Breakdown
| Endpoint | RPS | P95 | Errors |
| -------- | --- | --- | ------ |
| GET /products | 312 | 180ms | 0.1% |
| GET /search | 156 | 350ms | 0.4% |
| POST /orders | 62 | 420ms | 0.5% |

## Resource Utilization
- **CPU Peak:** 65%
- **Memory Peak:** 78%
- **Database Connections:** 45/100

## Comparison to Baseline
- P95 improved by 8%
- Throughput unchanged
- No regressions detected

## Recommendations
1. Consider increasing database connection pool
2. Search endpoint approaching threshold - monitor closely
```

## Checklist

### Test Setup

- [ ] Test scenarios model real user behavior
- [ ] Performance thresholds defined
- [ ] Baseline established
- [ ] Test environment matches production
- [ ] Test data is production-like

### Execution

- [ ] Smoke tests pass before load tests
- [ ] Monitoring active during tests
- [ ] Tests run in isolated environment
- [ ] Results compared to baseline
- [ ] Reports generated and stored

### Process

- [ ] Load tests in CI/CD pipeline
- [ ] Pre-release performance gates
- [ ] Regular baseline updates
- [ ] Regression alerts configured
- [ ] Results reviewed in sprint

## References

- [k6 Documentation](https://k6.io/docs/)
- [Artillery Documentation](https://www.artillery.io/docs)
- [NBomber Documentation](https://nbomber.com/docs)
- [Google's Approach to Performance Testing](https://sre.google/sre-book/testing-reliability/)
- [Netflix Performance Testing](https://netflixtechblog.com/tagged/performance)
