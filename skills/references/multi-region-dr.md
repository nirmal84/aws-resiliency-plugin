# Multi-Region DR — Patterns, RTO/RPO, ORR, Game Day

Load this file when the user asks about disaster recovery, multi-region architecture, failover planning, RTO/RPO targets, operational readiness reviews, or game day exercises.

---

## DR Pattern Selection

Choose the pattern based on the business RTO/RPO requirement — not the other way around.

| Pattern | RTO | RPO | Cost Multiplier | Use When |
|---------|-----|-----|-----------------|----------|
| **Backup & Restore** | 1–4 hours | 1–24 hours | 1× | Dev/test, non-critical, archive workloads |
| **Pilot Light** | 10–30 min | Minutes | 1.5–2× | Core services tolerant of brief outage; moderate budget |
| **Warm Standby** | 1–10 min | Seconds | 2–3× | Business-critical; moderate-high budget; wagering/payments |
| **Active-Active** | Near-zero | Near-zero | 2–4× | Mission-critical, revenue-generating, zero downtime requirement |

> **Key question to ask first:** "If your platform went completely down right now, what is the business impact per minute? Has that number been formally calculated and presented to leadership?" Most teams guess RTO/RPO without this anchor.

---

## Pattern 1: Backup & Restore

**IaC pattern:**
- RDS automated backups + manual snapshots → separate account or region
- S3 cross-region replication (CRR) or periodic export
- EC2 AMI snapshots via AWS Backup with cross-region copy
- No standby infrastructure — rebuild on demand from IaC

**Recovery sequence:**
1. Declare DR event
2. Apply IaC in DR region (~20–45 min depending on complexity)
3. Restore latest DB snapshot (~20 min for small DBs; hours for large)
4. Update DNS to DR region (~2 min for Route53; propagation adds 60–120s)
5. Validate and open traffic

**Key gaps to flag:**
- IaC must be tested in DR region — "it deploys in primary" ≠ "it deploys in DR"
- AMI IDs are region-specific — cross-region AMI copies or public AMIs required
- RDS snapshot restore creates a new instance — application connection strings must be configurable
- Without cross-account backups, account compromise = production and backups gone simultaneously

---

## Pattern 2: Pilot Light

**IaC pattern:**
- Core data tier always running: RDS Multi-AZ in DR region **or** Aurora Global DB secondary
- Compute at minimum: ASG `minSize: 0, desiredCapacity: 1` or Fargate at 1 task
- Route53 failover routing with health checks — secondary record exists but primary is weighted 100/0
- On failover: scale up compute, cut DNS

**CDK pattern:**
```typescript
// Primary region: Route53 failover primary
new route53.ARecord(this, 'PrimaryRecord', {
  zone,
  target: route53.RecordTarget.fromAlias(new targets.LoadBalancerTarget(primaryAlb)),
  recordName: 'api.myapp.com',
}).node.defaultChild.setProperty('Failover', 'PRIMARY');
new route53.CfnHealthCheck(this, 'PrimaryHealthCheck', {
  healthCheckConfig: {
    type: 'HTTPS',
    fullyQualifiedDomainName: 'api.myapp.com',
    resourcePath: '/health/live',
    requestInterval: 30,
    failureThreshold: 3,
  },
});

// DR region: Route53 failover secondary (same stack, different region)
new route53.ARecord(this, 'SecondaryRecord', {
  zone,
  target: route53.RecordTarget.fromAlias(new targets.LoadBalancerTarget(drAlb)),
  recordName: 'api.myapp.com',
}).node.defaultChild.setProperty('Failover', 'SECONDARY');
```

**Key gaps to flag:**
- Compute scale-up time (5–10 min) + health check grace period adds to RTO — test this
- DB promotion lag: Aurora Global DB managed failover ~1 min; standard RDS cross-region read replica = manual promote
- Route53 failover takes 90–120s — add to RTO calculation

---

## Pattern 3: Warm Standby

**IaC pattern:**
- Full stack in DR region at reduced capacity (e.g., 25% of production capacity)
- Aurora Global Database: RPO ~1 second; RTO ~1 minute (managed failover)
- DynamoDB Global Tables: active-active replication with last-writer-wins
- ALB + ASG at minimum viable capacity in DR region
- Route53 weighted routing (primary 100, secondary 0) **or** failover routing
- On failover: scale up DR region, cut Route53 traffic

**Key gaps to flag:**
- "Warm" means you must validate the DR stack regularly — a stack that was healthy 3 months ago may have drifted
- Aurora Global DB unplanned failover: requires manual `PromoteReadReplicaDBCluster` — automation reduces RTO dramatically
- DynamoDB Global Tables: writes go to local region; reads can be from any region; conflict resolution (last-writer-wins) must be designed for — not all data models tolerate this
- Application connection strings must be environment-variable driven — hardcoded primary region endpoints won't automatically switch

---

## Pattern 4: Active-Active (Multi-Region)

**IaC pattern:**
- Identical full stack in ≥ 2 regions, both serving live traffic
- Global Accelerator (anycast) or Route53 latency routing distributing traffic
- DynamoDB Global Tables or Aurora Global Database with write forwarding
- CloudFront with multi-region origin group
- Route53 ARC (Application Recovery Controller) for surgical traffic management

**Design constraints (must validate in code review):**
- **Data conflict resolution**: Last-writer-wins (DynamoDB Global Tables) — application must handle eventual consistency; writes from different regions may conflict
- **Shared session state**: Session must be stored in shared tier (DynamoDB Global Tables, ElastiCache Global Datastore) or application must be stateless
- **Deployment pipeline**: Deployments must roll out across regions — staggered region-by-region, not simultaneous
- **Third-party dependencies**: If payment gateway, identity provider, or SMS API only operates in one region — active-active for the application but single-region dependency defeats the purpose

**Automated failover timeline:**
```
T+0s:   Regional failure (AZ or full region)
T+10s:  Global Accelerator detects failure, reroutes traffic (anycast)
T+30s:  Route53 health check detects failure (standard interval)
T+90s:  Route53 DNS updates propagated
```
Global Accelerator provides faster and more reliable failover than Route53 alone for TCP traffic.

---

## RTO/RPO Calculation Guide

### RTO Breakdown
```
Actual RTO = Detection time
           + Escalation time (automated alert → human receives)
           + Decision time (human decides to declare DR)
           + Execution time (each manual step adds 5–15 min uncertainty)
           + Validation time (confirm service is healthy before opening traffic)

Example:
  Detection:   90 seconds  (Route53 health check)
  Escalation:   3 minutes  (PagerDuty → on-call phone)
  Decision:     5 minutes  (confirm it's not transient)
  Execution:   15 minutes  (automated DB promotion + DNS cut)
  Validation:   5 minutes  (smoke tests, error rate check)
  ─────────────────────────
  Actual RTO: ~30 minutes

Each additional manual step: +5 to +15 minutes of uncertainty
```

### RPO Breakdown
```
Actual RPO = Replication lag at time of failure
           + Time since last backup (for backup/restore patterns)

Aurora Global Database:  ~1 second replication lag (RPO ~1s)
DynamoDB Global Tables:  Sub-second to seconds under high write load
RDS Multi-AZ (same region): Synchronous replication (RPO ~0s for failover)
RDS cross-region read replica: Async replication (RPO = current replication lag)
S3 CRR (standard):       Minutes (best-effort)
S3 CRR with RTC:         15 minutes (SLA-backed)
```

---

## Operational Readiness Review (ORR) Checklist

Use this before any production launch or major traffic event (sports finals, sale events, product launches).

### Architecture
- [ ] Multi-AZ deployment for all stateful services
- [ ] Auto Scaling configured and tested for expected peak + 50% headroom
- [ ] Load tested to 150% of expected peak traffic
- [ ] Single points of failure identified and remediated (or accepted with documented risk)
- [ ] Blast radius of each component failure understood

### Data
- [ ] Backup policy defined with RTO/RPO targets
- [ ] Backup restore tested in last 30 days (with actual timing recorded — not estimated)
- [ ] PITR enabled on RDS and DynamoDB
- [ ] Database failover tested (Multi-AZ for RDS; cluster failover for Aurora)
- [ ] Replication lag alarmed

### Operations
- [ ] On-call rotation established with primary and secondary coverage
- [ ] Runbooks written for top 5 failure scenarios
- [ ] Runbooks tested in staging (table-top is not enough)
- [ ] Every CloudWatch alarm has a runbook link in its description
- [ ] Communication plan documented (internal escalation + customer comms)
- [ ] Change freeze window defined for event period (if applicable)
- [ ] DR procedure tested in last 90 days (Tier 1) or last 6 months (Tier 2)

### Monitoring
- [ ] Alarms on all Tier 1 metrics with on-call action
- [ ] Business metric alarms configured (transaction rate, not just CPU)
- [ ] `TreatMissingData: BREACHING` on all critical alarms
- [ ] AWS Health events subscribed via EventBridge
- [ ] Dashboards prepared for incident response (pre-bookmark)

---

## Game Day Exercise Template

### Pre-Game Day (2 weeks before)

**Define the scenario and hypothesis:**
```
Scenario:    [What failure are we simulating?]
Hypothesis:  When [failure], we expect [system behaviour] within [time].
Scope:       [Which service/region/AZ?]
Method:      [AWS FIS / manual API call / network NACL block]
```

**Preparation checklist:**
- [ ] Hypothesis documented and reviewed
- [ ] Stop conditions defined (CloudWatch alarm thresholds that auto-halt experiment)
- [ ] Rollback plan documented (if game day causes unintended impact)
- [ ] Stakeholders notified (especially for production game days)
- [ ] On-call availability confirmed for experiment window
- [ ] Monitoring dashboards pre-loaded and bookmarked
- [ ] Communication channel (Slack/Teams war room) established

**Success criteria (define before — not after — the experiment):**
- RTO target: _____ minutes
- RPO target: _____ seconds/minutes of data loss acceptable
- Peak error rate acceptable: < _____% for _____ minutes
- All critical alarms fired: Yes / No

### Game Day Execution

**Common AWS FIS experiments:**
```bash
# AZ failure simulation — terminate all instances in one AZ
aws fis start-experiment \
  --experiment-template-id <template-id>

# Force RDS/Aurora failover
aws rds failover-db-cluster \
  --db-cluster-identifier my-aurora-cluster

# Lambda throttle injection
# (via FIS aws:lambda:invocation-error action)

# Network latency injection
# (via FIS aws:network:latency action — requires SSM agent)
```

**Observation during experiment:**
- [ ] T+0: When did failure start?
- [ ] T+detection: When did first alarm fire?
- [ ] T+notify: When did on-call receive notification?
- [ ] T+respond: When did runbook execution begin?
- [ ] T+recover: When was service restored to acceptable state?
- [ ] T+validate: When did all health checks go green?

### Post-Game Day Review

| Metric | Target | Actual |
|--------|--------|--------|
| Detection time | < ___ seconds | |
| Notification time | < ___ minutes | |
| Recovery time (RTO) | < ___ minutes | |
| Data loss (RPO) | < ___ seconds | |
| Peak error rate | < ___% | |

**Findings template:**
```
Hypothesis result:  VALIDATED / PARTIALLY VALIDATED / FAILED
What worked:        [...]
What surprised us:  [...]
Gaps in runbook:    [...]
Missing monitoring: [...]
Automation gaps:    [...]

Action items:
  P1 (fix before next game day): [...]
  P2 (fix within 30 days):       [...]
  P3 (fix within 90 days):       [...]
```

---

## Key Resiliency Conversations

Questions that unlock the most valuable architectural discussions:

> **On RTO/RPO:** "If your platform went completely down right now, what's the business impact per minute? Has that number been formally calculated and presented to leadership?"

> **On testing:** "When was the last time you actually failed over your database in production — not a tabletop, but ran the actual procedure? What was the real RTO you measured?"

> **On the app/infra gap:** "Your RDS is Multi-AZ, great. But does your application handle the 60–120 second DNS propagation window during failover? What does your connection pool do when all connections drop simultaneously?"

> **On observability:** "If your service silently stopped processing orders at 3am on a Saturday, how long before someone would know? What would tell them?"

> **On DR:** "Your runbook says 'restore from backup to ap-southeast-2'. Has anyone on the team actually run that procedure end-to-end and timed it?"

> **On single NAT Gateway:** "I see a single NAT Gateway for the VPC. An AZ failure in that AZ kills all outbound internet connectivity for the entire workload — not just the instances in that AZ. Was that a conscious decision?"
