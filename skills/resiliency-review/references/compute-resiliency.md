# Compute Resiliency — EC2, ECS, Lambda, Auto Scaling

Load this file when the user's code contains EC2, ECS, EKS, Lambda, ASG, Fargate, or Auto Scaling resources.

---

## EC2 / Auto Scaling Groups

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| ASG configured | `AutoScalingGroup` resource present | Standalone EC2 with no ASG = unrecoverable single instance |
| Multi-AZ | `VPCZoneIdentifier` references subnets in ≥2 AZs | Single AZ in ASG = full outage on AZ failure |
| Health check type | `HealthCheckType: ELB` | Default `EC2` only detects host failure, not app-level failure |
| Health check grace period | Set to ≥ startup time of application | Too short = ASG terminates healthy instance mid-boot |
| Min capacity | `MinSize` ≥ 2 for production | `MinSize: 1` = brief outage during instance replacement |
| Termination protection | Enabled on critical standalone instances | Missing = accidental delete risk |
| IMDSv2 | `HttpTokens: required` in LaunchTemplate | IMDSv1 = credential theft via SSRF |
| Detailed monitoring | Enabled on instances | 5-minute default metric resolution delays alarm detection |

### Application Code Checks
- Connection pool max size — will it exhaust during a failover reconnection storm?
- Hardcoded AZ-specific endpoints or private IPs — must use DNS names
- Retry logic on downstream calls — missing retry = single transient failure = user error
- Graceful shutdown handling — `SIGTERM` handler must drain in-flight requests before exit

### Key Failure Modes
**AZ failure with single-AZ ASG** → full service outage, manual intervention required.
**ASG `HealthCheckType: EC2`** → ALB health check fails (app is broken), but ASG sees
instance as healthy (host is up) → broken instance stays in rotation indefinitely.
**Rolling deployment killing all tasks simultaneously** → `MaximumPercent: 200` with
`MinimumHealthyPercent: 100` required to avoid zero-capacity window.

---

## ECS / Fargate

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| Desired count | `desiredCount` ≥ 2 | `desiredCount: 1` = task failure = full service outage |
| Multi-AZ placement | `placementStrategies` with `spread` across `attribute:ecs.availability-zone` | Tasks may stack in one AZ |
| Deployment circuit breaker | `deploymentCircuitBreaker: {enable: true, rollback: true}` | Broken deployment keeps retrying, taking down service |
| Min healthy percent | `minimumHealthyPercent: 100` | Lower value = reduced capacity during deployments |
| Max percent | `maximumPercent: 200` | Ensures new tasks start before old ones stop |
| Health check grace period | `healthCheckGracePeriodSeconds` ≥ application startup time | Premature health check failure = task restart loop |
| Task CPU/memory | Sized for peak + 20% headroom | OOM kills or CPU throttle under load |
| Capacity provider | Fargate + Fargate Spot for non-critical; pure Fargate for critical | Spot interruptions without fallback = reduced capacity |

### Application Code Checks
- Environment variables for config — no hardcoded region, endpoint, or account IDs
- Graceful shutdown: ECS sends `SIGTERM` then waits `stopTimeout` (default 30s) before `SIGKILL`
- `stopTimeout` in task definition must be > time needed to drain in-flight requests

### Key Failure Modes
**Single task failure** → if `desiredCount: 1`, full outage until replacement task passes health check (~30–60 seconds).
**Deployment with `minimumHealthyPercent: 0`** → all running tasks stopped before new ones start → downtime window.
**Missing circuit breaker** → a broken task definition causes ECS to repeatedly start and stop tasks in a crash loop, consuming capacity and generating noise.

---

## Lambda

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| Timeout | Set explicitly, > downstream call timeout | Default 3s: silently times out on any slow DB or API call |
| DLQ / destination | `deadLetterQueue` or `onFailure` destination set | Async failures silently discarded after 2 retries |
| Reserved concurrency | Explicitly set (not `null`) | `null` = competes with all other functions for account pool |
| Memory size | Sized to measured peak usage + 20% | Under-sized = OOM kills; over-sized = wasted cost |
| VPC config | Private subnets only; security group allows only required egress | Public subnet Lambda = security exposure |
| Layers | Layer ARNs pinned to version | `$LATEST` on layers = uncontrolled updates |
| Architectures | Consistent (`x86_64` or `arm64`) | Mixed arch = different cold start profiles across versions |
| Tracing | `tracing: Active` | `PassThrough` = no X-Ray visibility into invocations |

### Application Code Checks
- **SDK clients outside handler** — initialising inside handler = cold start on every invocation, no connection reuse
- **Missing retry on transient errors** — Lambda SDK has retry built in for AWS API calls, but not for RDS/external HTTP
- **Timeout value vs. downstream** — Lambda timeout must be > downstream call timeout, or Lambda kills the call before the downstream times out gracefully
- **`/tmp` usage** — 512 MB default; contents persist across warm invocations — check for stale file assumptions
- **Hardcoded region** — must read from `AWS_REGION` environment variable

### Key Failure Modes
**Account concurrency limit (default 1,000/region)** → Lambda throttles all functions when total
concurrent executions hit limit. One runaway function starves all others.
**Async invocation with no DLQ** → failed events retried twice (at ~1 min and ~2 min intervals)
then silently discarded. No trace, no alert.
**VPC Lambda cold start** → ENI attachment adds 1–10 seconds (improved with Hyperplane ENI in 2019,
but VPC config still adds latency vs non-VPC on cold start).
**SQS trigger + Lambda timeout misconfiguration** → if Lambda timeout > SQS `VisibilityTimeout`,
message becomes visible again while Lambda is still processing → duplicate invocations.

### Lambda Concurrency Reference
```
Account limit (default):         1,000 concurrent executions / region
Burst limit:                     500–3,000 / minute (region-dependent)
Reserved concurrency:            Guarantees AND limits that function to N
Provisioned concurrency:         Pre-warmed; eliminates cold starts; billed always
Unreserved pool:                 Account limit minus all reserved concurrency
```

---

## CDK Patterns — Compute Resiliency

```typescript
// ✅ ECS Service with circuit breaker and multi-AZ
const service = new ecs.FargateService(this, 'PaymentsService', {
  cluster,
  taskDefinition,
  desiredCount: 2,
  minHealthyPercent: 100,
  maxHealthyPercent: 200,
  circuitBreaker: { rollback: true },
  vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
  // Tasks spread across AZs by default with Fargate
});

// ✅ Lambda with DLQ and explicit timeout
const fn = new lambda.Function(this, 'PaymentsProcessor', {
  runtime: lambda.Runtime.NODEJS_20_X,
  timeout: cdk.Duration.seconds(29),   // under ALB 30s limit
  memorySize: 512,
  reservedConcurrentExecutions: 100,   // protect account pool
  tracing: lambda.Tracing.ACTIVE,
  deadLetterQueue: new sqs.Queue(this, 'PaymentsDLQ', {
    retentionPeriod: cdk.Duration.days(14),
  }),
});
```

```typescript
// ❌ Common failure patterns
const service = new ecs.FargateService(this, 'Service', {
  desiredCount: 1,            // CRITICAL: single task = single point of failure
  minHealthyPercent: 0,       // CRITICAL: zero capacity during deployments
  // no circuitBreaker        // HIGH: broken deploys loop indefinitely
});

const fn = new lambda.Function(this, 'Fn', {
  timeout: cdk.Duration.seconds(3),   // HIGH: default 3s too low for DB calls
  // no deadLetterQueue               // HIGH: async failures silently lost
  // no reservedConcurrentExecutions  // MEDIUM: can starve account pool
});
```
