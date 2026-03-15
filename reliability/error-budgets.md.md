# Error Budget Management

> Authoritative error budget standards for SLO enforcement, reliability targets, and burn rate management.

## Purpose

Establish a framework for balancing reliability investments against feature velocity using error budgets as the decision-making mechanism.

## Core Principles

1. **100% is the wrong target** - Pursuing perfect reliability has diminishing returns
2. **Budget enables risk** - Error budget lets teams take calculated risks
3. **Shared accountability** - SRE and development share reliability goals
4. **Data-driven decisions** - Budget consumption guides priorities
5. **Automate enforcement** - Make budget tracking automatic
6. **Continuous improvement** - Learn from budget consumption

## SLI/SLO/SLA Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│                     SLA (External)                           │
│         Legal commitment to customers                        │
│         Example: 99.5% availability                          │
├─────────────────────────────────────────────────────────────┤
│                     SLO (Internal)                           │
│         Internal target, stricter than SLA                   │
│         Example: 99.9% availability                          │
├─────────────────────────────────────────────────────────────┤
│                     SLI (Measurement)                        │
│         Actual metric being measured                         │
│         Example: Successful requests / Total requests        │
└─────────────────────────────────────────────────────────────┘

Error Budget = 100% - SLO
Example: 99.9% SLO = 0.1% error budget = 43.2 minutes/month downtime
```

## Service Level Indicators (SLIs)

### SLI Types

| SLI Type | Description | Calculation |
| -------- | ----------- | ----------- |
| **Availability** | Service is accessible | Successful requests / Total requests |
| **Latency** | Response time | Requests < threshold / Total requests |
| **Throughput** | Request handling capacity | Requests processed / Time window |
| **Error Rate** | Failed operations | Failed requests / Total requests |
| **Correctness** | Data accuracy | Correct responses / Total responses |

### SLI Specifications

```yaml
# sli-specification.yaml
slis:
  availability:
    name: "Service Availability"
    description: "Proportion of successful HTTP requests"
    calculation: |
      sum(rate(http_requests_total{status!~"5.."}[5m])) /
      sum(rate(http_requests_total[5m]))
    unit: "ratio"
    goodThreshold: 1

  latency:
    name: "Request Latency (P99)"
    description: "99th percentile request latency"
    calculation: |
      histogram_quantile(0.99,
        sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
      )
    unit: "seconds"
    goodThreshold: 0.5

  errorRate:
    name: "Error Rate"
    description: "Proportion of requests resulting in errors"
    calculation: |
      sum(rate(http_requests_total{status=~"5.."}[5m])) /
      sum(rate(http_requests_total[5m]))
    unit: "ratio"
    goodThreshold: 0
```

## Service Level Objectives (SLOs)

### SLO Configuration

```typescript
// slo-config.ts
interface SLO {
  name: string;
  sli: string;
  target: number; // 0-100 percentage
  window: '7d' | '28d' | '30d' | '90d';
  consequences: {
    alertThreshold: number; // Percentage of budget consumed
    freezeThreshold: number; // Percentage that triggers feature freeze
  };
}

const SERVICE_SLOS: SLO[] = [
  {
    name: 'API Availability',
    sli: 'availability',
    target: 99.9,
    window: '30d',
    consequences: {
      alertThreshold: 50, // Alert at 50% budget consumed
      freezeThreshold: 100, // Freeze at 100% budget consumed
    },
  },
  {
    name: 'API Latency (P99)',
    sli: 'latency_p99',
    target: 99.0, // 99% of requests under 500ms
    window: '30d',
    consequences: {
      alertThreshold: 50,
      freezeThreshold: 100,
    },
  },
  {
    name: 'Error Rate',
    sli: 'error_rate',
    target: 99.9, // < 0.1% errors
    window: '30d',
    consequences: {
      alertThreshold: 50,
      freezeThreshold: 100,
    },
  },
];
```

### SLO Target Selection

| Service Tier | Availability | Latency P99 | Error Budget (30 days) |
| ------------ | ------------ | ----------- | ---------------------- |
| Critical (payments, auth) | 99.99% | 200ms | 4.32 minutes |
| High (core features) | 99.9% | 500ms | 43.2 minutes |
| Standard (non-critical) | 99.5% | 1000ms | 3.6 hours |
| Best effort (internal tools) | 99.0% | 2000ms | 7.2 hours |

## Error Budget Calculation

### Budget Formula

```typescript
// Error budget calculation
interface ErrorBudget {
  slo: number; // e.g., 99.9
  windowDays: number; // e.g., 30
  totalMinutes: number;
  budgetMinutes: number;
  currentSli: number;
  consumedMinutes: number;
  remainingMinutes: number;
  burnRate: number;
}

function calculateErrorBudget(
  sloTarget: number,
  windowDays: number,
  currentSli: number,
): ErrorBudget {
  const totalMinutes = windowDays * 24 * 60;
  const budgetMinutes = totalMinutes * (1 - sloTarget / 100);
  const consumedMinutes = totalMinutes * (1 - currentSli / 100);
  const remainingMinutes = Math.max(0, budgetMinutes - consumedMinutes);
  const burnRate = consumedMinutes / budgetMinutes;

  return {
    slo: sloTarget,
    windowDays,
    totalMinutes,
    budgetMinutes,
    currentSli,
    consumedMinutes,
    remainingMinutes,
    burnRate,
  };
}

// Example: 99.9% SLO over 30 days
// Budget = 30 * 24 * 60 * 0.001 = 43.2 minutes
// If current SLI = 99.85%, consumed = 30 * 24 * 60 * 0.0015 = 64.8 minutes
// Budget consumed = 64.8 / 43.2 = 150% (over budget!)
```

### Monitoring Dashboard

```yaml
# prometheus-rules.yaml
groups:
  - name: error_budget
    rules:
      # Calculate remaining error budget
      - record: error_budget:remaining_ratio
        expr: |
          1 - (
            (1 - avg_over_time(sli:availability:ratio[30d])) /
            (1 - 0.999)
          )

      # Calculate burn rate
      - record: error_budget:burn_rate
        expr: |
          (1 - avg_over_time(sli:availability:ratio[1h])) /
          (1 - 0.999) * 720
          # 720 = hours in 30 days, so this shows "hours of budget burned per hour"

      # Alert on fast burn
      - alert: ErrorBudgetFastBurn
        expr: error_budget:burn_rate > 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Error budget burning 10x faster than sustainable"
          description: "At current rate, error budget will be exhausted in {{ $value | humanizeDuration }}"

      # Alert on budget exhaustion
      - alert: ErrorBudgetExhausted
        expr: error_budget:remaining_ratio < 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Error budget exhausted"
          description: "SLO has been breached. Feature freeze recommended."
```

## Budget Policies

### Consumption Thresholds

| Budget Consumed | Status | Actions |
| --------------- | ------ | ------- |
| 0-25% | Healthy | Normal development velocity |
| 25-50% | Caution | Review recent changes, increase monitoring |
| 50-75% | Warning | Prioritize reliability work, limit risky changes |
| 75-100% | Critical | Freeze non-critical features, focus on reliability |
| >100% | Breached | Feature freeze, incident response mode |

### Policy Actions

```markdown
## Error Budget Policy

### When Budget > 75% Remaining (Healthy)
- Normal development velocity
- Standard deployment frequency
- Experimentation encouraged

### When Budget 25-75% Remaining (Caution)
- Review SLI trends weekly
- Postmortems for all incidents
- Consider reliability improvements

### When Budget < 25% Remaining (Warning)
- Daily SLI review
- Require reliability component in new features
- Increase deployment scrutiny

### When Budget Exhausted (Critical)
- **Feature freeze** - No new features until budget recovers
- All hands on reliability improvements
- Daily standups on reliability status
- Executive escalation
```

### Budget Recovery

```typescript
// Budget recovery tracking
interface RecoveryPlan {
  currentBudget: number; // Current remaining %
  targetBudget: number; // Target to resume normal ops
  actions: RecoveryAction[];
  estimatedRecovery: Date;
}

interface RecoveryAction {
  action: string;
  owner: string;
  expectedImpact: string;
  dueDate: Date;
  status: 'pending' | 'in_progress' | 'completed';
}

// Example recovery plan
const recoveryPlan: RecoveryPlan = {
  currentBudget: -10, // 10% over budget
  targetBudget: 25, // Resume normal ops at 25%
  actions: [
    {
      action: 'Rollback deployment v2.3.4',
      owner: '@oncall',
      expectedImpact: 'Stop ongoing error spike',
      dueDate: new Date('2024-01-16'),
      status: 'completed',
    },
    {
      action: 'Fix database connection leak',
      owner: '@backend-team',
      expectedImpact: 'Reduce 5xx errors by 50%',
      dueDate: new Date('2024-01-18'),
      status: 'in_progress',
    },
    {
      action: 'Add circuit breaker to external API',
      owner: '@platform-team',
      expectedImpact: 'Prevent cascade failures',
      dueDate: new Date('2024-01-20'),
      status: 'pending',
    },
  ],
  estimatedRecovery: new Date('2024-01-25'),
};
```

## Burn Rate Alerts

### Multi-Window Alerting

```yaml
# Burn rate alerting based on Google SRE recommendations
groups:
  - name: burn_rate_alerts
    rules:
      # Fast burn - will exhaust budget in 2 hours
      - alert: SLOBurnRateCritical
        expr: |
          (
            error_budget:burn_rate:1h > 14.4
            and
            error_budget:burn_rate:5m > 14.4
          )
        for: 2m
        labels:
          severity: critical
          page: true
        annotations:
          summary: "Critical SLO burn rate"
          description: "Error budget will be exhausted in ~2 hours"

      # Medium burn - will exhaust budget in 6 hours
      - alert: SLOBurnRateHigh
        expr: |
          (
            error_budget:burn_rate:6h > 6
            and
            error_budget:burn_rate:30m > 6
          )
        for: 5m
        labels:
          severity: high
          page: true
        annotations:
          summary: "High SLO burn rate"
          description: "Error budget will be exhausted in ~6 hours"

      # Slow burn - will exhaust budget in 3 days
      - alert: SLOBurnRateMedium
        expr: |
          (
            error_budget:burn_rate:24h > 3
            and
            error_budget:burn_rate:2h > 3
          )
        for: 30m
        labels:
          severity: medium
        annotations:
          summary: "Elevated SLO burn rate"
          description: "Error budget will be exhausted in ~3 days"
```

### Burn Rate Visualization

```
Budget Consumption Over Time

100% ├────────────────────────────────────┤ SLO Breach
     │                              ████████
 75% ├─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─████████── │ Warning
     │                      ████████
 50% ├─ ─ ─ ─ ─ ─ ─ ─ ─████████─ ─ ─ ─ ─ │ Caution
     │                ██████
 25% ├─ ─ ─ ─ ─ ─████████─ ─ ─ ─ ─ ─ ─ ─ │ Healthy
     │          ██████
  0% └────██████─────────────────────────┘
     Day 1    7        14       21      30
```

## Reporting & Reviews

### Weekly Error Budget Report

```markdown
# Error Budget Report - Week of 2024-01-15

## Summary

| Service | SLO | Current SLI | Budget Used | Status |
| ------- | --- | ----------- | ----------- | ------ |
| API | 99.9% | 99.87% | 65% | ⚠️ Warning |
| Auth | 99.99% | 99.98% | 50% | ⚠️ Caution |
| Database | 99.9% | 99.95% | 25% | ✅ Healthy |

## Notable Events

### API Service
- **Incident on 2024-01-14**: Database connection timeout
  - Duration: 23 minutes
  - Budget impact: 40% of weekly budget
  - Root cause: Connection pool exhaustion
  - Action items: Increase pool size, add monitoring

## Trends

- API error budget consumption increased 20% week-over-week
- Auth service recovered 15% budget after fix deployed
- Database maintaining healthy margins

## Recommendations

1. Prioritize connection pool fix for API service
2. Consider adding redundancy to auth service
3. Review API SLO - may be too aggressive
```

### Monthly Error Budget Review

```markdown
# Monthly Error Budget Review - January 2024

## Executive Summary

| Metric | Target | Actual | Status |
| ------ | ------ | ------ | ------ |
| API Availability | 99.9% | 99.82% | ❌ Missed |
| API Latency P99 | 500ms | 420ms | ✅ Met |
| Auth Availability | 99.99% | 99.97% | ❌ Missed |

## Budget Analysis

### API Service
- **Total budget**: 43.2 minutes
- **Used**: 77.8 minutes (180%)
- **Major incidents**:
  - 2024-01-14: Database timeout (23 min)
  - 2024-01-22: Memory leak (35 min)
  - 2024-01-28: Deployment rollback (20 min)

### Root Causes
1. Infrastructure: 45%
2. Code defects: 35%
3. External dependencies: 20%

## Actions Taken
- Increased database connection pool
- Added memory monitoring and alerts
- Improved deployment validation

## Next Month Goals
1. Recover budget to 50% remaining
2. Implement circuit breakers
3. Add chaos testing for identified failure modes

## SLO Review
- Consider relaxing API SLO from 99.9% to 99.5%
- Add latency SLO for critical endpoints
```

## Checklist

### Setup

- [ ] SLIs defined for all services
- [ ] SLOs established with stakeholder input
- [ ] Error budget calculations automated
- [ ] Dashboards created
- [ ] Alerts configured for burn rates

### Process

- [ ] Budget policies documented
- [ ] Feature freeze criteria defined
- [ ] Weekly reports automated
- [ ] Monthly reviews scheduled
- [ ] Escalation paths documented

### Culture

- [ ] Teams understand error budgets
- [ ] Budget status visible to all
- [ ] Trade-offs discussed openly
- [ ] Reliability prioritized when budget low
- [ ] Blameless culture maintained

## References

- [Google SRE Book - Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
- [The Art of SLOs](https://sre.google/workbook/implementing-slos/)
- [Burn Rate Alerting](https://sre.google/workbook/alerting-on-slos/)
- [Error Budgets](https://sre.google/sre-book/embracing-risk/)
- [SLO Generator](https://github.com/google/slo-generator)
