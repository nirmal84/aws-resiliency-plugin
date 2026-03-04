# Networking Resiliency — VPC, ALB/NLB, Route53, CloudFront, Global Accelerator

Load this file when the user's code contains VPC, subnets, ALB, NLB, Route53, CloudFront, NAT Gateway, or Global Accelerator resources.

---

## VPC & Subnets

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| AZ count | Subnets across ≥ 3 AZs for production | 2 AZs = AZ failure halves capacity; 3 AZs absorbs one failure gracefully |
| NAT Gateway per AZ | One NAT GW per AZ used by private subnets | Single shared NAT GW = AZ-level SPOF for **all** outbound traffic from private subnets |
| VPC endpoints | S3 and DynamoDB gateway endpoints; interface endpoints for Secrets Manager, SSM, CloudWatch, ECR | Traffic through NAT GW = cost + single point of failure for high-volume AWS API traffic |
| CIDR sizing | VPC CIDR allows future growth | Cannot expand VPC CIDR after creation; plan for 3x current subnet needs |
| VPC Flow Logs | Enabled on VPC or subnet | Missing = no forensic capability after network incidents; required for some compliance frameworks |
| NACL rules | Not overly restrictive | Default NACL allows all; custom NACLs with missing ephemeral port rules cause silent connection failures |

### Key Failure Modes
**Single NAT Gateway** → AZ failure takes down NAT GW → all private subnet instances lose outbound internet and AWS API connectivity, not just those in the failed AZ. This is the most commonly overlooked AZ-level SPOF.

**Missing VPC endpoints** → Lambda/ECS in private subnets calling S3/DynamoDB routes traffic through NAT GW → NAT GW bandwidth exhaustion under high load; NAT GW failure blocks all data access.

```typescript
// ✅ VPC with NAT Gateway per AZ
const vpc = new ec2.Vpc(this, 'ProductionVpc', {
  maxAzs: 3,
  natGateways: 3,  // one per AZ — do NOT set to 1 for production
  subnetConfiguration: [
    { cidrMask: 24, name: 'Public', subnetType: ec2.SubnetType.PUBLIC },
    { cidrMask: 22, name: 'Private', subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
  ],
});

// ✅ VPC endpoints for critical AWS services
new ec2.GatewayVpcEndpoint(this, 'S3Endpoint', {
  vpc,
  service: ec2.GatewayVpcEndpointAwsService.S3,
});
new ec2.InterfaceVpcEndpoint(this, 'SecretsManagerEndpoint', {
  vpc,
  service: ec2.InterfaceVpcEndpointAwsService.SECRETS_MANAGER,
  privateDnsEnabled: true,
});
```

---

## ALB / NLB

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| Multi-AZ | ALB deployed across ≥ 2 AZs | Single-AZ ALB = AZ failure = full service outage |
| Cross-zone load balancing | Enabled | Without it, unequal instance distribution across AZs causes overload in remaining AZs on failure |
| Health check path | Deep health check endpoint (tests DB connectivity, not just HTTP 200) | Root path `/` health check passes even when application is broken |
| Health check interval | `healthCheckIntervalSeconds` ≤ 30 | 60-second default = slow detection; 30s standard; 10s fast (additional cost) |
| Unhealthy threshold | `unhealthyThresholdCount` ≤ 3 | High threshold = broken targets stay in rotation longer |
| Healthy threshold | `healthyThresholdCount` ≤ 3 | High threshold = recovered targets join rotation slowly after an incident |
| Deregistration delay | 30–60 seconds for rolling deployments | Default 300s = slow deployments; 0s = in-flight connections cut abruptly |
| Deletion protection | Enabled | Missing = accidental deletion of load balancer |
| Access logs | Enabled to S3 | Missing = no request-level forensics after incidents |
| WAF association | WAF WebACL attached | Missing = no rate limiting protection against traffic-based failures |

### Application Code Checks
- **ALB idle timeout**: Default 60 seconds. Application keep-alive must be > 60s or ALB closes idle connections. Common cause of mysterious 502 errors.
- **NLB source IP preservation**: NLB preserves client IP by default — application must accept connections from internet IPs even when behind NLB. Security groups must allow the full internet range for internet-facing NLB.
- **Stickiness**: Session affinity (`stickinessCookieDuration`) hides stateful design problems — prefer stateless services and shared session stores (ElastiCache).

```typescript
// ✅ ALB with deep health check and access logging
const alb = new elbv2.ApplicationLoadBalancer(this, 'ALB', {
  vpc,
  internetFacing: true,
  deletionProtection: true,
});
alb.logAccessLogs(logsBucket);

const targetGroup = new elbv2.ApplicationTargetGroup(this, 'TG', {
  vpc,
  port: 8080,
  protocol: elbv2.ApplicationProtocol.HTTP,
  healthCheck: {
    path: '/health/live',       // deep health check — tests DB, dependencies
    interval: cdk.Duration.seconds(30),
    healthyThresholdCount: 2,
    unhealthyThresholdCount: 3,
    timeout: cdk.Duration.seconds(5),
  },
  deregistrationDelay: cdk.Duration.seconds(30),  // fast rolling deployments
});
```

---

## Route53

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| Health checks on critical records | `HealthCheckId` attached to all production DNS records | Missing = Route53 never removes unhealthy endpoints |
| Health check type | `HTTP` or `HTTPS` with string match | TCP-only check passes even when application returns errors |
| Health check interval | 30s standard; 10s fast | 30s × 3 failures = 90s detection before DNS update |
| Failover routing | `FAILOVER` or `WEIGHTED` routing with health check | `SIMPLE` routing = no automatic failover |
| TTL on failover records | ≤ 60 seconds | High TTL = clients continue using stale IP long after failover |
| Geolocation default record | Present when geolocation routing used | Missing default = DNS failure for clients not matching any geolocation rule |
| Route53 ARC | Zonal shift configured for Tier 1 workloads | Without ARC, AZ evacuation requires manual DNS changes |

### Route53 Failover Timeline
```
T+0s:    AZ or endpoint failure occurs
T+30s:   Route53 health check detects failure (3 failed checks × 10s fast interval)
         Or T+90s with 30s standard interval
T+31s:   Route53 DNS begins pointing to failover record
T+61s:   Recursive resolvers with 60s TTL start seeing new record
T+120s:  Most clients have failed over (some resolvers ignore TTL)
```
**Total failover time: 90–120 seconds** with standard health checks. Use Route53 ARC zonal shift for near-instant AZ evacuation independent of TTL.

### Application Code Checks
- **DNS caching in JVM**: `networkaddress.cache.ttl` defaults to 30s in recent JDKs but may be set to infinite in older configs. Check `$JAVA_HOME/jre/lib/security/java.security`.
- **Hardcoded IPs**: Any hardcoded IP bypasses DNS-based failover. Must use DNS names exclusively.
- **HTTP client connection reuse with old IP**: Long-lived HTTP connections bypass DNS resolution after initial connect. Set `maxConnectionsPerHost` and `maxIdleTime` to force periodic reconnection.

---

## CloudFront

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| Origin failover | Origin group with primary + secondary origin | Single origin = origin failure = 100% error rate for cache misses |
| Custom error pages | Static error page (S3-hosted) configured for 5xx | Without it, CloudFront returns AWS default error page on origin failure |
| Origin connection attempts | `connectionAttempts: 3` | Default is fine; lower value = faster failover to secondary |
| Origin response timeout | Set explicitly | Default 30s; too high if origin is consistently slow |
| Cache TTL | `defaultTtl` appropriate for content type | 0s TTL = every request hits origin (no availability benefit); too-high TTL = stale content after deployment |
| Price class | `PriceClass.PRICE_CLASS_ALL` for global availability | Limiting edge locations reduces geographic availability |

### Key Failure Mode
**Cache miss during origin failure with no secondary origin** → CloudFront fetches from origin → origin returns 5xx → CloudFront forwards error to user. Mitigation: configure origin group (primary + secondary) and `customErrorResponse` to serve a static S3 error page on 5xx.

---

## Global Accelerator

Use Global Accelerator instead of (or in addition to) Route53 when:
- TCP/UDP traffic (not just HTTP) needs global failover
- Consistent anycast IP is required (allow-listing use case)
- Sub-30-second failover is required (GA detects failure in ~10s, Route53 takes 90–120s)

### IaC Checks
| Check | PASS | FAIL / Flag |
|-------|------|-------------|
| Multiple endpoint groups | ≥ 2 regions with endpoints | Single endpoint group = no global failover |
| Health checks on endpoints | Enabled | Missing = GA never removes unhealthy endpoints |
| Traffic dial | `trafficDialPercentage` for each group | Set to 0 in DR region for passive standby; 100 in both for active-active |
| Client affinity | `NONE` for stateless; `SOURCE_IP` for stateful | Wrong setting causes requests to bounce between regions for stateful apps |
