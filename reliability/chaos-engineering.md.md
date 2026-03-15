# Chaos Engineering

> Authoritative chaos engineering standards for fault injection, game days, and resilience validation.

## Purpose

Establish practices for proactively testing system resilience by introducing controlled failures, enabling teams to identify weaknesses before they cause production incidents.

## Core Principles

1. **Build hypothesis** - Define expected behavior before experiments
2. **Start small** - Begin with low-impact experiments
3. **Minimize blast radius** - Contain potential damage
4. **Run in production** - Real conditions yield real insights
5. **Automate experiments** - Continuous chaos catches regressions
6. **Learn and improve** - Every experiment should drive improvements

## Chaos Engineering Process

### The Scientific Method

```
┌─────────────────────────────────────────────────────────────┐
│  1. STEADY STATE                                             │
│     Define what "normal" looks like                          │
│     (metrics, behaviors, SLIs)                               │
├─────────────────────────────────────────────────────────────┤
│  2. HYPOTHESIS                                               │
│     "If X fails, the system should Y"                        │
│     Define expected resilient behavior                       │
├─────────────────────────────────────────────────────────────┤
│  3. EXPERIMENT                                               │
│     Introduce controlled failure                             │
│     Observe system behavior                                  │
├─────────────────────────────────────────────────────────────┤
│  4. VERIFY                                                   │
│     Compare actual vs expected behavior                      │
│     Document findings                                        │
├─────────────────────────────────────────────────────────────┤
│  5. IMPROVE                                                  │
│     Fix vulnerabilities discovered                           │
│     Update runbooks                                          │
│     Re-run experiment to verify fix                          │
└─────────────────────────────────────────────────────────────┘
```

### Experiment Template

```yaml
# chaos-experiment.yaml
name: "Database Connection Failure"
description: "Test system behavior when database becomes unavailable"

hypothesis:
  steady_state:
    - metric: "api_availability"
      condition: ">= 99.9%"
    - metric: "error_rate"
      condition: "< 0.1%"
  expected_behavior: |
    When the database connection is lost:
    1. API returns cached data for read requests (if available)
    2. Write requests return 503 with retry-after header
    3. Circuit breaker opens after 5 failures
    4. System recovers automatically when DB returns
    5. No data loss or corruption

experiment:
  type: "network"
  target: "database"
  action: "block_traffic"
  duration: "5m"
  rollback: "automatic"

blast_radius:
  affected_services: ["api", "worker"]
  affected_users: "5%" # If using canary
  geographic_scope: "us-east-1"

abort_conditions:
  - metric: "error_rate"
    threshold: "> 10%"
  - metric: "p99_latency"
    threshold: "> 5s"

schedule:
  frequency: "weekly"
  maintenance_window: "Tuesday 2-4 AM UTC"
```

## Failure Categories

### Infrastructure Failures

| Failure Type | Description | Tools |
| ------------ | ----------- | ----- |
| Instance termination | Kill compute instances | Chaos Monkey, Gremlin |
| Network partition | Block traffic between services | Toxiproxy, tc |
| Network latency | Add delay to network calls | Toxiproxy, tc |
| Packet loss | Drop percentage of packets | tc, Gremlin |
| DNS failure | Block DNS resolution | Custom scripts |
| Disk full | Fill disk to capacity | dd, Gremlin |
| Memory pressure | Consume available memory | stress-ng |
| CPU stress | Saturate CPU | stress-ng |

### Application Failures

| Failure Type | Description | Tools |
| ------------ | ----------- | ----- |
| Exception injection | Throw errors in code paths | Chaos toolkit, custom |
| Response delay | Slow down specific endpoints | Middleware, Toxiproxy |
| Response corruption | Return malformed responses | Custom middleware |
| Dependency failure | Fail calls to dependencies | Toxiproxy, WireMock |
| Feature flag flip | Sudden feature enable/disable | LaunchDarkly, custom |

### Data Failures

| Failure Type | Description | Tools |
| ------------ | ----------- | ----- |
| Database failover | Force primary/replica switch | Manual, chaos toolkit |
| Cache eviction | Clear all cached data | Redis CLI, custom |
| Stale data | Return outdated data | Custom middleware |
| Data corruption | Inject invalid data | Custom scripts |

## Implementation

### Chaos Toolkit Experiments

```yaml
# experiments/database-failure.yaml
version: 1.0.0
title: Database Connection Failure
description: Verify API graceful degradation when DB unavailable

steady-state-hypothesis:
  title: API is healthy
  probes:
    - name: api-responds
      type: probe
      tolerance: 200
      provider:
        type: http
        url: http://api.example.com/health
        timeout: 3

    - name: error-rate-normal
      type: probe
      tolerance:
        type: range
        range: [0, 0.01]
      provider:
        type: prometheus
        query: sum(rate(http_errors_total[5m])) / sum(rate(http_requests_total[5m]))

method:
  - name: block-database-traffic
    type: action
    provider:
      type: process
      path: iptables
      arguments: ["-A", "OUTPUT", "-p", "tcp", "--dport", "5432", "-j", "DROP"]
    pauses:
      after: 60

  - name: verify-graceful-degradation
    type: probe
    provider:
      type: http
      url: http://api.example.com/api/v1/products
      timeout: 10
    tolerance:
      status: [200, 503]

rollbacks:
  - name: restore-database-traffic
    type: action
    provider:
      type: process
      path: iptables
      arguments: ["-D", "OUTPUT", "-p", "tcp", "--dport", "5432", "-j", "DROP"]
```

### Toxiproxy for Network Chaos

```typescript
// chaos/toxiproxy-setup.ts
import Toxiproxy from 'toxiproxy-node-client';

const toxiproxy = new Toxiproxy('http://localhost:8474');

// Create proxy for database
async function setupDatabaseProxy() {
  const proxy = await toxiproxy.createProxy({
    name: 'postgres',
    listen: '0.0.0.0:5433',
    upstream: 'postgres:5432',
  });

  return proxy;
}

// Chaos experiments
async function runLatencyExperiment() {
  const proxy = await toxiproxy.get('postgres');

  // Add 500ms latency
  await proxy.addToxic({
    name: 'latency',
    type: 'latency',
    attributes: {
      latency: 500,
      jitter: 100,
    },
  });

  // Run for 5 minutes
  await sleep(300000);

  // Remove toxic
  await proxy.removeToxic('latency');
}

async function runConnectionResetExperiment() {
  const proxy = await toxiproxy.get('postgres');

  // Reset 50% of connections
  await proxy.addToxic({
    name: 'reset_peer',
    type: 'reset_peer',
    toxicity: 0.5,
    attributes: {
      timeout: 0,
    },
  });

  await sleep(60000);

  await proxy.removeToxic('reset_peer');
}
```

### Application-Level Chaos

```typescript
// middleware/chaos.middleware.ts
interface ChaosConfig {
  enabled: boolean;
  experiments: {
    latency?: {
      probability: number;
      minDelay: number;
      maxDelay: number;
    };
    errors?: {
      probability: number;
      statusCode: number;
    };
    timeout?: {
      probability: number;
    };
  };
}

@Injectable()
export class ChaosMiddleware implements NestMiddleware {
  private config: ChaosConfig;

  constructor(private configService: ConfigService) {
    this.config = configService.get('chaos');
  }

  async use(req: Request, res: Response, next: NextFunction) {
    if (!this.config.enabled) {
      return next();
    }

    // Latency injection
    if (this.config.experiments.latency) {
      const { probability, minDelay, maxDelay } = this.config.experiments.latency;
      if (Math.random() < probability) {
        const delay = minDelay + Math.random() * (maxDelay - minDelay);
        await sleep(delay);
      }
    }

    // Error injection
    if (this.config.experiments.errors) {
      const { probability, statusCode } = this.config.experiments.errors;
      if (Math.random() < probability) {
        return res.status(statusCode).json({
          error: 'Chaos experiment: injected error',
          code: 'CHAOS_ERROR',
        });
      }
    }

    // Timeout injection
    if (this.config.experiments.timeout) {
      const { probability } = this.config.experiments.timeout;
      if (Math.random() < probability) {
        // Just don't respond - let it timeout
        return;
      }
    }

    next();
  }
}
```

## Game Days

### Game Day Planning

```markdown
# Game Day: [Name]

## Overview
- **Date:** YYYY-MM-DD
- **Duration:** 2-4 hours
- **Participants:** [Team members]
- **Environment:** [Staging/Production subset]

## Objectives
1. Validate incident response procedures
2. Test specific failure scenarios
3. Identify gaps in monitoring
4. Practice cross-team communication

## Pre-Game Checklist
- [ ] Notify stakeholders of game day
- [ ] Ensure rollback procedures are ready
- [ ] Verify monitoring dashboards accessible
- [ ] Confirm communication channels ready
- [ ] Review abort criteria

## Scenarios

### Scenario 1: Database Failover
**Time:** 10:00 - 10:30
**Facilitator:** @engineer
**Expected outcome:** Automatic failover in < 30s

### Scenario 2: Service A Outage
**Time:** 10:30 - 11:00
**Facilitator:** @engineer
**Expected outcome:** Circuit breaker activates, graceful degradation

### Scenario 3: Cache Failure
**Time:** 11:00 - 11:30
**Facilitator:** @engineer
**Expected outcome:** Fallback to database, no errors

## Abort Criteria
- Error rate > 20%
- Customer impact reported
- Unable to roll back within 5 minutes

## Post-Game Review
- [ ] Document all findings
- [ ] Create tickets for improvements
- [ ] Update runbooks
- [ ] Schedule follow-up game day
```

### Game Day Runbook

```markdown
## Game Day Execution

### Before Starting
1. Announce in #engineering: "Game Day starting in 15 minutes"
2. Verify all participants are online
3. Open monitoring dashboards
4. Have rollback procedures ready

### During Scenarios
1. Facilitator announces scenario
2. Execute failure injection
3. Observe system behavior
4. Note any unexpected behavior
5. Execute rollback after observation period

### If Abort Needed
1. Announce "ABORT" in game day channel
2. Execute immediate rollback
3. Verify system recovery
4. Document what triggered abort

### After Each Scenario
1. Review what happened vs expected
2. Capture screenshots of dashboards
3. Note any action items
4. 5-minute break before next scenario
```

## Continuous Chaos

### Automated Chaos Pipeline

```yaml
# .github/workflows/chaos.yml
name: Chaos Engineering

on:
  schedule:
    - cron: '0 3 * * 2' # Tuesday at 3 AM

jobs:
  chaos-experiments:
    runs-on: ubuntu-latest
    environment: chaos
    steps:
      - uses: actions/checkout@v4

      - name: Setup Chaos Toolkit
        run: pip install chaostoolkit chaostoolkit-kubernetes

      - name: Notify start
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -d '{"text": "🔥 Starting chaos experiments"}'

      - name: Run experiments
        run: |
          chaos run experiments/database-failure.yaml
          chaos run experiments/cache-failure.yaml
          chaos run experiments/network-latency.yaml
        continue-on-error: true

      - name: Upload results
        uses: actions/upload-artifact@v4
        with:
          name: chaos-results
          path: chaos-report.json

      - name: Notify completion
        if: always()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -d '{"text": "Chaos experiments completed. Check results."}'
```

### Chaos Monkey Configuration (Kubernetes)

```yaml
# chaos-monkey.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: Schedule
metadata:
  name: pod-failure-schedule
spec:
  schedule: "0 3 * * *"
  type: PodChaos
  historyLimit: 5
  concurrencyPolicy: Forbid
  podChaos:
    action: pod-failure
    mode: one
    selector:
      namespaces:
        - production
      labelSelectors:
        "chaos-enabled": "true"
    duration: "5m"
---
apiVersion: chaos-mesh.org/v1alpha1
kind: Schedule
metadata:
  name: network-delay-schedule
spec:
  schedule: "30 3 * * *"
  type: NetworkChaos
  networkChaos:
    action: delay
    mode: all
    selector:
      namespaces:
        - production
      labelSelectors:
        "chaos-enabled": "true"
    delay:
      latency: "200ms"
      correlation: "50"
      jitter: "50ms"
    duration: "5m"
```

## Maturity Model

### Chaos Engineering Maturity Levels

| Level | Description | Practices |
| ----- | ----------- | --------- |
| **0 - None** | No chaos engineering | N/A |
| **1 - Initial** | Manual experiments, staging only | Manual failure injection, basic documentation |
| **2 - Managed** | Planned game days, production experiments | Scheduled experiments, incident correlation |
| **3 - Defined** | Automated chaos, continuous verification | CI/CD integration, automated rollbacks |
| **4 - Optimized** | Proactive chaos, predictive resilience | ML-based experiment selection, auto-remediation |

### Progression Checklist

```markdown
## Level 1 → Level 2
- [ ] Document all known failure modes
- [ ] Create hypothesis for each failure mode
- [ ] Run first production chaos experiment
- [ ] Establish game day cadence (quarterly)
- [ ] Create chaos experiment library

## Level 2 → Level 3
- [ ] Automate experiment execution
- [ ] Integrate chaos into CI/CD
- [ ] Implement automatic abort conditions
- [ ] Track chaos experiment coverage
- [ ] Automate rollback procedures

## Level 3 → Level 4
- [ ] Use production traffic for canary chaos
- [ ] Implement auto-remediation for known failures
- [ ] Correlate chaos results with incidents
- [ ] Predict failure modes from metrics
- [ ] Continuous chaos in production
```

## Safety Measures

### Blast Radius Control

```typescript
// Limit chaos to percentage of traffic
const CHAOS_CONFIG = {
  // Only affect 5% of requests
  trafficPercentage: 5,

  // Only during business hours (for monitoring)
  allowedHours: { start: 9, end: 17 },

  // Exclude critical paths
  excludedPaths: [
    '/health',
    '/api/v1/payments',
    '/api/v1/auth',
  ],

  // Automatic abort thresholds
  abortConditions: {
    errorRate: 0.1,      // 10% error rate
    latencyP99: 5000,    // 5 second P99
    affectedUsers: 100,  // 100 users seeing errors
  },
};
```

### Kill Switch

```typescript
// Immediate chaos shutdown
class ChaosKillSwitch {
  private redis: Redis;

  async activate(reason: string): Promise<void> {
    await this.redis.set('chaos:killed', 'true');
    await this.redis.set('chaos:kill_reason', reason);
    await this.redis.set('chaos:kill_time', Date.now());

    // Alert on-call
    await this.alerting.send({
      severity: 'critical',
      title: 'Chaos Kill Switch Activated',
      description: reason,
    });
  }

  async isKilled(): Promise<boolean> {
    return (await this.redis.get('chaos:killed')) === 'true';
  }

  async deactivate(): Promise<void> {
    await this.redis.del('chaos:killed');
  }
}

// Check before any chaos action
async function runChaosAction(action: ChaosAction): Promise<void> {
  if (await killSwitch.isKilled()) {
    console.log('Chaos is disabled via kill switch');
    return;
  }

  await action.execute();
}
```

## Checklist

### Getting Started

- [ ] Identify critical services for chaos testing
- [ ] Document expected resilient behaviors
- [ ] Set up chaos testing environment
- [ ] Define abort criteria
- [ ] Get stakeholder buy-in

### Experiment Design

- [ ] Clear hypothesis defined
- [ ] Steady state metrics identified
- [ ] Blast radius controlled
- [ ] Rollback procedure documented
- [ ] Communication plan ready

### Execution

- [ ] Monitoring dashboards ready
- [ ] On-call aware of experiment
- [ ] Kill switch accessible
- [ ] Results documented
- [ ] Action items created

## References

- [Principles of Chaos Engineering](https://principlesofchaos.org/)
- [Netflix Chaos Engineering](https://netflixtechblog.com/tagged/chaos-engineering)
- [Chaos Toolkit](https://chaostoolkit.org/)
- [Gremlin Documentation](https://www.gremlin.com/docs/)
- [Chaos Mesh](https://chaos-mesh.org/)
- [AWS Fault Injection Simulator](https://aws.amazon.com/fis/)
- [Google DiRT](https://sre.google/sre-book/accelerating-sre-on-call/)
