# Observability Resiliency — CloudWatch, X-Ray, Health Dashboard

Load this file when the user's code contains CloudWatch alarms, log groups, dashboards, X-Ray tracing, or monitoring-related resources. Also load when the user asks how they would detect a failure.

---

## Core Principle: Observability Is a Resiliency Requirement

> **If you cannot detect a failure within your detection RTO, your actual RTO is unbounded.**
> Missing observability is a resiliency finding, not just an operational preference.

The key question for every service: **"If this failed silently at 3am on a Saturday, how long before anyone would know?"** If the answer is "a user would tell us" — that is a CRITICAL finding.

---

## CloudWatch Alarms

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| Alarm on all Tier 1 metrics | Alarms on CPU, error rate, latency p99, queue depth, DLQ | Missing = silent failures; no on-call paging |
| Alarm action | `alarmAction` pointing to SNS → PagerDuty/OpsGenie | Alarm with no action is decorative — it won't wake anyone up |
| `TreatMissingData` | `TreatMissingData.BREACHING` on critical alarms | Default `MISSING` = alarm goes OK when data stops flowing — metric pipeline failure looks like healthy service |
| Composite alarms | Used to capture multi-signal failures | Individual alarms = alert fatigue; composite = fewer, higher-signal alerts |
| Alarm evaluation period | ≤ 5 minutes for critical alarms | 15-minute evaluation = failure undetected for up to 15 minutes |
| Alarm description | Contains runbook link | On-call engineer needs to know what to do at 3am — link the runbook in the alarm description |

### Critical Alarms by Service Type

**Lambda:**
```typescript
// Error rate alarm
fn.metricErrors({ period: cdk.Duration.minutes(1) })
  .createAlarm(this, 'LambdaErrors', {
    threshold: 5,
    evaluationPeriods: 3,
    treatMissingData: cloudwatch.TreatMissingData.BREACHING,
    alarmDescription: 'Lambda error rate high. Runbook: https://wiki/lambda-errors',
  });

// Throttle alarm — throttles are always actionable
fn.metricThrottles({ period: cdk.Duration.minutes(1) })
  .createAlarm(this, 'LambdaThrottles', {
    threshold: 1,
    evaluationPeriods: 2,
    treatMissingData: cloudwatch.TreatMissingData.NOT_BREACHING,
  });
```

**ALB:**
- `HTTPCode_ELB_5XX_Count` > 10 in 1 minute → service failing at load balancer level
- `HTTPCode_Target_5XX_Count` > 10 in 1 minute → application returning 5xx
- `TargetResponseTime` p99 > SLO threshold → latency degradation
- `UnHealthyHostCount` > 0 → targets failing health checks

**RDS:**
- `CPUUtilization` > 80% for 5 minutes → query tuning or scaling needed
- `DatabaseConnections` > 80% of `max_connections` → connection exhaustion risk
- `FreeStorageSpace` < 20% → storage growth risk (database stops on 0)
- `ReadLatency` / `WriteLatency` p99 > threshold → performance degradation
- `ReplicaLag` > 30 seconds → stale reads; lagging replica won't be promoted cleanly

**SQS:**
- `ApproximateAgeOfOldestMessage` > processing SLA → consumer falling behind
- DLQ `ApproximateNumberOfMessagesVisible` > 0 → poison pills or consumer errors

**EC2/ECS:**
- `StatusCheckFailed` > 0 → instance-level hardware or OS failure
- ECS `RunningTaskCount` < `DesiredCount` → tasks failing or not starting

### `TreatMissingData` — The Hidden Alarm Gap

```
Scenario: Application crashes and stops emitting metrics.
          CloudWatch alarm on error count with TreatMissingData: NOT_BREACHING

Result: Alarm sees no data → treats as 0 errors → alarm goes to OK state
        On-call sees green dashboard → believes service is healthy
        Service is actually completely down

Fix: TreatMissingData: BREACHING for all critical alarms
     Rationale: if you stop hearing from the service, assume it's broken
```

---

## CloudWatch Logs

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| Log group retention | `retention` set explicitly | Default = never expire → unbounded log storage cost |
| Structured logging | JSON-formatted logs with consistent fields | Unstructured logs = slow CloudWatch Insights queries; difficult correlation |
| Metric filters | Created for critical error patterns | Without metric filters, error patterns require manual log inspection |
| Log group encryption | `encryptionKey` set for sensitive logs | Default = AWS-managed key; fine for most; CMK required for compliance |
| Export to S3 | Long-term retention strategy | CloudWatch Logs retention max is configurable but costly at scale; S3 is cheaper for archival |

### Application Code Checks — Structured Logging Pattern

```python
# ✅ Structured logging with correlation IDs
import json, logging, os

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def handler(event, context):
    log = {
        'service': 'payments-processor',
        'requestId': context.aws_request_id,
        'traceId': os.environ.get('_X_AMZN_TRACE_ID'),
        'environment': os.environ.get('ENVIRONMENT', 'unknown'),
    }

    try:
        result = process_payment(event)
        logger.info(json.dumps({**log, 'event': 'payment.processed',
                                'orderId': event['orderId'], 'amount': result['amount']}))
        return result
    except PaymentDeclinedException as e:
        logger.warning(json.dumps({**log, 'event': 'payment.declined',
                                   'reason': str(e)}))
        raise
    except Exception as e:
        logger.error(json.dumps({**log, 'event': 'payment.error',
                                 'error': str(e), 'errorType': type(e).__name__}))
        raise
```

**Required fields in every log entry:**
- `service` — which service emitted this
- `requestId` or `correlationId` — trace across service boundaries
- `environment` — prod/staging/dev (prevents log cross-contamination)
- `level` — explicit severity
- `event` — machine-parseable event name (enables metric filters)

---

## X-Ray Tracing

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| Lambda tracing | `tracing: Tracing.ACTIVE` | `PASS_THROUGH` = no X-Ray visibility into Lambda invocations |
| API Gateway tracing | `tracingEnabled: true` | Missing = no trace from API GW to Lambda |
| ECS X-Ray sidecar | X-Ray daemon as sidecar container in task definition | Missing = traces not collected from ECS services |
| Sampling rules | Custom sampling rules for high-volume services | Default 5% sampling may miss rare errors; 100% sampling is expensive at scale |
| Service map | Groups configured for logical service boundaries | Without groups, X-Ray service map shows all services as a flat list |

### X-Ray Insights
Enable X-Ray Insights for automatic anomaly detection on service maps — it surfaces latency outliers and error rate spikes without manual threshold configuration. Most useful for microservices with complex dependency graphs.

---

## AWS Health Dashboard

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| Health events via EventBridge | `EventBridge` rule on `aws.health` events | Missing = no automated notification of AWS service disruptions affecting your account |
| Personal Health Dashboard | Checked during incidents | Forgetting to check PHD during incidents wastes hours investigating application code when the issue is an AWS service disruption |
| Account-level events | Both account-specific and public health events subscribed | Public events = regional disruptions; account-specific = issues with your resources directly |

```typescript
// ✅ AWS Health events → SNS → on-call
new events.Rule(this, 'AWSHealthEvents', {
  eventPattern: {
    source: ['aws.health'],
    detailType: ['AWS Health Event'],
    detail: {
      service: ['EC2', 'RDS', 'LAMBDA', 'SQS', 'ECS'],
      eventTypeCategory: ['issue', 'scheduledChange'],
    },
  },
  targets: [new targets.SnsTopic(oncallTopic)],
});
```

---

## Business Metrics — The Layer Nobody Builds Until After an Incident

Infrastructure metrics tell you a service is slow. Business metrics tell you the business is losing money.

**Always recommend business-level alarms in addition to infrastructure alarms:**

| Industry | Infrastructure alarm | Business metric alarm |
|----------|---------------------|----------------------|
| E-commerce | Lambda error rate > 1% | Order creation rate drops > 20% vs. 5-min rolling average |
| Wagering / iGaming | ALB 5xx > 10/min | Bet placement rate drops > 30% vs. baseline |
| Payments | RDS latency p99 > 200ms | Successful payment rate drops below 99.5% |
| SaaS | ECS task count < desired | Active session count drops > 25% |

```python
# ✅ Custom business metric — publish from application code
import boto3
cloudwatch = boto3.client('cloudwatch')

def record_payment_outcome(success: bool, payment_method: str):
    cloudwatch.put_metric_data(
        Namespace='MyApp/Payments',
        MetricData=[{
            'MetricName': 'PaymentAttempt',
            'Dimensions': [
                {'Name': 'Method', 'Value': payment_method},
                {'Name': 'Outcome', 'Value': 'success' if success else 'failure'},
            ],
            'Value': 1,
            'Unit': 'Count',
        }]
    )
```

---

## Observability Maturity Model

Use this to frame the conversation with teams at different levels:

| Level | What they have | What's missing | Next step |
|-------|---------------|----------------|-----------|
| **0 — Dark** | No alarms or logs | Everything | Start with ALB 5xx + DLQ alarms |
| **1 — Basic** | CPU/memory alarms, some logs | Error rate, latency, DLQ alarms | Add per-service error rate and latency p99 alarms |
| **2 — Informed** | Error + latency alarms, structured logs | Business metrics, TreatMissingData fix | Add business metric alarms, fix TreatMissingData |
| **3 — Proactive** | Full metric coverage, X-Ray, runbook links | Composite alarms, Health events | Reduce alert fatigue with composite alarms |
| **4 — Predictive** | All of above + anomaly detection, SLO tracking | Chaos validation of monitoring | Validate alarms fire correctly during game days |
