# AWS Well-Architected Reliability Pillar — Reference

Mapped to resiliency best practices for TAM reviews and formal WARs.

---

## Design Principles

1. **Automatically recover from failure** — monitor KPIs and trigger automation on threshold breach
2. **Test recovery procedures** — use automation to simulate failures and validate recovery
3. **Scale horizontally** — replace one large resource with multiple small ones; reduce single-component impact
4. **Stop guessing capacity** — use Auto Scaling; eliminate over/under provisioning
5. **Manage change through automation** — use IaC; avoid manual configuration drift

---

## Reliability Pillar Questions

### REL 1: How do you manage service quotas and constraints?
**Best practices:**
- Know your account-level and service-level quotas for every service in use
- Request quota increases proactively — don't discover limits during a traffic spike
- Implement retry with backoff for throttled requests
- Use CloudWatch to alarm when approaching quota limits (e.g. Lambda concurrency at 80% of limit)

**TAM questions to ask:**
- "Have you mapped your peak traffic projections against your Lambda concurrency limits?"
- "Do you have quota increase requests in flight for your growth projections?"

---

### REL 2: How do you plan your network topology?
**Best practices:**
- Use multiple AZs in every subnet configuration
- Size VPC CIDR blocks for future growth — cannot be reduced after creation
- Use VPC endpoints for AWS services to reduce NAT Gateway dependency
- Avoid overlapping CIDR blocks with on-premises networks for hybrid connectivity
- Use Transit Gateway for hub-and-spoke multi-VPC topologies at scale

**Common gaps:**
- Single NAT Gateway serving all AZs (single point of failure for all outbound traffic)
- VPC CIDR too small — no room to add subnets or AZs
- No VPC Flow Logs — no forensic capability after network incidents

---

### REL 3: How do you design your workload service architecture?
**Best practices:**
- Implement loose coupling — queues and event buses between services
- Design for graceful degradation — core functionality works when non-core services fail
- Use service discovery rather than hardcoded endpoints
- Implement bulkhead patterns — isolate failures so one component doesn't cascade

**Common gaps:**
- Synchronous chains — ServiceA → ServiceB → ServiceC where any failure cascades
- Missing circuit breakers between services
- Shared databases between services — one service's load degrades another's performance

---

### REL 4: How do you anticipate, accommodate, and manage disruption?
**Best practices:**
- Implement throttling and rate limiting at API boundaries
- Use SQS/EventBridge to absorb traffic bursts
- Implement exponential backoff with jitter on all retry logic
- Test with load that exceeds expected peak — know where the breaking point is

**Key failure modes:**
- Thundering herd: Multiple clients retry simultaneously after a failure → amplifies load on recovering service
- Retry storm: Aggressive retry logic creates more load than original request rate
- Fix: Exponential backoff with full jitter (`sleep = random_between(0, min(cap, base * 2^attempt))`)

---

### REL 5: How do you monitor workload resources?
**Best practices:**
- Define SLOs per workload tier before building the monitoring stack
- Monitor at every layer: infrastructure metrics, application metrics, business metrics
- Use structured logging with consistent correlation IDs across services
- Set alarms on both leading indicators (queue depth rising) and lagging indicators (error rate)
- Use composite alarms to capture multi-signal failure modes
- `TreatMissingData: breaching` on all critical alarms

**TAM questions to ask:**
- "What's your MTTD (mean time to detect) for a full service outage?"
- "Do you have alarms on business metrics — bet placement rate, transaction volume — not just infrastructure?"
- "Would you know if your DLQ was filling up at 3am?"

---

### REL 6: How do you design your workload to adapt to changes in demand?
**Best practices:**
- Use Auto Scaling Groups with predictive and dynamic scaling
- Use Lambda for inherently auto-scaling compute
- Configure DynamoDB on-demand or provisioned with auto-scaling
- Load test to validate scaling policies actually trigger and scale fast enough
- Pre-warm resources before known demand events (sports events, race days)

**Common gaps:**
- ASG scaling policy with `CooldownPeriod` too long — instances scale out too slowly under sudden load
- DynamoDB provisioned mode without auto-scaling — hits throttling cliff under burst
- Lambda not considering account concurrency limits under fan-out patterns

---

### REL 7: How do you implement change?
**Best practices:**
- Use blue/green or canary deployments for all production changes
- Automate rollback triggers — alarm state triggers automatic rollback
- Test changes in staging with production-like traffic and data volumes
- Use feature flags to decouple deployment from release
- Maintain IaC in version control with PR review for all infrastructure changes

**Common gaps:**
- Manual deployments with no automated rollback
- Staging environment that doesn't match production scale
- Database schema changes deployed without backward compatibility period

---

### REL 8: How do you back up data?
**Best practices:**
- Define backup policy per data tier: RPO drives backup frequency
- Use AWS Backup for centralised policy management across services
- Test restores regularly — backup is useless if restore doesn't work
- Store backups in a separate account or region
- Enable versioning on S3; PITR on RDS and DynamoDB
- Encrypt all backups with CMK

**TAM questions to ask:**
- "When did you last test a restore from backup?"
- "If your primary region became unavailable for 48 hours, where would you restore to?"
- "Are your backups in the same account as production?" (If yes — account compromise = lose backups)

---

### REL 9: How do you use fault isolation to protect your workload?
**Best practices:**
- Deploy across ≥ 3 AZs for production workloads
- Use cell-based architecture for highest-tier workloads
- Implement AZ-aware routing — Route53 ARC zonal shift for rapid AZ evacuation
- Isolate data plane from control plane — data plane must work even if AWS console/APIs are slow

**Cell-based architecture:**
- Divide customers into cells (shards); failures contained to one cell
- Each cell is fully independent — shared dependencies = shared failure radius
- Recommended for: wagering platforms, payment systems, anything at scale where partial failure is better than full failure

---

### REL 10: How do you design your workload to withstand component failures?
**Best practices:**
- Implement health checks at every layer: ALB, ASG, ECS, RDS
- Use exponential backoff with jitter for all retry logic
- Implement circuit breakers for external dependencies
- Design idempotent operations — safe to retry without side effects
- Use DLQs on all async event processing

---

### REL 11: How do you test reliability?
**Best practices:**
- Game days: simulate failure scenarios in production or production-like environment
- Chaos engineering: inject faults systematically (AWS Fault Injection Simulator — FIS)
- DR drills: actually fail over to DR region, measure RTO, identify gaps
- Load testing: validate scaling and breaking points before peak events

**AWS Fault Injection Simulator (FIS) experiments to recommend:**
- `aws:ec2:terminate-instances` — AZ failure simulation
- `aws:rds:failover-db-cluster` — Aurora failover
- `aws:ecs:stop-task` — ECS task failure
- `aws:sqs:send-message` — Inject poison pill messages
- `aws:fis:inject-api-internal-error` — Simulate AWS API failures

**TAM questions to ask:**
- "Have you run a game day in the last 6 months?"
- "Do you have documented, tested runbooks for your top 5 failure scenarios?"
- "Has your on-call team ever actually executed a DR failover, or only tabletop exercised it?"

---

### REL 12: How do you plan for disaster recovery?
**Best practices:**
- Define RTO and RPO per workload tier — get business sign-off
- Choose DR pattern based on RTO/RPO and cost constraints
- Automate DR failover where possible — manual steps = human error under pressure
- Document runbooks with step-by-step instructions including expected timing
- Test DR quarterly minimum; Tier 1 workloads should test monthly

**RTO/RPO calculation guidance:**
- **RTO** = Time from failure declaration to service restored
  - Detection time + escalation time + decision time + execution time + validation time
  - Each manual step adds 5–15 minutes of uncertainty
- **RPO** = Maximum acceptable data loss measured in time
  - Drives backup frequency, replication lag requirements, transaction log retention

**Operational Readiness Review (ORR) checklist:**
- [ ] RTO and RPO defined and approved by business stakeholders
- [ ] DR runbook written and reviewed
- [ ] DR runbook executed in staging with timing recorded
- [ ] On-call rotation knows where runbook lives and has access during an incident
- [ ] Monitoring covers DR region (not just primary)
- [ ] Third-party dependencies have their own DR capability assessed
- [ ] Communication plan defined (who notifies who, what channels)
- [ ] Post-incident review process defined
