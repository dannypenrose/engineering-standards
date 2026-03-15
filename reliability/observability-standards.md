# Observability Standards

> Authoritative observability standards for logging, metrics, tracing, and alerting across all services.

## Purpose

Establish comprehensive observability practices that enable teams to understand system behavior, diagnose issues quickly, and maintain reliable services.

## Core Principles

1. **Three pillars** - Logs, metrics, and traces working together
2. **Structured data** - Machine-parseable formats for all telemetry
3. **Context propagation** - Trace IDs flow through all systems
4. **Actionable alerts** - Every alert should have a clear response
5. **Minimal performance impact** - Observability shouldn't degrade UX
6. **Privacy by design** - No PII in telemetry data

## The Three Pillars

```
┌─────────────────────────────────────────────────────────────────┐
│                        OBSERVABILITY                            │
├─────────────────┬─────────────────┬─────────────────────────────┤
│      LOGS       │     METRICS     │          TRACES             │
├─────────────────┼─────────────────┼─────────────────────────────┤
│ What happened   │ How much/many   │ How requests flow           │
│ Discrete events │ Numeric values  │ Request journey             │
│ High cardinality│ Low cardinality │ Distributed context         │
│ Debug & audit   │ Trends & alerts │ Latency & dependencies      │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

## Logging Standards

### Log Levels

| Level | Usage | Example |
| ----- | ----- | ------- |
| ERROR | Failures requiring attention | Database connection failed |
| WARN | Potentially harmful situations | Rate limit approaching |
| INFO | Significant business events | User created, order placed |
| DEBUG | Detailed diagnostic info | Request parameters, SQL queries |
| TRACE | Very detailed debugging | Function entry/exit |

### Structured Logging Format

```typescript
// Use structured JSON logging
const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level: (label) => ({ level: label }),
  },
  timestamp: pino.stdTimeFunctions.isoTime,
});

// Log with context
logger.info({
  event: 'user.created',
  userId: user.id,
  email: maskEmail(user.email),
  source: 'registration',
  duration: performance.now() - startTime,
}, 'User created successfully');
```

### Standard Log Fields

```json
{
  "timestamp": "2024-01-23T10:30:00.000Z",
  "level": "info",
  "message": "User created successfully",
  "service": "user-service",
  "version": "1.2.3",
  "environment": "production",
  "traceId": "abc123def456",
  "spanId": "span789",
  "event": "user.created",
  "userId": "usr_123",
  "duration": 45.2,
  "host": "pod-xyz"
}
```

### PII Handling

```typescript
// Mask sensitive data before logging
function maskEmail(email: string): string {
  const [local, domain] = email.split('@');
  return `${local[0]}***@${domain}`;
}

function maskCard(card: string): string {
  return `****${card.slice(-4)}`;
}

// NEVER log
const NEVER_LOG = ['password', 'token', 'apiKey', 'ssn', 'creditCard'];
```

## Metrics Standards

### Metric Types

| Type | Usage | Example |
| ---- | ----- | ------- |
| Counter | Monotonically increasing | http_requests_total |
| Gauge | Current value | active_connections |
| Histogram | Distribution of values | http_request_duration_seconds |
| Summary | Similar to histogram | request_latency |

### Naming Conventions

```
# Format: <namespace>_<name>_<unit>

# Good
http_requests_total
http_request_duration_seconds
db_connections_active
cache_hit_ratio

# Bad
requests              # No namespace
httpRequestsTotal     # Not snake_case
request_time          # Ambiguous unit
```

### Required Application Metrics

```typescript
// RED metrics (Rate, Errors, Duration)
const metrics = {
  // Rate - requests per second
  httpRequestsTotal: new Counter({
    name: 'http_requests_total',
    help: 'Total HTTP requests',
    labelNames: ['method', 'path', 'status'],
  }),

  // Errors - error rate
  httpRequestErrorsTotal: new Counter({
    name: 'http_request_errors_total',
    help: 'Total HTTP request errors',
    labelNames: ['method', 'path', 'error_type'],
  }),

  // Duration - latency
  httpRequestDurationSeconds: new Histogram({
    name: 'http_request_duration_seconds',
    help: 'HTTP request duration in seconds',
    labelNames: ['method', 'path'],
    buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5],
  }),
};
```

### USE Metrics (Resources)

```typescript
// Utilization, Saturation, Errors for resources
const resourceMetrics = {
  // CPU
  cpuUsagePercent: new Gauge({...}),

  // Memory
  memoryUsageBytes: new Gauge({...}),
  memoryLimitBytes: new Gauge({...}),

  // Database connections
  dbConnectionsActive: new Gauge({...}),
  dbConnectionsMax: new Gauge({...}),

  // Queue
  queueDepth: new Gauge({...}),
  queueProcessingSeconds: new Histogram({...}),
};
```

## Distributed Tracing

### Trace Context Propagation

```typescript
// Express middleware for trace context
import { trace, context, propagation } from '@opentelemetry/api';

app.use((req, res, next) => {
  const parentContext = propagation.extract(context.active(), req.headers);

  const tracer = trace.getTracer('my-service');
  const span = tracer.startSpan(
    `${req.method} ${req.path}`,
    { kind: SpanKind.SERVER },
    parentContext
  );

  // Add trace ID to logs
  req.traceId = span.spanContext().traceId;

  context.with(trace.setSpan(context.active(), span), () => {
    res.on('finish', () => {
      span.setStatus({
        code: res.statusCode >= 400 ? SpanStatusCode.ERROR : SpanStatusCode.OK,
      });
      span.end();
    });
    next();
  });
});
```

### Span Attributes

```typescript
// Add meaningful attributes to spans
span.setAttributes({
  'http.method': req.method,
  'http.url': req.url,
  'http.status_code': res.statusCode,
  'user.id': req.user?.id,
  'db.system': 'postgresql',
  'db.operation': 'SELECT',
  'db.statement': 'SELECT * FROM users WHERE id = ?',
});
```

## Alerting Standards

### Alert Severity Levels

| Level | Response Time | Examples |
| ----- | ------------- | -------- |
| P1 - Critical | 15 minutes | Service down, data loss |
| P2 - High | 1 hour | Degraded performance, partial outage |
| P3 - Medium | 4 hours | Non-critical errors, capacity warning |
| P4 - Low | Next business day | Optimization opportunities |

### Alert Rules

```yaml
# Good alert - actionable and specific
- alert: HighErrorRate
  expr: |
    (sum(rate(http_request_errors_total[5m])) /
     sum(rate(http_requests_total[5m]))) > 0.05
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "High error rate detected"
    description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"
    runbook: "https://wiki.example.com/runbooks/high-error-rate"

# Bad alert - too sensitive, no context
- alert: AnyError
  expr: http_request_errors_total > 0  # Fires on ANY error
```

### Alert Checklist

Every alert should have:

- [ ] Clear, descriptive name
- [ ] Appropriate severity level
- [ ] Threshold that avoids false positives
- [ ] `for` duration to prevent flapping
- [ ] Runbook link
- [ ] Clear description with current value

## SLIs, SLOs, and SLAs

### Service Level Indicators (SLIs)

```yaml
# Availability SLI
availability: |
  sum(rate(http_requests_total{status!~"5.."}[30d])) /
  sum(rate(http_requests_total[30d]))

# Latency SLI (P99)
latency_p99: |
  histogram_quantile(0.99,
    sum(rate(http_request_duration_seconds_bucket[30d])) by (le))

# Error rate SLI
error_rate: |
  sum(rate(http_request_errors_total[30d])) /
  sum(rate(http_requests_total[30d]))
```

### Service Level Objectives (SLOs)

| SLI | Target | Error Budget (30 days) |
| --- | ------ | ---------------------- |
| Availability | 99.9% | 43.2 minutes downtime |
| P99 Latency | < 500ms | 0.1% of requests slower |
| Error Rate | < 0.1% | 0.1% of requests can fail |

## Dashboard Standards

### Required Dashboards

1. **Service Overview** - Health at a glance
2. **Request Flow** - Traffic patterns and latency
3. **Error Analysis** - Error rates and types
4. **Resource Usage** - CPU, memory, connections
5. **Business Metrics** - KPIs relevant to service

### Dashboard Layout

```
┌─────────────────────────────────────────────────────────────────┐
│                    SERVICE HEALTH OVERVIEW                       │
├─────────────────┬─────────────────┬─────────────────────────────┤
│   Availability  │    Error Rate   │       P99 Latency           │
│     99.95%      │      0.02%      │         245ms               │
├─────────────────┴─────────────────┴─────────────────────────────┤
│                     REQUEST RATE (last 24h)                      │
│  [════════════════════════════════════════════════════════]     │
├─────────────────────────────────────────────────────────────────┤
│                     ERROR RATE (last 24h)                        │
│  [════════════════════════════════════════════════════════]     │
├─────────────────┬───────────────────────────────────────────────┤
│  Top Errors     │            Latency Distribution               │
│  - 404: 45%     │  [histogram]                                  │
│  - 500: 30%     │                                               │
│  - 503: 25%     │                                               │
└─────────────────┴───────────────────────────────────────────────┘
```

## Checklist

### Logging

- [ ] Structured JSON logging enabled
- [ ] Log levels used appropriately
- [ ] Trace IDs included in all logs
- [ ] PII masked or excluded
- [ ] Log retention configured

### Metrics

- [ ] RED metrics implemented
- [ ] USE metrics for resources
- [ ] Consistent naming conventions
- [ ] Appropriate cardinality
- [ ] Prometheus/compatible endpoint

### Tracing

- [ ] Distributed tracing enabled
- [ ] Context propagation configured
- [ ] Meaningful span attributes
- [ ] Sampling configured for production

### Alerting

- [ ] Critical paths have alerts
- [ ] Alerts have runbooks
- [ ] Alert fatigue minimized
- [ ] Escalation paths defined

## References

- [Google SRE Book](https://sre.google/sre-book/table-of-contents/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Prometheus Best Practices](https://prometheus.io/docs/practices/)
- [The RED Method](https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/)
- [The USE Method](https://www.brendangregg.com/usemethod.html)
