# Messaging Resiliency — SQS, SNS, EventBridge, Kinesis

Load this file when the user's code contains SQS, SNS, EventBridge, Kinesis, MSK, Amazon MQ, or event-driven messaging resources.

---

## SQS

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| Dead Letter Queue (DLQ) | `deadLetterQueue` with `maxReceiveCount: 3–5` | No DLQ = failed messages silently discarded after max attempts |
| Visibility timeout | `visibilityTimeout` > maximum processing time | VT ≤ processing time = guaranteed message duplication |
| Message retention | `retentionPeriod` ≥ expected recovery window | Default 4 days; too short = messages expire during extended outage |
| DLQ retention | DLQ `retentionPeriod` > source queue retention | DLQ messages must outlive source queue to allow investigation and redriving |
| CloudWatch alarm on DLQ | Alarm on DLQ `ApproximateNumberOfMessagesVisible` > 0 | Missing = poison pills accumulate silently |
| CloudWatch alarm on queue age | Alarm on `ApproximateAgeOfOldestMessage` | Consumer lag invisible without this alarm |
| FIFO vs Standard | Correct type for use case | FIFO for ordering guarantees; Standard for throughput — wrong choice causes either ordering failures or throughput bottleneck |
| KMS encryption | `encryptionMasterKey` set | Unencrypted SQS = compliance risk for regulated industries |

### Critical: Visibility Timeout Misconfiguration
The most common SQS failure mode in production:

```
Scenario: Lambda timeout=60s, SQS VisibilityTimeout=30s (default)

T+0s:   Lambda picks up message, VisibilityTimeout starts
T+30s:  VisibilityTimeout expires — message becomes visible again
T+31s:  Second Lambda picks up the SAME message — now processing in parallel
T+60s:  First Lambda completes and deletes message
T+61s:  Second Lambda completes, tries to delete — message already gone (silent)
Result: Double processing. Under load spikes (which slow Lambda), this compounds.
```

**Rule:** `visibilityTimeout` must be ≥ 6 × Lambda timeout (AWS recommendation for safety buffer).

```typescript
// ✅ SQS with proper configuration
const dlq = new sqs.Queue(this, 'PaymentsDLQ', {
  retentionPeriod: cdk.Duration.days(14),  // longer than source queue
  encryption: sqs.QueueEncryption.KMS_MANAGED,
});

const queue = new sqs.Queue(this, 'PaymentsQueue', {
  visibilityTimeout: cdk.Duration.seconds(360),  // 6× Lambda timeout of 60s
  retentionPeriod: cdk.Duration.days(7),
  deadLetterQueue: {
    queue: dlq,
    maxReceiveCount: 3,
  },
  encryption: sqs.QueueEncryption.KMS_MANAGED,
});

// Alarm on DLQ — any message in DLQ = failed processing
new cloudwatch.Alarm(this, 'DLQAlarm', {
  metric: dlq.metricApproximateNumberOfMessagesVisible(),
  threshold: 1,
  evaluationPeriods: 1,
  treatMissingData: cloudwatch.TreatMissingData.NOT_BREACHING,
});
```

### Application Code Checks
- **Idempotency is mandatory**: SQS Standard delivers at-least-once. Every consumer must handle duplicate messages without side effects. Use deduplication keys, database upserts, or idempotency tokens.
- **VisibilityTimeout extension during long processing**: Call `ChangeMessageVisibility` as a heartbeat for long-running jobs — prevents message redelivery while still being processed.
- **Batch processing partial failure**: If Lambda processes a batch of 10 messages and 2 fail, the entire batch retries by default. Use `reportBatchItemFailures` to return only the failed message IDs for retry, leaving successful messages deleted.
- **Poison pill detection**: A message that always fails will cycle through maxReceiveCount and land in DLQ. Without a DLQ alarm, this is silent. Always alarm on DLQ > 0.

```python
# ✅ Lambda SQS handler with batch item failure reporting
def handler(event, context):
    failures = []
    for record in event['Records']:
        try:
            process_message(json.loads(record['body']))
        except Exception as e:
            logger.error(f"Failed to process {record['messageId']}: {e}")
            failures.append({'itemIdentifier': record['messageId']})

    return {'batchItemFailures': failures}  # only failed items retry
```

---

## SNS

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| DLQ on HTTP/S subscriptions | `deadLetterQueue` on each subscription | Failed HTTP deliveries silently discarded after retry exhaustion |
| Delivery retry policy | Custom retry policy for HTTP/S endpoints | Default: 3 immediate + 2 pre-backoff + 10 backoff + 100,000 post-backoff retries over 23 days; verify this matches your use case |
| Subscription filter policies | `filterPolicy` on subscriptions | Without filters, all subscribers receive all messages — increased processing cost and unwanted coupling |
| Topic encryption | `masterKey` set | Unencrypted topic = compliance risk |
| Cross-account access | `addToResourcePolicy` with explicit principals | Missing = accidental public access or over-broad cross-account access |
| FIFO SNS + FIFO SQS | Paired correctly for ordering requirements | SNS FIFO can only fan out to SQS FIFO subscribers |

### Application Code Checks
- **Fan-out with one slow consumer**: SNS delivers to all subscribers independently — a slow consumer doesn't block others. But if one subscriber's SQS queue fills up and SNS delivery fails, messages go to the subscription DLQ.
- **Message size limit**: 256 KB per SNS message. For larger payloads, use the SNS extended client library (stores payload in S3, passes reference).
- **Duplicate delivery**: SNS delivers at-least-once. Subscribers must be idempotent.

---

## EventBridge

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| DLQ on rule targets | `deadLetterQueue` on each target | Failed deliveries exhausting retries silently discarded |
| Retry policy | `retryAttempts` and `maximumEventAgeInSeconds` set | Default: 185 retries over 24 hours — may be too long or too short for your use case |
| Archive and replay | `EventBus.archive()` configured | Without archive, events cannot be replayed after consumer outage |
| CloudWatch alarm on FailedInvocations | Alarm on `FailedInvocations` metric | Missing = silent delivery failures |
| Event bus resource policy | Explicit cross-account principals | Default = account-only; cross-account requires explicit policy |
| Rule `State` | `ENABLED` | Disabled rules look correct in IaC review but don't fire |

### Application Code Checks
- **Idempotency required**: EventBridge delivers at-least-once. Targets must handle duplicate events.
- **Event size limit**: 256 KB per event. Use S3 pointer pattern for larger payloads.
- **Schema validation**: Without `SchemaRegistry`, producers can silently change event structure breaking consumers. Use EventBridge Schema Registry for contract enforcement.
- **Target invocation failures**: Lambda throttles, SQS queue full, and Step Functions API throttles all cause EventBridge to retry. Without a DLQ, exhausted retries = silent event loss.

```typescript
// ✅ EventBridge rule with DLQ and retry policy
const dlq = new sqs.Queue(this, 'EventBridgeDLQ');

new events.Rule(this, 'OrderProcessingRule', {
  eventBus,
  eventPattern: { source: ['com.myapp.orders'] },
  targets: [new targets.LambdaFunction(orderProcessor, {
    deadLetterQueue: dlq,
    retryAttempts: 3,
    maxEventAge: cdk.Duration.hours(2),
  })],
});

// Archive for replay capability
eventBus.archive('OrdersArchive', {
  archiveName: 'orders-archive',
  retention: cdk.Duration.days(30),
  eventPattern: { source: ['com.myapp.orders'] },
});
```

---

## Kinesis Data Streams

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| Shard count | Sized for peak throughput + 20% headroom | Under-provisioned = `ProvisionedThroughputExceededException` |
| Enhanced fan-out | Enabled for consumers requiring low latency | Standard GetRecords polling adds 200ms–1s latency; enhanced fan-out = ~70ms |
| Retention period | `retentionPeriod` appropriate for recovery window | Default 24 hours; extend to 7 days for re-processing capability |
| Server-side encryption | `encryption: StreamEncryption.KMS` | Unencrypted stream = compliance risk |
| On-demand vs. provisioned | `streamMode: StreamMode.ON_DEMAND` for variable throughput | Provisioned mode with wrong shard count = over/under provisioning |
| CloudWatch alarm | On `GetRecords.IteratorAgeMilliseconds` | High iterator age = consumer falling behind; silent without alarm |

### Application Code Checks
- **Shard key distribution**: Hot shards (low-cardinality partition keys) throttle despite unused capacity in other shards. Use high-cardinality shard keys or composite keys.
- **Iterator age monitoring**: `GetRecords.IteratorAgeMilliseconds` is the most important Kinesis operational metric. High iterator age = consumer lag; action needed before retention window expires and data is lost.
- **Retry on `ProvisionedThroughputExceededException`**: AWS SDK retries automatically, but without exponential backoff and jitter, retry storms can amplify the problem.
- **Checkpoint management**: When using Kinesis Client Library (KCL), checkpoint after successful processing — not before. Checkpoint before = data loss on failure; checkpoint after is idempotent.

```python
# ✅ Producer with shard key distribution
import hashlib

def get_shard_key(order_id, num_shards=10):
    # Distribute evenly across logical shards
    shard_num = int(hashlib.md5(order_id.encode()).hexdigest(), 16) % num_shards
    return f"shard-{shard_num}"

kinesis.put_record(
    StreamName='orders-stream',
    Data=json.dumps(order),
    PartitionKey=get_shard_key(order['id']),
)
```

---

## General Messaging Patterns — Resiliency Anti-Patterns

| Anti-pattern | Risk | Fix |
|---|---|---|
| Synchronous chain A → B → C | Failure at C fails entire chain | Use queues between services; async fan-out |
| Non-idempotent consumer | At-least-once delivery causes duplicate side effects | Use idempotency keys, upserts, or deduplication tokens |
| Missing DLQ everywhere | Failed messages silently lost | DLQ on every queue, topic subscription, and EventBridge target |
| Visibility timeout = processing timeout | Guaranteed duplicate processing under any slowdown | VT = 6× processing timeout |
| Fire-and-forget on publish failure | Data loss if publish fails | Retry publish with exponential backoff; use transactional outbox pattern for critical data |
| Outbox pattern missing | Messages can be lost between DB commit and publish | Write event to DB in same transaction; separate process publishes from outbox table |
