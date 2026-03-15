# Incident Management

> Authoritative incident management standards for severity classification, on-call procedures, and post-mortems.

## Purpose

Establish clear procedures for detecting, responding to, and learning from incidents to minimize user impact and prevent recurrence.

## Core Principles

1. **Users first** - Minimize user impact above all else
2. **Communicate early** - Over-communicate during incidents
3. **Blameless culture** - Focus on systems, not individuals
4. **Learn continuously** - Every incident is a learning opportunity
5. **Automate response** - Reduce human intervention where possible
6. **Practice regularly** - Run game days and fire drills

## Severity Levels

### Severity Definitions

| Severity | Name | User Impact | Response Time | Examples |
| -------- | ---- | ----------- | ------------- | -------- |
| **SEV1** | Critical | Complete outage, data loss | 15 minutes | Service down, security breach, data corruption |
| **SEV2** | High | Major functionality degraded | 30 minutes | Core feature broken, significant slowdown |
| **SEV3** | Medium | Minor functionality affected | 2 hours | Non-critical feature broken, intermittent errors |
| **SEV4** | Low | Minimal impact | Next business day | Cosmetic issues, minor bugs |

### Severity Matrix

```
                    User Impact
                    Low    Medium    High    Critical
                 ┌──────┬─────────┬───────┬──────────┐
    Many Users   │ SEV3 │  SEV2   │ SEV1  │   SEV1   │
                 ├──────┼─────────┼───────┼──────────┤
    Some Users   │ SEV4 │  SEV3   │ SEV2  │   SEV1   │
                 ├──────┼─────────┼───────┼──────────┤
    Few Users    │ SEV4 │  SEV4   │ SEV3  │   SEV2   │
                 └──────┴─────────┴───────┴──────────┘
```

## Roles & Responsibilities

### Incident Commander (IC)

- Declares incident severity
- Coordinates response efforts
- Makes key decisions
- Communicates with stakeholders
- Ensures handoffs between shifts

### Technical Lead

- Leads technical investigation
- Coordinates debugging efforts
- Proposes and implements fixes
- Documents technical findings

### Communications Lead

- Updates status page
- Notifies affected users
- Coordinates internal communications
- Handles external inquiries

### Scribe

- Documents timeline of events
- Records decisions and actions
- Captures artifacts (logs, screenshots)
- Prepares post-incident report

## Incident Response Process

### Phase 1: Detection (0-5 minutes)

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Automated  │────▶│   Alert     │────▶│  On-Call    │
│  Monitoring │     │  Triggered  │     │  Notified   │
└─────────────┘     └─────────────┘     └─────────────┘
       │                                       │
       │            ┌─────────────┐            │
       └───────────▶│   User      │────────────┘
                    │   Report    │
                    └─────────────┘
```

### Phase 2: Triage (5-15 minutes)

```markdown
## Triage Checklist

- [ ] Acknowledge the alert
- [ ] Assess severity using severity matrix
- [ ] Declare incident in #incidents channel
- [ ] Page additional responders if needed
- [ ] Start incident document
- [ ] Update status page
```

### Phase 3: Investigation & Mitigation (15+ minutes)

```markdown
## Investigation Steps

1. **Scope the impact**
   - How many users affected?
   - Which regions/features?
   - When did it start?

2. **Identify recent changes**
   - Deployments in last 24 hours
   - Configuration changes
   - Infrastructure changes

3. **Form hypotheses**
   - What could cause these symptoms?
   - What data supports/refutes each theory?

4. **Implement mitigation**
   - Rollback if deployment-related
   - Scale resources if capacity-related
   - Enable feature flags if feature-related
```

### Phase 4: Resolution

```markdown
## Resolution Checklist

- [ ] Confirm user impact has ended
- [ ] Verify monitoring shows healthy state
- [ ] Update status page to resolved
- [ ] Notify stakeholders
- [ ] Schedule post-incident review
- [ ] Create follow-up tickets
```

## Communication Templates

### Status Page Update (Initial)

```markdown
**Investigating - [Service Name]**

We are currently investigating issues affecting [brief description].

**Impact:** [What users are experiencing]
**Status:** Investigating
**Started:** [Time] UTC

We will provide updates every [15/30] minutes.
```

### Status Page Update (Identified)

```markdown
**Identified - [Service Name]**

We have identified the cause of the issue affecting [service].

**Cause:** [Brief technical explanation]
**Mitigation:** [What we're doing]
**ETA:** [Expected resolution time, if known]

Next update in [X] minutes.
```

### Status Page Update (Resolved)

```markdown
**Resolved - [Service Name]**

The issue affecting [service] has been resolved.

**Duration:** [Start time] - [End time] UTC ([X] minutes)
**Impact:** [Summary of user impact]
**Resolution:** [Brief explanation]

We will publish a full incident report within [48 hours].
```

### Internal Slack Update

```markdown
🚨 **INCIDENT DECLARED** - [SEV1/2/3]

**Service:** [Affected service]
**Impact:** [User impact]
**IC:** @username
**Tech Lead:** @username
**Status Doc:** [link]

Please join #incident-[name] for coordination.
```

## Runbook Template

```markdown
# Runbook: [Service/Alert Name]

## Overview
Brief description of the service and what this runbook covers.

## Alert Triggers
- Alert: `[alert_name]`
- Threshold: [condition that triggers]
- Dashboard: [link]

## Quick Diagnosis

### Step 1: Check Service Health
\`\`\`bash
# Check if service is running
kubectl get pods -n [namespace] | grep [service]

# Check recent logs
kubectl logs -n [namespace] -l app=[service] --tail=100
\`\`\`

### Step 2: Check Dependencies
- Database: [link to dashboard]
- Cache: [link to dashboard]
- External APIs: [link to dashboard]

### Step 3: Recent Changes
\`\`\`bash
# List recent deployments
kubectl rollout history deployment/[service] -n [namespace]
\`\`\`

## Common Issues & Solutions

### Issue: High Memory Usage
**Symptoms:** OOM kills, slow responses
**Solution:**
1. Scale horizontally: `kubectl scale deployment [service] --replicas=X`
2. If persistent, investigate memory leaks

### Issue: Database Connection Errors
**Symptoms:** Connection refused, timeout errors
**Solution:**
1. Check connection pool: [dashboard link]
2. Verify database is healthy: [dashboard link]
3. Check for connection leaks in application

### Issue: High Latency
**Symptoms:** P99 > threshold, timeouts
**Solution:**
1. Check for slow queries: [dashboard link]
2. Verify cache hit rate: [dashboard link]
3. Check for resource contention

## Escalation

If unable to resolve within [X] minutes:
1. Page: [team/individual]
2. Contact: [phone/Slack]
3. Escalation Slack: #[channel]

## Rollback Procedure

\`\`\`bash
# Rollback to previous version
kubectl rollout undo deployment/[service] -n [namespace]

# Verify rollback
kubectl rollout status deployment/[service] -n [namespace]
\`\`\`

## Related Documentation
- Architecture: [link]
- Deployment Guide: [link]
- Contact: [team email/Slack]
```

## Post-Incident Review (PIR)

### PIR Document Template

```markdown
# Post-Incident Review: [Incident Title]

## Incident Summary

| Field | Value |
| ----- | ----- |
| **Date** | YYYY-MM-DD |
| **Duration** | X hours Y minutes |
| **Severity** | SEV[1-4] |
| **Services Affected** | [list] |
| **Users Impacted** | [number/percentage] |
| **Incident Commander** | [name] |

## Timeline (UTC)

| Time | Event |
| ---- | ----- |
| 14:00 | First alert triggered |
| 14:05 | On-call engineer acknowledged |
| 14:10 | Incident declared, IC assigned |
| 14:15 | Root cause identified |
| 14:30 | Fix deployed |
| 14:35 | Monitoring confirmed recovery |
| 14:40 | Incident resolved |

## Impact

### User Impact
- [X] users experienced [specific issue]
- [Y]% of requests failed
- [Z] minutes of degraded service

### Business Impact
- [Revenue impact, if applicable]
- [SLA impact, if applicable]
- [Reputation impact]

## Root Cause Analysis

### What Happened
[Clear, technical explanation of the failure]

### Why It Happened
Use the "5 Whys" technique:

1. **Why did X fail?** Because Y happened.
2. **Why did Y happen?** Because Z was misconfigured.
3. **Why was Z misconfigured?** Because we didn't have validation.
4. **Why didn't we have validation?** Because it wasn't in our checklist.
5. **Why wasn't it in our checklist?** Because we hadn't seen this failure mode.

### Contributing Factors
- [Factor 1: e.g., Missing monitoring]
- [Factor 2: e.g., Insufficient documentation]
- [Factor 3: e.g., Recent change without adequate testing]

## What Went Well
- Detection was fast (X minutes)
- Communication was clear
- Team collaboration was effective
- Rollback process worked smoothly

## What Could Be Improved
- Detection could be faster
- Runbook was missing key steps
- Communication to users was delayed
- No automated rollback

## Action Items

| Priority | Action | Owner | Due Date | Status |
| -------- | ------ | ----- | -------- | ------ |
| P1 | Add monitoring for [X] | @engineer | YYYY-MM-DD | TODO |
| P1 | Update runbook with [Y] | @engineer | YYYY-MM-DD | TODO |
| P2 | Implement automated rollback | @engineer | YYYY-MM-DD | TODO |
| P3 | Add chaos testing for [Z] | @engineer | YYYY-MM-DD | TODO |

## Lessons Learned

### Technical Lessons
- [Lesson 1]
- [Lesson 2]

### Process Lessons
- [Lesson 1]
- [Lesson 2]

## Appendix

### Relevant Logs
[Sanitized log snippets]

### Metrics/Graphs
[Screenshots of relevant dashboards]

### Related Incidents
- [Link to similar past incidents]
```

## On-Call Standards

### On-Call Expectations

```markdown
## On-Call Engineer Responsibilities

1. **Availability**
   - Respond to pages within 15 minutes
   - Have laptop and internet access at all times
   - Be within 30 minutes of a work-capable location

2. **Handoff Requirements**
   - Review ongoing incidents
   - Check recent deployments
   - Verify alerting is working
   - Update on-call rotation tool

3. **During Shift**
   - Monitor #alerts channel
   - Review and close non-actionable alerts
   - Document any issues encountered
   - Escalate if unable to resolve

4. **Shift End**
   - Write handoff notes
   - Ensure all incidents are resolved or handed off
   - Update tickets with status
```

### On-Call Rotation

```yaml
# Example PagerDuty-style rotation
rotations:
  primary:
    name: "Primary On-Call"
    schedule: weekly
    participants:
      - team_a
    escalation_timeout: 15m

  secondary:
    name: "Secondary On-Call"
    schedule: weekly
    participants:
      - team_b
    escalation_timeout: 30m

escalation_policies:
  default:
    - level: 1
      targets: [primary]
      timeout: 15m
    - level: 2
      targets: [secondary]
      timeout: 30m
    - level: 3
      targets: [engineering_manager]
```

## Tooling Requirements

### Required Tools

| Tool | Purpose | Examples |
| ---- | ------- | -------- |
| **Alerting** | Page on-call | PagerDuty, Opsgenie, VictorOps |
| **Status Page** | User communication | Statuspage.io, Atlassian Statuspage |
| **Incident Tracking** | Coordination | Slack, Incident.io, FireHydrant |
| **Runbooks** | Documented procedures | Notion, Confluence, GitHub |
| **Timeline** | Audit trail | Built into incident tool |

### Slack Channels

| Channel | Purpose |
| ------- | ------- |
| `#alerts` | Automated alerts |
| `#incidents` | Incident declarations |
| `#incident-[name]` | Per-incident coordination |
| `#postmortems` | PIR discussions |

## Checklist

### Incident Readiness

- [ ] On-call rotation established
- [ ] Alerting tools configured
- [ ] Status page set up
- [ ] Runbooks documented for critical services
- [ ] Communication templates ready
- [ ] Escalation paths defined

### During Incident

- [ ] Incident declared and roles assigned
- [ ] Status page updated
- [ ] Stakeholders notified
- [ ] Timeline being documented
- [ ] Regular updates provided

### Post-Incident

- [ ] PIR document created
- [ ] PIR meeting scheduled (within 48 hours)
- [ ] Action items created and assigned
- [ ] Status page updated to resolved
- [ ] Team retrospective completed

## References

- [Google SRE Book - Managing Incidents](https://sre.google/sre-book/managing-incidents/)
- [PagerDuty Incident Response](https://response.pagerduty.com/)
- [Atlassian Incident Management](https://www.atlassian.com/incident-management)
- [Post-Incident Review Best Practices](https://www.blameless.com/post-incident-review)
- [On-Call Handbook](https://increment.com/on-call/)
