# AWS Service Failure Modes — Reference

Specific timeouts, quota limits, propagation windows, and known edge cases per service.
These are the numbers that matter during incident response and architecture review.

---

## RDS / Aurora

### RDS Multi-AZ Failover
- **Detection time:** 10–30 seconds (CloudWatch health check interval)
- **DNS propagation:** 60–120 seconds end-to-end
- **Total failover window:** 60–120 seconds typically; up to 3 minutes in some cases
- **DNS TTL requirement:** Application DNS cache must respect TTL ≤ 5 seconds
- **Java JDBC issue:** `com.amazonaws.sdk.disableEc2Metadata` or JDBC URL caching can
  hold old IP. Must set `autoReconnect=true` and low `connectTimeout`.
- **Connection storm risk:** All pooled connections drop simultaneously on failover.
  Pool reconnect attempts can overwhelm the new primary. Use exponential backoff with jitter.
- **Read replica promotion:** Manual — NOT automatic in single-region RDS. Only Multi-AZ
  standby promotes automatically. Read replicas require manual intervention.

### Aurora Failover
- **Tier 0 replica promotion:** ~15–30 seconds (writer fails, reader promoted)
- **Tier 1–15 replicas:** Promoted in tier order — lower tier = faster promotion candidate
- **No replica scenario:** Aurora creates new instance — takes 3–5 minutes
- **Aurora Global Database managed failover:** ~1 minute RTO, ~1 second RPO
- **Aurora Global Database unplanned failover:** Manual detach + promote; ~1 minute if automated
- **Cluster endpoint DNS TTL:** 5 seconds — always use cluster endpoint, never instance endpoint
- **Reader endpoint:** Automatically excludes unavailable readers; ~5 second propagation

### RDS Maintenance Windows
- **Minor version upgrade:** Can cause 5–10 minutes of downtime for Multi-AZ (standby upgraded first, then failover)
- **Storage scaling:** Online for gp2/gp3/io1 up to 16 TiB; brief I/O suspension at start (~1 second)
- **Instance type change:** Requires restart — 3–5 minute outage even for Multi-AZ
- **Parameter group changes:** Static parameters require restart; dynamic parameters apply immediately

### Backup and Restore
- **Automated backup window:** Brief I/O suspension at backup initiation (~1 second for Multi-AZ; longer for single-AZ)
- **Restore time:** 20 minutes to several hours depending on database size
- **Point-in-time recovery:** To any second within retention period; excludes last 5 minutes
- **Cross-region snapshot copy:** Async; copy time depends on snapshot size and distance

---

## DynamoDB

### Throttling
- **ProvisionedThroughput:** Throttling begins immediately when consumed capacity exceeds provisioned
- **Burst capacity:** Up to 300 seconds of unused capacity banked — not guaranteed
- **On-demand mode:** Capacity scales automatically but has limits — new tables limited to 4,000 WCU/s initial
- **Hot partition problem:** Data distributed across partitions by partition key — hot keys = throttling
  even when total table capacity is sufficient
- **Adaptive capacity:** Automatically reallocates capacity to hot partitions — takes ~5–30 minutes to activate

### Global Tables
- **Replication lag:** Typically sub-second; can be seconds under high write load
- **Conflict resolution:** Last-writer-wins based on timestamp — no merge, no custom logic
- **Write local, read anywhere:** Always write to local region; reads can be from any region
- **Adding a region:** Requires sufficient read capacity in existing regions to bootstrap replication

### Recovery
- **Point-in-time recovery:** Enabled per table; restores to any second in last 35 days
- **Restore:** Creates new table — does not restore to same table; ~20 minutes for small tables
- **Table deletion:** Immediate and permanent without PITR. `DeletionProtection` prevents this.
- **Export to S3:** For compliance/audit; does not affect table availability

---

## Lambda

### Concurrency and Throttling
- **Account default concurrency limit:** 1,000 per region (soft limit — can be increased)
- **Throttling response:** `TooManyRequestsException` (HTTP 429) — synchronous callers get error immediately
- **Async invocations:** Throttled events retry for up to 6 hours before going to DLQ/EventBridge
- **Reserved concurrency:** Setting to 0 disables function; setting to N limits and reserves N from account pool
- **Provisioned concurrency:** Eliminates cold starts; scales on schedule or via Application Auto Scaling

### Cold Starts
- **Duration:** 100ms–1s+ depending on runtime, package size, VPC configuration
- **VPC cold starts:** Additional ENI attachment time — can add 1–10 seconds (improved with Hyperplane ENI)
- **Mitigation:** Provisioned concurrency; keep package size small; initialise SDK clients outside handler

### Timeouts and Limits
- **Maximum timeout:** 15 minutes
- **Default timeout:** 3 seconds — frequently too low for DB or external API calls
- **Payload limit:** 6 MB synchronous; 256 KB async (SQS/SNS event payload)
- **Ephemeral storage:** 512 MB default; up to 10 GB configurable (`/tmp`)
- **Environment variable size:** 4 KB total

### Failure Modes
- **Async invocation retry:** Retries twice automatically before DLQ; retry intervals ~1 minute and ~2 minutes
- **Missing DLQ:** Failed async events silently discarded after retries
- **SQS trigger:** Lambda scales to consume queue; function errors → message returns to queue after VisibilityTimeout
- **Idempotency:** SQS standard queues deliver at-least-once; Lambda must handle duplicate invocations

---

## SQS

### Visibility Timeout
- **Default:** 30 seconds
- **Maximum:** 12 hours
- **Critical rule:** Must be > maximum processing time. If processing takes longer, message
  becomes visible again and is redelivered while still being processed → duplicate processing
- **Extension API:** `ChangeMessageVisibility` — consumer must call this during long processing
- **Common misconfiguration:** Setting VT to 30s when Lambda timeout is 60s → guaranteed duplicates

### Dead Letter Queues
- **`maxReceiveCount`:** Number of receive attempts before routing to DLQ. Default is none (no DLQ).
- **Recommended:** 3–5 for most workloads; lower for idempotent operations
- **DLQ monitoring:** Always alarm on `ApproximateNumberOfMessagesVisible` > 0 — indicates poison pills
- **DLQ message retention:** Matches source queue retention. Set DLQ retention > source queue retention.
- **Redrive:** Messages can be moved from DLQ back to source queue for reprocessing

### FIFO Queues
- **Throughput limit:** 3,000 messages/second with batching; 300/second without
- **Message group:** Single consumer per group — one slow message blocks entire group
- **Deduplication:** 5-minute deduplication window — duplicate message IDs silently discarded
- **Use FIFO when:** Order matters OR exactly-once processing is required
- **Avoid FIFO when:** High throughput needed or consumer parallelism is critical

### Standard Queues
- **Delivery:** At-least-once — duplicates are possible and normal
- **Ordering:** Best-effort — not guaranteed
- **Throughput:** Nearly unlimited
- **Consumer must be idempotent** — this is a design requirement, not optional

---

## Route53

### Health Checks
- **Check interval:** 30 seconds standard; 10 seconds fast (additional cost)
- **Failure threshold:** 3 consecutive failures before marking unhealthy (default)
- **Detection to DNS propagation:** 30–60 seconds for standard checks after failure detected
- **Total failover time:** 90–120 seconds from failure to DNS update propagating globally
- **String matching:** Health checks can match response body string — use for deep health checks
- **Calculated health checks:** Combine multiple child checks — useful for composite health

### DNS Propagation
- **TTL impact:** Low TTL (≤60s) is essential for fast failover. High TTL = slow client cutover.
- **Recursive resolver caching:** Resolvers may cache beyond TTL; some clients ignore TTL entirely
- **Route53 ARC zonal shift:** Near-instant — shifts traffic away from an AZ at Route53 level;
  effective within ~1 minute regardless of TTL

### Routing Policies
- **Failover routing:** Primary/secondary; automatic cutover based on health check
- **Weighted routing:** Useful for blue/green and canary deployments
- **Latency routing:** Routes to lowest-latency region; health-check-aware
- **Geolocation/Geoproximity:** Route by location; requires default record or risk DNS failure

---

## CloudFront

### Origin Failover
- **Origin group:** Primary + secondary; CloudFront retries secondary on 4xx/5xx or timeout
- **Failover criteria:** Configurable HTTP status codes that trigger failover
- **Cache hit during origin failure:** Cached content served normally — no impact
- **Cache miss during origin failure:** Secondary origin attempted; if both fail, CloudFront returns error
- **Custom error pages:** Configure static error page (S3 hosted) returned on origin failure

### Caching and Availability
- **Default TTL:** 24 hours — stale content served during origin outage for up to 24 hours
- **Stale-while-revalidate:** CloudFront can serve stale content while fetching fresh — reduces origin load
- **Shield Advanced:** Adds DDoS protection + 24/7 AWS DDoS Response Team access

---

## EBS

### I/O Suspension
- **Snapshot initiation:** Brief I/O suspension (~1 second) at snapshot start for non-optimised volumes
- **Optimised volumes:** `gp2`, `gp3`, `io1`, `io2` — minimal or no I/O suspension during snapshot
- **Magnetic (`standard`):** Avoid in production; significant I/O impact during snapshots

### Volume Types — Performance Characteristics
- **gp3:** Baseline 3,000 IOPS, 125 MB/s; independently configurable up to 16,000 IOPS, 1,000 MB/s
- **gp2:** Burstable; performance tied to volume size (3 IOPS/GB). Burst credits can be exhausted.
- **io2 Block Express:** Up to 256,000 IOPS; sub-millisecond latency; Multi-Attach capable
- **Migration:** gp2 → gp3 is online with no downtime; immediate performance and cost improvement

### Multi-Attach
- **Supported:** `io1`, `io2` only; up to 16 instances per volume; same AZ only
- **Use case:** Clustered workloads (RAC, Pacemaker); application must manage concurrent writes
- **Not supported:** gp2, gp3, st1, sc1

---

## ALB / NLB

### Health Checks
- **ALB:** HTTP/HTTPS health checks; checks target group members
- **Unhealthy threshold:** Default 2 consecutive failures before deregistering
- **Healthy threshold:** Default 5 consecutive successes before registering
- **Deregistration delay:** Default 300 seconds — in-flight connections drain before target removed
  (reduce to 30–60 seconds for fast rolling deployments)
- **Health check path:** Use a deep health check endpoint that validates DB connectivity,
  not just `200 OK` from web server

### Connection Limits
- **ALB:** Scales automatically; handles millions of connections
- **NLB:** Preserves source IP; TCP-level; lower latency than ALB; no HTTP-aware features
- **Idle timeout:** ALB default 60 seconds; NLB default 350 seconds; must match application expectations

---

## ElastiCache (Redis)

### Failover
- **Primary failure detection:** ~10 seconds
- **Replica promotion:** Additional 10–30 seconds
- **Total failover window:** ~20–60 seconds
- **DNS TTL:** 5 seconds for cluster endpoint — application must respect this
- **Multi-AZ required:** `AutomaticFailoverEnabled: true` requires ≥2 nodes across ≥2 AZs

### Common Failure Modes
- **Connection storm on failover:** All application instances reconnect simultaneously
  → rate limit connection pool size and stagger reconnection attempts
- **Memory eviction:** `maxmemory-policy` determines eviction behaviour; `allkeys-lru`
  is safest for cache workloads; `noeviction` causes write failures when memory full
- **Cluster mode:** Sharding across multiple nodes; keyspace must align with hash slots;
  cross-slot operations (multi-key commands) not supported

---

## EventBridge

### Delivery and Retry
- **At-least-once delivery:** Duplicates possible; consumers must be idempotent
- **Retry policy:** Up to 185 times over 24 hours (configurable: 0–24 hours, 0–185 retries)
- **Event age:** Events older than `maximumEventAge` discarded without delivery
- **Dead letter queue:** Events that exhaust retries or exceed max age → DLQ (must be configured)
- **Missing DLQ:** Failed events silently discarded after retry exhaustion

### Limits
- **PutEvents:** 10,000 events/second/region default; 256 KB max per request; 64 KB max per event
- **Rule limit:** 300 rules per event bus default
- **Target limit:** 5 targets per rule

---

## S3

### Consistency
- **Strong read-after-write consistency:** Since December 2020 — no eventual consistency lag
- **Versioning:** Must be enabled before first write to protect data; cannot protect objects
  written before versioning was enabled
- **Replication lag:** S3 Replication Time Control (RTC) guarantees 99.99% of objects
  replicated within 15 minutes; standard replication is best-effort (typically minutes)

### Throttling
- **Request rate:** 3,500 PUT/COPY/POST/DELETE and 5,500 GET/HEAD per second per prefix
- **Prefix partitioning:** Distribute objects across multiple prefixes to scale beyond per-prefix limits
- **503 SlowDown:** Returned when request rate exceeds limits; must retry with exponential backoff

### Durability and Availability
- **Durability:** 11 nines (99.999999999%) — S3 is not a backup risk; it's an accidental deletion risk
- **Standard availability:** 99.99% — four nines
- **S3 One Zone-IA:** Single AZ; not resilient to AZ failure; appropriate only for reproducible data
