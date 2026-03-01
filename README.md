# AWS Resiliency Plugin

Equip AI coding agents with the skills to review, validate, and harden AWS architectures for resiliency. Works with **Claude Code**, **Kiro**, **Cursor**, **Windsurf**, and any MCP-compatible editor.

> **Note:** Generative AI can make mistakes. Always review generated findings and recommendations before applying changes to production infrastructure.

![AWS Resiliency Demo](demo.svg)

---

## What This Plugin Does

The AWS Resiliency Plugin gives your AI agent a **failure-mode-first** approach to architecture reviews. Share your IaC or application code, and the agent reviews it across 7 resiliency domains — surfacing single points of failure, missing failover configurations, and application-level gaps that cause production incidents.

### Agent Skill Triggers

The skill activates when you ask questions like:

| Trigger Phrase | What It Does |
|---|---|
| *"Review this CDK/CloudFormation/Terraform for resiliency"* | Two-layer review: IaC configuration + application behaviour gaps |
| *"What happens if this AZ goes down?"* | Blast radius analysis with specific failure modes and RTO/RPO impact |
| *"Is this architecture highly available?"* | Well-Architected Reliability pillar review (REL 1-12) with RAG scoring |
| *"What's my RTO/RPO here?"* | DR pattern assessment with recovery time estimates |
| *"Help me prepare for a game day / ORR"* | Game day exercise templates, ORR checklists, runbook guidance |
| *"What are the failure modes for RDS/Lambda/SQS?"* | Service-specific failure modes with exact timeouts and propagation windows |

---

## Workflow

When you share infrastructure or application code, the plugin follows this workflow:

1. **Identify domains** — Detects which resiliency domains are relevant (Compute, Data, Networking, Storage, Messaging, Observability, Multi-Region & DR)
2. **Two-layer scan** — Reviews both the infrastructure configuration (Layer 1) and how the application behaves under failure (Layer 2)
3. **Surface failure modes** — Flags specific failure scenarios with blast radius, RTO/RPO impact, and Well-Architected references
4. **Provide fixes** — Includes corrected code/config snippets in the same IaC language as the input
5. **Prioritise findings** — Ranks by severity (Critical → Low) with effort estimates

---

## Two-Layer Review Model

Most production incidents live in the gap between infrastructure config and application behaviour. The skill always reviews **both layers**:

| Layer | What It Reviews | Example |
|---|---|---|
| **Layer 1: IaC** | Infrastructure topology, redundancy, failover config | RDS `MultiAZ: true` is correctly set |
| **Layer 2: Application Code** | SDK config, retry logic, timeouts, circuit breakers | But the app doesn't handle the 60-120s DNS failover window — 2-minute outage |

---

## Resiliency Domains

Reviews span **7 domains**, each with IaC and application code checks:

| Domain | Services | Example Findings |
|---|---|---|
| **Compute** | EC2, ECS, Lambda | Single instance (no ASG), Lambda missing DLQ, ECS `desiredCount: 1` |
| **Data** | RDS, Aurora, DynamoDB, ElastiCache | `MultiAZ: false`, missing PITR, connection pool exhaustion on failover |
| **Networking** | VPC, Route53, CloudFront, ALB/NLB | Single NAT Gateway, missing health checks, high DNS TTL |
| **Storage** | S3, EBS, EFS | Missing versioning, no snapshot schedule, burst credit exhaustion |
| **Messaging** | SQS, SNS, EventBridge | Missing DLQ, visibility timeout < processing time, no retry policy |
| **Observability** | CloudWatch, X-Ray | Missing alarms, `TreatMissingData: missing`, no business metrics |
| **Multi-Region & DR** | Global Tables, Aurora Global, Route53 ARC | No DR pattern defined, hardcoded regions, manual runbooks |

---

## Who Is This For

| Role | How This Helps |
|---|---|
| **Cloud Engineers** | Line-level findings on IaC and application code with corrected code examples |
| **SREs** | Validate failure modes, blast radius, and recovery objectives against actual infrastructure |
| **Platform Engineers** | Enforce resiliency standards across teams with compliance rules and scanning |
| **Cloud Architects** | Well-Architected Reliability pillar reviews with specific, quantified findings |

---

## Best Practices

- **Always review findings** before applying changes — the agent provides recommendations, not guarantees
- **Test in staging first** — validate remediation in non-production environments
- **Use with live account access** for the most complete reviews (CloudWatch metrics, alarm validation)
- Treat this plugin as an **accelerator, not a replacement** for human judgment on architecture decisions

---

## Plugin Components

### Agent Skill

| Component | Description |
|---|---|
| `skills/SKILL.md` | Core resiliency skill — failure-mode-first architect persona, two-layer review model, 7 domains, structured output format |
| `skills/references/service-failure-modes.md` | AWS service failure modes with exact timeouts, quota limits, and propagation windows |
| `skills/references/well-architected-reliability.md` | REL 1-12 pillar questions mapped to resiliency best practices |
| `skills/references/dr-patterns-and-runbooks.md` | DR pattern templates, game day exercise templates, ORR checklist |
| `skills/scripts/resiliency-review-template.md` | Structured output template for formal review deliverables |

### MCP Servers

The plugin configures 6 official AWS MCP servers from [`awslabs/mcp`](https://github.com/awslabs/mcp) to give the agent live tooling — IaC validation (cfn-lint, cfn-guard, Checkov), CloudWatch metrics and alarms, SLO tracking, and AWS documentation lookup. See [`mcp.json`](mcp.json) for the full configuration.

---

## Prerequisites

- Python 3.10+ with [**uv**](https://docs.astral.sh/uv/getting-started/installation/) installed — MCP servers run via `uvx`
- AWS credentials configured (`aws configure` or environment variables) — for CloudWatch and live account access

---

## Installation

### Claude Code

```bash
# Install the skill
git clone https://github.com/nirmal84/aws-resiliency-plugin.git
cp -r aws-resiliency-plugin/skills ~/.claude/skills/aws-resiliency

# Add MCP servers
claude mcp add aws-iac -- uvx awslabs.aws-iac-mcp-server@latest
claude mcp add aws-terraform -- uvx awslabs.terraform-mcp-server@latest
claude mcp add aws-knowledge -- uvx awslabs.aws-knowledge-mcp-server@latest
claude mcp add aws-cloudwatch -- uvx awslabs.cloudwatch-mcp-server@latest
claude mcp add aws-cloudwatch-signals -- uvx awslabs.cloudwatch-applicationsignals-mcp-server@latest
claude mcp add aws-docs -- uvx awslabs.aws-documentation-mcp-server@latest
```

### Kiro

Copy the `skills/` directory to your Kiro workspace and add the MCP servers from [`mcp.json`](mcp.json) to your Kiro MCP configuration.

### Cursor / Windsurf / Other MCP Editors

Copy `skills/` to your editor's skill location and merge [`mcp.json`](mcp.json) into your editor's MCP server configuration.

---

## File Structure

```
aws-resiliency-plugin/
├── skills/
│   ├── SKILL.md                                      # Core resiliency skill
│   ├── references/
│   │   ├── service-failure-modes.md                  # Failure modes per AWS service
│   │   ├── well-architected-reliability.md           # REL 1-12 pillar mapping
│   │   └── dr-patterns-and-runbooks.md               # DR patterns, game day, ORR
│   └── scripts/
│       └── resiliency-review-template.md             # Output template for reviews
├── mcp.json                                          # MCP server configuration
├── demo.svg                                          # Animated demo
├── README.md
├── CONTRIBUTORS.md
├── LICENSE                                           # MIT-0
└── .gitignore
```

---

## Related Resources

- [awslabs/mcp](https://github.com/awslabs/mcp) — Official AWS MCP servers
- [awslabs/agent-plugins](https://github.com/awslabs/agent-plugins) — AWS agent plugin framework

---

## Author

**Nirmal Rajan** — [@nirmal84](https://github.com/nirmal84)
