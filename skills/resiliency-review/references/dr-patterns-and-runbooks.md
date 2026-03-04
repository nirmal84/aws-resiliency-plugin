# DR Patterns, Runbooks & Game Day — Reference

Templates and guidance for preparing DR assessments, ORRs, and game days.

---

## DR Pattern Selection

### Pattern 1: Backup & Restore
**RTO:** Hours | **RPO:** Hours | **Cost multiplier:** 1x

**When to use:** Dev/test environments; non-critical workloads; data archival

**IaC pattern:**
- RDS: Automated backups + manual snapshots to separate account/region
- S3: Cross-region replication or periodic export
- EC2: AMI snapshots via AWS Backup
- No standby infrastructure — rebuild on demand

**Recovery procedure:**
1. Declare DR event
2. Launch infrastructure from IaC in DR region (~30 min for simple stacks)
3. Restore latest database snapshot (~20 min for small DBs; hours for large)
4. Update DNS to point to DR region (~5 min)
5. Validate and open traffic

**Total estimated RTO:** 1–4 hours depending on data volume

---

### Pattern 2: Pilot Light
**RTO:** 10–30 min | **RPO:** Minutes | **Cost multiplier:** 1.5–2x

**When to use:** Core services that can tolerate brief outage; moderate budget

**IaC pattern:**
- Core data tier always running: RDS Multi-AZ in DR region (or Aurora Global DB secondary)
- Compute at minimum capacity: ASG with `minSize: 0` or `desiredCapacity: 1`
- Route53 failover routing with health checks configured but pointing to primary
- On failover: scale up compute, update DNS

**Recovery procedure:**
1. Declare DR event (or automated health check triggers)
2. Scale up ASG in DR region (5–10 min for instances to be healthy)
3. Verify DB replication is current and promote if needed
4. Cut Route53 DNS to DR region (propagation: 60–120 seconds)
5. Validate application health

**Total estimated RTO:** 15–30 minutes

---

### Pattern 3: Warm Standby
**RTO:** Minutes | **RPO:** Seconds | **Cost multiplier:** 2–3x

**When to use:** Business-critical workloads; moderate-to-high budget; wagering/payments

**IaC pattern:**
- Full stack running at reduced capacity in DR region
- Aurora Global Database: RPO ~1 second
- DynamoDB Global Tables: active-active replication
- ALB + ASG at minimum viable capacity in DR region
- Route53 weighted routing (primary: 100, secondary: 0) or failover routing
- On failover: scale up DR region, cut traffic

**Recovery procedure:**
1. Automated: health check failure triggers Route53 failover (90–120 seconds)
   OR
   Manual: Update Route53 weights (primary: 0, secondary: 100)
2. Scale up ASG in DR region to full capacity (5–10 min)
3. Validate Aurora Global DB promoted or DynamoDB serving local writes
4. Monitor error rates and latency — validate recovery

**Total estimated RTO:** 5–15 minutes

---

### Pattern 4: Active-Active (Multi-Region)
**RTO:** Near-zero | **RPO:** Near-zero | **Cost multiplier:** 2–4x

**When to use:** Mission-critical revenue-generating workloads; zero tolerance for downtime

**IaC pattern:**
- Full stack in ≥ 2 regions, both serving live traffic
- Global Accelerator or Route53 latency routing distributing traffic
- DynamoDB Global Tables: last-writer-wins, application handles conflicts
- Aurora Global Database write forwarding (or write local to each region's cluster)
- CloudFront multi-region origin group
- Route53 ARC routing controls for surgical traffic management

**Key design constraints:**
- Data conflict resolution strategy must be defined upfront
- Session state must be shared (ElastiCache Global Datastore) or stateless
- All services must be deployed identically in all regions
- Deployment pipeline must be multi-region aware

**On regional failure:**
- Global Accelerator detects failure in ~30 seconds; redirects traffic automatically
- Route53 health check failover: 90–120 seconds
- No manual intervention required if health checks are correctly configured

---

## Game Day Exercise Template

### Pre-Game Day (2 weeks before)

**Define the scenario:**
- Which failure are we simulating? (AZ failure / region failure / service degradation / database failover)
- Which workload/service is in scope?
- What is the expected outcome? (RTO target, expected error rate, recovery procedure)

**Prepare:**
- [ ] Runbook reviewed and accessible to all participants
- [ ] Monitoring dashboards prepared and bookmarked
- [ ] Communication channel established (Slack/Teams war room)
- [ ] Customer stakeholders notified (if production game day)
- [ ] Rollback plan documented if game day causes unintended impact
- [ ] AWS account team notified (if applicable)

**Define success criteria:**
- RTO target: _____ minutes
- RPO target: _____ minutes/seconds of data loss acceptable
- Error rate during recovery: < _____% acceptable
- All critical alerts fired: Yes/No

---

### Game Day Execution

**T-0: Inject failure**
Using AWS Fault Injection Simulator (FIS) or manual action:

```bash
# Example: Terminate all EC2 instances in one AZ
aws ec2 terminate-instances \
  --instance-ids $(aws ec2 describe-instances \
    --filters "Name=availability-zone,Values=ap-southeast-2a" \
    --query "Reservations[].Instances[].InstanceId" \
    --output text)

# Example: Force RDS failover
aws rds failover-db-cluster \
  --db-cluster-identifier my-aurora-cluster

# Example: FIS experiment
aws fis start-experiment \
  --experiment-template-id EXT1234567890abcde
```

**T+0 to T+RTO target: Observe and document**
- [ ] When did the first alarm fire? (Detection time)
- [ ] When did on-call receive the notification? (Notification time)
- [ ] When did the runbook execution begin? (Response time)
- [ ] When was the service restored to acceptable state? (Recovery time)
- [ ] What was the actual error rate peak?
- [ ] What customer impact occurred?

**Post-recovery validation checklist:**
- [ ] All health checks green
- [ ] Error rate back to baseline
- [ ] Latency back to baseline
- [ ] Database replication caught up (for multi-region scenarios)
- [ ] DLQ empty (no messages stuck during recovery)
- [ ] No data loss detected

---

### Post-Game Day Review

**Metrics to record:**
| Metric | Target | Actual |
|---|---|---|
| Detection time | < ___ seconds | |
| Alert notification time | < ___ minutes | |
| Recovery time (RTO) | < ___ minutes | |
| Data loss (RPO) | < ___ seconds | |
| Peak error rate | < ___% | |

**Findings to capture:**
- What worked as expected?
- What surprised us?
- What gaps were found in the runbook?
- What monitoring was missing?
- What automation would have reduced RTO?

**Action items (prioritised):**
1. Critical (fix before next game day): _____
2. High (fix within 30 days): _____
3. Medium (fix within 90 days): _____

---

## Operational Readiness Review (ORR) Checklist

Use this checklist when a customer is preparing to launch a new workload or before a
major traffic event (sports finals, racing carnival, product launch).

### Architecture
- [ ] Multi-AZ deployment for all stateful services
- [ ] Auto Scaling configured and tested for expected peak + 50% headroom
- [ ] Load tested to 150% of expected peak traffic
- [ ] Single points of failure identified and remediated or accepted with documented risk
- [ ] Blast radius of each component failure understood and documented

### Data
- [ ] Backup policy defined with RTO/RPO targets
- [ ] Backup restore tested in last 30 days (with timing recorded)
- [ ] PITR enabled on RDS and DynamoDB
- [ ] Database failover tested (for RDS Multi-AZ and Aurora)
- [ ] Replication lag monitored and alarmed

### Networking
- [ ] Health checks configured on ALB targets (deep health check, not just root path)
- [ ] Route53 health checks configured and tested
- [ ] NAT Gateway per AZ (not single shared NAT)
- [ ] VPC endpoints for S3/DynamoDB where used at scale

### Observability
- [ ] Alarms on all Tier 1 metrics (CPU, memory, error rate, latency p99, queue depth)
- [ ] Alarms send to on-call system (PagerDuty/OpsGenie)
- [ ] Business metric alarms configured (transaction rate, bet placement rate)
- [ ] `TreatMissingData: breaching` on all critical alarms
- [ ] Runbook linked from every alarm description
- [ ] Dashboards prepared for incident response

### Operations
- [ ] On-call rotation established with primary and secondary coverage
- [ ] Runbooks written for top 5 failure scenarios
- [ ] Runbooks tested in staging (not just tabletop reviewed)
- [ ] Communication plan documented (internal escalation + customer communication)
- [ ] Post-incident review process defined
- [ ] Change freeze window defined for event period (if applicable)

### Security (resiliency-relevant)
- [ ] Account separation: production in dedicated AWS account
- [ ] Backups in separate account from production
- [ ] IAM roles follow least privilege — blast radius of credential compromise limited
- [ ] CloudTrail enabled for audit and forensics

---

## Key Resiliency Questions — Conversations That Matter

Questions that unlock the most valuable resiliency discussions with engineering teams:

**On RTO/RPO:**
> "If your platform went completely down right now, what would be the business impact
> per minute? Has that number ever been formally calculated and presented to leadership?"

**On testing:**
> "When was the last time you actually failed over your database in production — not a
> tabletop exercise, but actually ran the procedure? What was the real RTO you measured?"

**On the gap between IaC and application behaviour:**
> "Your RDS is Multi-AZ, which is great. But does your application handle the 60-120
> second DNS propagation window during failover? Have you tested what your connection
> pool does when all connections drop simultaneously?"

**On observability:**
> "If your wagering platform silently stopped accepting bets at 2am on a Saturday night,
> how long before someone would know? What would tell them?"

**On DR regions:**
> "You have a DR runbook that says 'restore from backup to ap-southeast-2'. Has anyone
> on the team actually run that procedure end-to-end? Do you know how long it takes?"

**On single points of failure:**
> "I see you have a single NAT Gateway for the VPC. That means an AZ failure in that
> AZ takes out all outbound internet connectivity for the entire workload — not just
> the instances in that AZ. Was that a conscious decision?"
