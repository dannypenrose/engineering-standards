# Deployment Standard

> Authoritative deployment strategy standards aligned with continuous delivery best practices and zero-downtime patterns.

## Purpose

Define deployment strategies, procedures, and validation requirements that enable safe, rapid, and reliable releases with minimal user impact.

## Core Principles

1. **Zero-downtime deployment** - Services remain available during release
2. **Fast rollback** - Previous version can be restored in minutes
3. **Smoke testing** - Critical paths validated immediately post-deploy
4. **Incremental rollout** - Deploy to subset of users/servers first
5. **Feature flags** - Toggle features without code redeployment
6. **Pre-deployment validation** - Health and readiness checks before traffic

## Deployment Strategies

### Rolling Deployment

```
BEFORE:
[Pod A (v1)] [Pod B (v1)] [Pod C (v1)]

STEP 1: Replace one pod
[Pod A (v2)] [Pod B (v1)] [Pod C (v1)]  ← 1 new version

STEP 2: Replace another
[Pod A (v2)] [Pod B (v2)] [Pod C (v1)]  ← 2 new versions

STEP 3: Replace final
[Pod A (v2)] [Pod B (v2)] [Pod C (v2)]  ← All v2

ADVANTAGES: Simple, no extra capacity needed, natural fallback
DISADVANTAGES: Slow (per-pod timing), state management complexity
```

**Use case:** Stateless services with >= 3 replicas

### Blue-Green Deployment

```
BEFORE:
Load Balancer
    ├─ Blue   [Pod A (v1)] [Pod B (v1)] [Pod C (v1)]  ← Active
    └─ Green  [Old version decommissioned]

AFTER (Cut-over):
Load Balancer
    ├─ Blue   [Pod A (v1)] [Pod B (v1)] [Pod C (v1)]  ← Old (standby)
    └─ Green  [Pod A (v2)] [Pod B (v2)] [Pod C (v2)]  ← New (active)

INSTANT SWITCH (single LB config change)

ADVANTAGES: Instant rollback, full test environment, zero overlap
DISADVANTAGES: Requires 2x capacity, database schema changes complex
```

**Use case:** Database changes, breaking API changes, schema migrations

### Canary Deployment

```
BEFORE:
Load Balancer
  ├─ 95% traffic → [Pod A (v1)] [Pod B (v1)] [Pod C (v1)]
  └─ 5% traffic  → [Pod D (v1)]

AFTER (Initial):
Load Balancer
  ├─ 95% traffic → [Pod A (v1)] [Pod B (v1)] [Pod C (v1)]
  └─ 5% traffic  → [Pod D (v2)]  ← Canary (new version)

MONITOR: Error rates, latency, business metrics

AFTER (If healthy):
Load Balancer
  ├─ 50% traffic → [Pod A (v1)] [Pod B (v2)] [Pod C (v1)]
  └─ 50% traffic → [Pod D (v2)]

FINAL:
Load Balancer → [All v2]

ADVANTAGES: Risk mitigation, real user feedback, gradual rollout
DISADVANTAGES: Complex monitoring, slow rollout, state management
```

**Use case:** High-risk features, algorithm changes, performance-critical paths

## Deployment Procedure Checklist

### Pre-Deployment (T-5 minutes)

```markdown
## Pre-Deployment Validation

- [ ] All tests passing (unit, integration, e2e)
- [ ] Code review approved
- [ ] Dependency audit clean
- [ ] Database migrations tested on staging
- [ ] Feature flags configured for new features
- [ ] Rollback plan documented
- [ ] Team notified of deployment window
- [ ] Monitoring dashboards open
- [ ] Health check endpoints verified
```

### Deployment Phase

```bash
# 1. Health check - ensure staging is healthy
curl -s https://staging-api.example.com/health | jq .status
# Expected: "status": "healthy"

# 2. Build verification
npm run build && npm run test:smoke

# 3. Staging smoke tests
npm run test:smoke:staging
# All tests must pass

# 4. Trigger deployment workflow
git tag -a v1.2.3 -m "Release v1.2.3: New user dashboard"
git push origin v1.2.3

# GitHub Actions workflow triggered automatically
# Monitor: https://github.com/org/repo/actions/runs/...
```

### Post-Deployment (T+0 to T+10 minutes)

```markdown
## Post-Deployment Checks

- [ ] Service health check passes
- [ ] Error rates normal (< baseline + 1%)
- [ ] Latency p99 within SLO (< 500ms)
- [ ] New feature working for 5% of users (canary)
- [ ] Database connection pool healthy
- [ ] Cache hit rates normal
- [ ] No unusual log errors
- [ ] Smoke tests passing
```

### Validation Period (T+10 to T+30 minutes)

```bash
# Monitor key metrics
# Check Datadog/Prometheus dashboard

# Query: Error rate
sum(rate(http_requests_total{status="5xx"}[1m])) / sum(rate(http_requests_total[1m]))

# Query: P99 latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Query: Cache hit ratio
sum(cache_hits_total) / (sum(cache_hits_total) + sum(cache_misses_total))

# If any metric degrades > 5%:
#   1. Check logs for errors
#   2. Consider rollback
#   3. Investigate root cause
```

## Zero-Downtime Patterns

### Connection Draining

```nginx
# Nginx configuration for graceful shutdown
server {
    listen 3000;

    location / {
        proxy_pass http://backend;

        # Connection draining on backend restart
        proxy_read_timeout 30s;
        proxy_connect_timeout 5s;

        # Keepalive for reuse
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

### Readiness & Liveness Probes

```typescript
// Express health check endpoint
app.get('/health', (req, res) => {
  // Liveness: Is the process alive?
  // Returns immediately, no dependencies
  res.status(200).json({ status: 'alive' });
});

app.get('/ready', (req, res) => {
  // Readiness: Is the service ready for traffic?
  // Checks dependencies: database, cache, etc.
  const checks = await Promise.all([
    database.query('SELECT 1'),
    redis.ping(),
    // Check external dependencies
  ]);

  const ready = checks.every(c => c.success);
  res.status(ready ? 200 : 503).json({
    status: ready ? 'ready' : 'not_ready',
    checks: checks
  });
});

// Kubernetes config
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  template:
    spec:
      containers:
      - name: api
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Graceful Shutdown

```typescript
// Handle SIGTERM for clean shutdown
process.on('SIGTERM', async () => {
  logger.info('SIGTERM received, starting graceful shutdown...');

  // 1. Stop accepting new connections
  server.close(() => {
    logger.info('HTTP server closed');
  });

  // 2. Wait for in-flight requests (max 30 seconds)
  const gracefulShutdownTimer = setTimeout(() => {
    logger.error('Graceful shutdown timeout, forcing exit');
    process.exit(1);
  }, 30000);

  // 3. Complete in-flight database operations
  await database.end();

  // 4. Close caches
  await redis.quit();

  // 5. Cleanup complete
  clearTimeout(gracefulShutdownTimer);
  logger.info('Graceful shutdown complete');
  process.exit(0);
});

// Docker ENTRYPOINT should use exec form
# Dockerfile
ENTRYPOINT ["node", "server.js"]
# NOT: ENTRYPOINT ["sh", "-c", "node server.js"]
```

## Feature Flags in Deployment

### Simple Feature Flags

```typescript
// Feature flag evaluation
interface FeatureFlags {
  [key: string]: {
    enabled: boolean;
    rolloutPercentage?: number;  // 0-100
    allowedUsers?: string[];     // User IDs
  };
}

const flags: FeatureFlags = {
  'new-dashboard': {
    enabled: true,
    rolloutPercentage: 10  // 10% of users
  },
  'advanced-filters': {
    enabled: true,
    allowedUsers: ['user123', 'user456']
  }
};

function isFeatureEnabled(
  flagName: string,
  userId: string
): boolean {
  const flag = flags[flagName];
  if (!flag || !flag.enabled) return false;

  // Check allowlist
  if (flag.allowedUsers) {
    return flag.allowedUsers.includes(userId);
  }

  // Check percentage rollout
  if (flag.rolloutPercentage !== undefined) {
    const hash = hashFunction(`${flagName}:${userId}`);
    return (hash % 100) < flag.rolloutPercentage;
  }

  return true;
}

// Usage in routes
app.get('/api/dashboard', (req, res) => {
  if (isFeatureEnabled('new-dashboard', req.user.id)) {
    return res.json(dashboardV2Data);
  }
  return res.json(dashboardV1Data);
});
```

### Feature Flag Service

```typescript
// External feature flag service (LaunchDarkly, Unleash)
import { UnleashClient } from 'unleash-client';

const unleash = new UnleashClient({
  url: 'https://unleash.example.com',
  appName: 'api-service',
  customHeaders: {
    Authorization: process.env.UNLEASH_API_TOKEN
  }
});

// In route handler
const newDashboardEnabled = unleash.isEnabled('new-dashboard', {
  userId: req.user.id,
  environment: process.env.NODE_ENV
});

if (newDashboardEnabled) {
  // New code path
} else {
  // Old code path
}
```

## Environment Promotion

### Promotion Path

```
Commit → GitHub → GitHub Actions
                  ├─ Lint, Type-check
                  ├─ Tests
                  ├─ Build
                  └─ Push to Docker Registry

Manual Approval
    ↓
Deploy to Staging
    ├─ Same config/secrets as prod (except data)
    ├─ Run smoke tests
    ├─ Team validates
    └─ ~2 hours soak time

Manual Approval (Prod)
    ↓
Deploy to Production
    ├─ Canary: 5% traffic for 10 minutes
    ├─ Monitor metrics
    ├─ Gradual rollout to 100%
    └─ Full validation
```

### Staging Validation

```bash
# After deploy to staging
npm run test:smoke:staging

# Check critical user flows
- Login flow
- Create resource
- Update resource
- Delete resource
- Payment (if applicable)

# Performance benchmarks
- API response time < 500ms
- Error rate < 0.1%
- Cache hit ratio > 80%

# Data validation
- All tables accessible
- Foreign keys intact
- No orphaned records
```

## Rollback Procedures

### Quick Rollback (< 5 minutes)

```bash
# For rolling/canary deployments
# Use load balancer to shift traffic to previous version

# 1. Check current version
kubectl set image deployment/api \
  api=api:v1.2.2  # Previous version

# 2. Verify pods restarted
kubectl rollout status deployment/api

# 3. Run smoke tests
npm run test:smoke:production

# 4. Monitor metrics
# Ensure error rate returns to baseline
```

### Blue-Green Rollback (Instant)

```bash
# Load balancer configuration (instant switch)
# Current: Green (v1.2.3) is active
# Previous: Blue (v1.2.2) is standby

# Rollback: Switch traffic back to Blue
aws elbv2 modify-rule \
  --rule-arn arn:aws:elasticloadbalancing:... \
  --conditions Field=path-pattern,Values="/api/*" \
  --actions Type=forward,TargetGroupArn=arn:...:targetgroup/blue/...

# OR via configuration management
# Update load_balancer_target: blue
# Then: terraform apply
```

### Database Rollback

```sql
-- If deployment includes schema migration
-- 1. Verify rollback procedure exists BEFORE deploy
-- 2. Keep downgrade migration in code

-- Forward migration (new version)
ALTER TABLE users ADD COLUMN new_field VARCHAR(255);

-- Backward migration (previous version)
-- Safe to run while service is running old code
ALTER TABLE users DROP COLUMN new_field;

-- Always test rollback migration on staging first
psql staging_db -f migrations/downgrade_001.sql
```

## Smoke Testing

### Smoke Test Suite

```typescript
// tests/smoke.ts
describe('Smoke Tests - Critical Paths', () => {

  test('Health check responds', async () => {
    const res = await fetch('https://api.example.com/health');
    expect(res.status).toBe(200);
    expect(res.json()).toHaveProperty('status', 'healthy');
  });

  test('Authentication works', async () => {
    const token = await login('test@example.com', 'password');
    expect(token).toBeDefined();

    const res = await fetch('https://api.example.com/me', {
      headers: { Authorization: `Bearer ${token}` }
    });
    expect(res.status).toBe(200);
  });

  test('Payment endpoint responds', async () => {
    const res = await fetch('https://api.example.com/api/v1/payments');
    expect([200, 401]).toContain(res.status);
    // Don't test full flow, just endpoint availability
  });

  test('Database is accessible', async () => {
    const count = await db.query('SELECT COUNT(*) FROM users');
    expect(count).toBeGreaterThan(0);
  });
});

// Run post-deployment
npm run test:smoke:production
```

## Monitoring & Metrics

### Deployment Metrics

```yaml
# Prometheus metrics to track
deployment_duration_seconds
deployment_rollback_total
deployment_failure_total
deployment_traffic_shifted_percentage
service_health_status
error_rate_post_deployment
request_latency_p99_post_deployment
database_migration_duration_seconds
```

### Key Indicators

| Metric | Threshold | Action |
|--------|-----------|--------|
| Error rate increase | +1% | Check logs |
| Error rate increase | +5% | Consider rollback |
| Latency increase | +10% | Check resource usage |
| Latency increase | +25% | Consider rollback |
| Service availability | <99.9% | Immediate investigation |

## Checklist

### Pre-Deployment

- [ ] All tests passing
- [ ] Code review approved
- [ ] Database migrations tested
- [ ] Rollback procedure documented
- [ ] Team notified
- [ ] Feature flags configured
- [ ] Health endpoints verified

### Deployment

- [ ] Staging validated completely
- [ ] Smoke tests prepared
- [ ] Monitoring dashboards open
- [ ] Deployment strategy selected (rolling/blue-green/canary)
- [ ] Increment plan defined for canary

### Post-Deployment

- [ ] Smoke tests passing
- [ ] Error rates normal
- [ ] Latency within SLO
- [ ] Database connections healthy
- [ ] New features working for percentage of users
- [ ] 30-minute validation period completed

### Rollback Readiness

- [ ] Rollback procedure tested
- [ ] Previous version available
- [ ] Rollback can complete in < 5 minutes
- [ ] Team trained on rollback steps

## References

- [Continuous Delivery by Jez Humble](https://continuousdelivery.com/)
- [The Phoenix Project](https://itrevolution.com/the-phoenix-project/)
- [Blue-Green Deployments](https://martinfowler.com/bliki/BlueGreenDeployment.html)
- [Canary Deployments](https://martinfowler.com/bliki/CanaryRelease.html)
- [Kubernetes Deployment Strategy](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
