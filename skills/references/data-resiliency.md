# Data Resiliency — RDS, Aurora, DynamoDB, ElastiCache

Load this file when the user's code contains RDS, Aurora, DynamoDB, ElastiCache, Redshift, or any managed database resource.

---

## RDS (MySQL, PostgreSQL, SQL Server, Oracle, MariaDB)

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| Multi-AZ | `multiAz: true` | Single-AZ RDS = no automatic failover; manual recovery 10–30 min |
| Deletion protection | `deletionProtection: true` | Missing = accidental delete permanently destroys data |
| Backup retention | `backupRetentionPeriod` ≥ 7 days | Default 1 day; insufficient for enterprise RPO |
| Backup window | `preferredBackupWindow` set to low-traffic period | Default random window may hit peak hours |
| Maintenance window | `preferredMaintenanceWindow` set explicitly | Uncontrolled maintenance can cause unexpected restarts |
| Encrypted storage | `storageEncrypted: true` | Unencrypted = compliance failure + some failover limitations |
| Parameter group | Custom parameter group attached | Default group = suboptimal for production |
| Enhanced monitoring | `monitoringInterval` ≥ 60 (seconds) | Default 0 = no OS-level metrics |
| Performance Insights | Enabled | Missing = blind to slow query root cause during incidents |
| Auto minor version upgrade | `autoMinorVersionUpgrade: false` | Uncontrolled minor upgrades can cause unexpected restarts |
| Free storage alarm | CloudWatch alarm on `FreeStorageSpace` < 20% | Storage full = database stops accepting writes |

### Application Code Checks
**RDS Multi-AZ failover window (60–120 seconds)** — the most commonly missed gap:
- DNS TTL must be ≤ 5 seconds — check application DNS cache settings
- Java JDBC: `autoReconnect=true` and short `connectTimeout` required; JDBC URL caching holds old IP
- Connection pool must handle all connections dropping simultaneously and reconnecting — use exponential backoff with jitter
- `connect_timeout` must be low enough to detect failure quickly (≤ 5 seconds for production)
- `read_timeout` / `statement_timeout` must be set — infinite timeout = request hangs during failover

```python
# ✅ Python (psycopg2) with failover-resilient config
conn = psycopg2.connect(
    host=os.environ['DB_HOST'],      # cluster endpoint, not instance endpoint
    connect_timeout=5,               # detect failure quickly
    options='-c statement_timeout=30000',  # 30s statement timeout
)

# ✅ Connection pool with reconnection handling
pool = psycopg2.pool.ThreadedConnectionPool(
    minconn=2,
    maxconn=10,           # keep small during failover reconnect storm
    host=os.environ['DB_HOST'],
)
```

```java
// ✅ Java JDBC with failover config
String url = "jdbc:mysql://" + System.getenv("DB_HOST") + ":3306/mydb"
    + "?autoReconnect=true"
    + "&connectTimeout=5000"
    + "&socketTimeout=30000"
    + "&cachePrepStmts=true";
```

---

## Aurora (MySQL-compatible, PostgreSQL-compatible)

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| Reader instance | ≥ 1 Aurora Replica in different AZ | Single writer with no replica = 3–5 min recovery (creates new instance) |
| Replica tier | Tier-0 replica present | Higher tier replicas promoted last — tier 0 = fastest promotion |
| Deletion protection | `deletionProtection: true` | Critical — cluster deletion is irreversible |
| Backup retention | ≥ 7 days | |
| Aurora Serverless v2 | `serverlessV2ScalingConfiguration` with appropriate min/max ACU | Too low min ACU = cold scaling latency; too high = unnecessary cost |
| Global Database | Configured for cross-region? | Without Global DB, cross-region DR requires Aurora cross-region snapshot restore (hours) |
| CloudWatch alarms | On `AuroraReplicaLag`, `DatabaseConnections`, `ServerlessDatabaseCapacity` | Missing = blind to replication lag and capacity pressure |

### Aurora-Specific Failure Modes
- **No replica → primary failure**: Aurora creates a new instance (~3–5 minutes) vs. promoting existing replica (~15–30 seconds). Always have ≥ 1 replica in a different AZ.
- **Aurora Global Database unplanned failover**: Requires manual detach + promote in secondary region. With managed failover (preview): ~1 minute RTO.
- **Aurora Serverless v2 scaling lag**: Scales in ACU increments. Sudden burst → brief latency spike while scaling. Set `minCapacity` to warm value for production, not 0.5 ACU.
- **Writer endpoint vs. cluster endpoint**: Always use cluster endpoint — it automatically points to the current writer after failover. Never hardcode the writer instance endpoint.

---

## DynamoDB

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| Billing mode | `PAY_PER_REQUEST` or provisioned + auto-scaling | Provisioned without auto-scaling = throttling cliff on traffic burst |
| PITR | `pointInTimeRecovery: true` | Default OFF — table deletion or corruption = permanent data loss |
| Deletion protection | `deletionProtection: true` | Prevents accidental table delete (irreversible) |
| Global Tables | Configured for multi-region? | Single-region DynamoDB: regional failure = full data unavailability |
| CloudWatch alarms | On `ThrottledRequests`, `SystemErrors`, `ConsumedWriteCapacityUnits` | Missing = silent throttling |
| GSI capacity | GSI has same or higher capacity than base table | Under-provisioned GSI throttles base table writes |
| TTL | Configured for ephemeral data? | Missing TTL on time-series data = unbounded table growth |

### Application Code Checks
- **Missing exponential backoff on `ProvisionedThroughputExceededException`**: AWS SDK v2+ retries automatically; v1 requires manual handling
- **`UnprocessedItems` / `UnprocessedKeys` not handled**: `BatchWriteItem` / `BatchGetItem` can return partial results — must retry unprocessed items
- **Hot partition problem**: Sequential or low-cardinality partition keys concentrate writes → throttling even when total table capacity is sufficient. Use UUIDs, sharding, or write sharding.
- **Global Tables conflict resolution**: Last-writer-wins by timestamp — application must be designed for eventual consistency and idempotent writes
- **Transactional writes (`TransactWriteItems`)**: Costs 2x WCU; fails entire transaction on conflict — use sparingly and handle `TransactionConflictException`

```python
# ✅ DynamoDB client with proper retry config
import boto3
from botocore.config import Config

dynamodb = boto3.resource('dynamodb', config=Config(
    retries={
        'max_attempts': 10,
        'mode': 'adaptive'   # adaptive mode backs off based on throttle signals
    }
))

# ✅ BatchWriteItem with UnprocessedItems handling
def batch_write_with_retry(table, items):
    response = table.meta.client.batch_write_item(RequestItems={table.name: items})
    unprocessed = response.get('UnprocessedItems', {})
    while unprocessed:
        time.sleep(0.1)   # brief backoff before retry
        response = table.meta.client.batch_write_item(RequestItems=unprocessed)
        unprocessed = response.get('UnprocessedItems', {})
```

---

## ElastiCache (Redis / Valkey)

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| Replication group | `numCacheClusters` ≥ 2 | Single node = no failover; node failure = cache unavailable |
| Automatic failover | `automaticFailoverEnabled: true` | Manual failover only = human in the loop |
| Multi-AZ | `multiAzEnabled: true` | Single-AZ replication group = AZ failure = cache outage |
| Snapshot retention | `snapshotRetentionLimit` > 0 | Missing = no recovery option for data loss |
| Cluster mode | Enabled for large datasets / high throughput | Single shard = throughput ceiling and larger blast radius |
| Parameter group | Custom parameter group | Default `maxmemory-policy: noeviction` causes write failures on memory full |
| Subnet group | Spans ≥ 2 AZs | Single-AZ subnet group = AZ failure takes replication group offline |

### Application Code Checks
- **Cluster endpoint vs. primary endpoint**: Use the cluster configuration endpoint for cluster mode; primary endpoint for single-shard replication group. Hardcoding the primary node endpoint bypasses automatic failover.
- **Connection storm on failover (~20–60 second window)**: All application instances reconnect simultaneously → overwhelm the new primary. Implement connection pool with staggered reconnection and exponential backoff.
- **`maxmemory-policy`**: `noeviction` causes write failures when memory full. `allkeys-lru` is safest for cache workloads — evicts least-recently-used keys to make room.
- **Cross-slot commands in cluster mode**: `MGET`, `MSET`, `KEYS` with keys in different hash slots fail in cluster mode. Use hash tags `{user}:profile` to co-locate related keys.
- **Serialisation failures**: Missing try/catch on deserialise errors causes application crash on malformed cache entries — treat cache miss as the fallback, not an error.

```python
# ✅ Redis with connection resilience
import redis
from redis.backoff import ExponentialBackoff
from redis.retry import Retry

client = redis.Redis(
    host=os.environ['REDIS_CLUSTER_ENDPOINT'],
    port=6379,
    retry=Retry(ExponentialBackoff(), 6),
    retry_on_error=[redis.exceptions.ConnectionError, redis.exceptions.TimeoutError],
    socket_connect_timeout=5,
    socket_timeout=5,
)

# ✅ Cache-aside with fallback — never let cache miss be a hard failure
def get_user(user_id):
    try:
        cached = client.get(f'user:{user_id}')
        if cached:
            return json.loads(cached)
    except Exception:
        pass  # cache miss or Redis unavailable — fallback to DB
    user = db.query('SELECT * FROM users WHERE id = %s', user_id)
    try:
        client.setex(f'user:{user_id}', 3600, json.dumps(user))
    except Exception:
        pass  # don't fail the request if we can't write to cache
    return user
```

---

## CDK Patterns — Data Resiliency

```typescript
// ✅ RDS with Multi-AZ and full protection
const db = new rds.DatabaseInstance(this, 'PaymentsDB', {
  engine: rds.DatabaseInstanceEngine.postgres({ version: rds.PostgresEngineVersion.VER_15 }),
  instanceType: ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.LARGE),
  multiAz: true,
  deletionProtection: true,
  backupRetention: cdk.Duration.days(14),
  storageEncrypted: true,
  enablePerformanceInsights: true,
  monitoringInterval: cdk.Duration.seconds(60),
  removalPolicy: cdk.RemovalPolicy.RETAIN,
});

// ✅ DynamoDB with PITR and deletion protection
const table = new dynamodb.Table(this, 'Orders', {
  partitionKey: { name: 'orderId', type: dynamodb.AttributeType.STRING },
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
  pointInTimeRecovery: true,
  deletionProtection: true,
  removalPolicy: cdk.RemovalPolicy.RETAIN,
});
```

```typescript
// ❌ Common data layer failure patterns
const db = new rds.DatabaseInstance(this, 'DB', {
  multiAz: false,              // CRITICAL: no automatic failover
  backupRetention: cdk.Duration.days(1),  // HIGH: 1-day RPO
  deletionProtection: false,   // HIGH: accidental delete risk
  // no storageEncrypted       // MEDIUM: compliance and failover limitations
});

const table = new dynamodb.Table(this, 'Table', {
  billingMode: dynamodb.BillingMode.PROVISIONED,
  readCapacity: 5,
  writeCapacity: 5,
  // no pointInTimeRecovery    // HIGH: table corruption or delete = permanent loss
  // no deletionProtection     // HIGH: accidental delete risk
});
```
