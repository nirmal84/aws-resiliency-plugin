# Storage Resiliency — S3, EBS, EFS, AWS Backup

Load this file when the user's code contains S3, EBS, EFS, AWS Backup, FSx, or storage-related resources.

---

## S3

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| Versioning | `versioned: true` | Without versioning, object overwrites and deletes are **permanent and irreversible** |
| MFA Delete | Enabled for critical buckets | Prevents accidental or malicious bulk deletion even with valid credentials |
| Block Public Access | All 4 flags `true` | Any `false` = potential public data exposure |
| Replication (CRR) | Cross-region replication for DR or compliance | Single-region S3 is durable (11 nines) but unavailable if region fails |
| Object Lock | WORM mode for compliance data | Prevents deletion by any user including root during retention period |
| Lifecycle rules | Configured for old versions | Unbounded version accumulation = storage cost + operational noise |
| Event notifications | Configured for critical operations | Missing = no awareness of unexpected deletes, errors, or pipeline failures |
| Server-side encryption | `SSE-S3` minimum; `SSE-KMS` for sensitive data | Unencrypted = compliance failure |
| Intelligent tiering | For large buckets with unpredictable access | Without it, infrequently accessed objects incur full Standard pricing |

### Application Code Checks
- **Missing retry on `SlowDown` (503)**: S3 throttles at ~3,500 PUT/s and 5,500 GET/s per prefix. Must retry with exponential backoff.
- **Large object uploads without multipart**: Single-part upload failure = restart entire upload. Use multipart for objects > 100 MB.
- **Prefix design for throughput**: Low-cardinality prefixes (e.g., `YYYY/MM/DD/`) concentrate requests → throttling. Use high-cardinality prefixes or S3 request rate increase.
- **`ListObjects` on large buckets**: Paginated listing is slow and expensive at scale. Use S3 Inventory for bulk operations.

```python
# ✅ S3 multipart upload with retry
import boto3
from boto3.s3.transfer import TransferConfig

s3 = boto3.client('s3')
config = TransferConfig(
    multipart_threshold=100 * 1024 * 1024,   # 100 MB
    max_concurrency=10,
    multipart_chunksize=50 * 1024 * 1024,    # 50 MB chunks
    use_threads=True
)
s3.upload_file('large_file.csv', 'my-bucket', 'large_file.csv', Config=config)
```

### S3 Durability vs Availability
```
S3 Standard durability:      99.999999999% (11 nines) — data loss is not the risk
S3 Standard availability:    99.99% — regional service disruption is the risk
S3 One Zone-IA:              Single AZ — AZ failure = data unavailable (not lost if AZ recovers)
S3 Replication lag (standard): Minutes (best-effort, not guaranteed)
S3 RTC (Replication Time Control): 99.99% of objects replicated within 15 minutes (with SLA)
```
> **Key insight:** S3 is a durability solution, not an availability solution. For HA, use CRR + Route53 failover or CloudFront with multi-region origin group.

---

## EBS

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| Snapshot schedule | AWS Backup or DLM policy creates snapshots ≥ daily | Missing = manual snapshots only; human-dependent RPO |
| `deleteOnTermination` | `false` for data volumes | Default `true` = data volume deleted when instance terminates |
| Encryption | `encrypted: true` | Unencrypted EBS = compliance failure; cannot encrypt in-place (must snapshot + restore) |
| Volume type | `gp3` preferred over `gp2` | `gp2` IOPS tied to size; burst credits exhaust under sustained load |
| Snapshot lifecycle | Old snapshots cleaned up via DLM | Unbounded snapshot accumulation = cost + operational noise |
| Multi-Attach | Only for `io1`/`io2`; requires cluster-aware app | Wrong volume type for multi-attach = silent failure |

### Key Failure Modes
**EBS I/O suspension during snapshot**: For non-EBS-optimised instances or `standard` (magnetic) volumes, snapshot initiation causes brief I/O pause (~1 second). For `gp2/gp3/io1/io2`, impact is minimal but not zero — schedule snapshots during low-traffic windows.

**Single-AZ limitation**: EBS volumes exist in one AZ. Instance termination in that AZ = volume inaccessible until instance restarts in same AZ. EBS cannot move across AZs without snapshot + restore.

**gp2 burst credit exhaustion**: Small `gp2` volumes (< 1 TB) rely on burst IOPS (3,000 burst vs. 3 IOPS/GB baseline). Under sustained I/O, credits exhaust and IOPS drop to baseline. Migrate to `gp3` for independent IOPS configuration.

```typescript
// ✅ EBS with backup and encryption
const instance = new ec2.Instance(this, 'AppServer', {
  blockDevices: [{
    deviceName: '/dev/xvda',
    volume: ec2.BlockDeviceVolume.ebs(100, {
      volumeType: ec2.EbsDeviceVolumeType.GP3,
      encrypted: true,
      deleteOnTermination: false,  // preserve on instance termination
      iops: 3000,
      throughput: 125,
    }),
  }],
});

// ✅ AWS Backup plan for EBS
const plan = new backup.BackupPlan(this, 'EBSBackupPlan', {
  backupPlanRules: [
    new backup.BackupPlanRule({
      scheduleExpression: events.Schedule.cron({ hour: '3', minute: '0' }),
      deleteAfter: cdk.Duration.days(30),
      moveToColdStorageAfter: cdk.Duration.days(7),
    }),
  ],
});
```

---

## EFS

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| Multi-AZ mount targets | Mount target in each AZ used by application | Missing mount target in an AZ = instances in that AZ cannot mount EFS |
| Throughput mode | `ELASTIC` (recommended) or `PROVISIONED` for sustained high throughput | `BURSTING` exhausts burst credits under sustained load; credit recovery is slow |
| Backup policy | `ENABLED` | Default `DISABLED`; no backup = data loss risk on file corruption |
| Encryption | `encrypted: true` | Required for compliance |
| Performance mode | `MAX_IO` for high-concurrency workloads (≥ 100 EC2 instances) | Default `GENERAL_PURPOSE` throttles at high file operation rates |
| Lifecycle policies | Configured for infrequently accessed files | Without lifecycle, all files stay in Standard class regardless of access pattern |

### Application Code Checks
- **NFS mount timeout handling**: EFS latency spikes under high concurrency — application must handle NFS timeout errors, not crash
- **Mount helper with TLS**: Use the AWS EFS mount helper (`amazon-efs-utils`) for automatic TLS and retry — raw NFS mount lacks automatic encryption and retry on transient mount failures
- **Connection draining on unmount**: Application must finish writes and flush before unmounting — NFS client does not guarantee flush on process exit

---

## AWS Backup

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| Backup plan covers all critical resources | EC2, RDS, DynamoDB, EFS, EBS, S3 (via Backup) in backup plan | Point-in-time recovery settings on individual resources alone = no centralised visibility |
| Cross-region copy | `copyActions` with destination vault in DR region | Single-region backups: region failure = backup unavailable when most needed |
| Cross-account copy | Backup vault in separate account | Same account as production = ransomware or account compromise deletes both |
| Retention period | Matches RPO and compliance requirements | Default varies by resource; shorter than compliance requirement = audit failure |
| Vault lock | Enabled for compliance-critical vaults | Without vault lock, backups can be deleted within retention period |
| Restore testing | Automated restore test configured | Backup without tested restore = false confidence; treat as untested until proven |

```typescript
// ✅ AWS Backup with cross-region copy
const vault = new backup.BackupVault(this, 'PrimaryVault', {
  encryptionKey: kmsKey,
  removalPolicy: cdk.RemovalPolicy.RETAIN,
});

const plan = new backup.BackupPlan(this, 'CriticalDataBackup', {
  backupPlanRules: [
    new backup.BackupPlanRule({
      backupVault: vault,
      scheduleExpression: events.Schedule.cron({ hour: '1', minute: '0' }),
      deleteAfter: cdk.Duration.days(35),
      copyActions: [{
        destinationBackupVault: drVault,  // vault in DR region
        deleteAfter: cdk.Duration.days(35),
      }],
    }),
  ],
});

// Add all critical resources to backup selection
plan.addSelection('CriticalResources', {
  resources: [
    backup.BackupResource.fromRdsDatabaseInstance(db),
    backup.BackupResource.fromDynamoDbTable(table),
    backup.BackupResource.fromTag('Backup', 'true'),  // tag-based selection
  ],
});
```
