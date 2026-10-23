# AWS Compute Optimizer

- Recommends optimal AWS resources (rightsizing) for workloads to **reduce cost and improve performance**, using **machine learning** on historical utilization metrics (CloudWatch)
- Free service (basic) - **opt-in** required (per account, or for all accounts from the management account / delegated admin)
- Regional recommendations, viewable per account or org-wide; data refreshed daily
- Findings: **Under-provisioned**, **Over-provisioned**, **Optimized**, (Lambda: also "Unavailable"), **Idle**
- Each recommendation shows up to 3 options with projected utilization, **performance risk** and estimated monthly savings (with or without your discounts)

## Supported Resources (2026)

- **EC2 instances**: instance type rightsizing (incl. move to **Graviton**, newer generations); not for Spot instances
- **EC2 Auto Scaling groups**: rightsizing of instance types (single type or mixed C/M/R types, CPU-based scaling policies) + idle recommendations
- **EBS volumes**: volume type, size, IOPS, throughput (gp2 => gp3, io1 => io2, previous-gen magnetic); unattached volumes
- **Lambda functions**: memory size (CPU scales with memory)
- **ECS services on Fargate**: task CPU/memory and container sizes
- **RDS and Aurora** (MySQL, PostgreSQL): DB instance class, RDS storage, Aurora Standard => **I/O-Optimized** recommendations; idle DB instances
- **Commercial software licenses**: Microsoft SQL Server on EC2 (e.g. Enterprise => Standard edition)
- **Idle resource recommendations**: EC2, ASG, EBS, RDS, NAT Gateway (2025), DynamoDB provisioned tables, ElastiCache (Valkey/Redis), MemoryDB, DocumentDB, WorkSpaces, SageMaker endpoints
- Requirements: enough metric history (EC2: at least 30 consecutive hours in the last 14 days), supported instance families

## Metrics and Recommendation Preferences

- Default lookback: **14 days** of CloudWatch metrics (free)
- **Enhanced infrastructure metrics** (paid): extends the lookback to **up to 93 days** (3 months) for EC2, ASG instances and RDS - captures monthly/quarterly cycles
  - Activated at organization, account or resource level (resource overrides account, account overrides org)
- **Memory**: not collected by default - install the **CloudWatch agent** (memory metrics) or use **external metrics ingestion** from Datadog, Dynatrace, Instana, New Relic => much better EC2 rightsizing accuracy (exam trap: "recommendations ignore memory")
- **Rightsizing preferences**: CPU/memory utilization thresholds and **headroom**, lookback (14/32/93 days), preferred instance families, Graviton/AWS-based CPU preference
- **Savings estimation mode**: with discounts (after RI/SP/private pricing) or public On-Demand prices
- **Inferred workload types** (e.g. Kafka, Redis, NGINX) to avoid recommending incompatible types; migration effort shown (e.g. x86 => Graviton)

## Organization-Level Enablement

- The **management account** can opt in all member accounts (trusted access with AWS Organizations) - recommendations for the whole org from one place
- A **delegated administrator** member account can view recommendations and manage preferences for the org (avoid using the management account)
- Member accounts can opt in individually to see their own recommendations

## Export and Integrations

- **Export** recommendations to **S3** (CSV + metadata JSON) for each resource type, per region - `ExportEC2InstanceRecommendations`, etc.; org-wide exports from the management/delegated admin account
- APIs: `GetEC2InstanceRecommendations`, `GetEBSVolumeRecommendations`, `GetLambdaFunctionRecommendations`, `GetIdleRecommendations`...
- Feeds **Cost Optimization Hub** (consolidated savings), **Trusted Advisor** cost checks (opt-in) and Cost Explorer rightsizing recommendations
- **Automation rules** (Nov 2025): automatically apply recommended EBS actions on a schedule (daily/weekly/monthly), scoped by region/tags, at account or org level - action types: **snapshot and delete EBS volumes unattached for 32+ days** and **upgrade EBS volume type** (previous gen => gp3/io2); actions can be rolled back (restore from snapshot)

## Compute Optimizer vs Trusted Advisor vs Cost Optimization Hub

| | Compute Optimizer | Trusted Advisor | Cost Optimization Hub |
| --- | --- | --- | --- |
| Purpose | ML rightsizing + idle detection from utilization | Broad best-practice checks (cost, security, fault tolerance, performance, limits, ops) | Single consolidated, deduplicated view of cost savings |
| Data | 14 days (93 with enhanced metrics), CPU/memory/network/IOPS | Account configuration and simple utilization thresholds | Recommendations from Compute Optimizer + SP/RI purchase + others |
| Scope | EC2, ASG, EBS, Lambda, ECS Fargate, RDS/Aurora, licenses, idle resources | Many services, 6 categories | Accounts and regions of the org |
| Cost | Free (enhanced metrics paid) | Full checks need Business Support+/Enterprise/Unified Ops | Free |
| Best answer when | "Choose the right instance type / Lambda memory based on actual usage" | "Check for security gaps, service limits, idle LBs, best practices" | "Prioritize all savings opportunities across the org, with my discounts" |

## Exam Tips

- "Rightsize EC2 based on CPU **and memory**" => Compute Optimizer + CloudWatch agent memory metrics
- "Workload has monthly peaks, recommendations are too aggressive" => enable **enhanced infrastructure metrics** (93 days)
- "Optimize Lambda memory/cost" => Compute Optimizer (or Lambda Power Tuning)
- "Org-wide rightsizing reports to S3 for analysis" => opt in at org level + export to S3
- "Automatically clean up unattached EBS volumes" => Compute Optimizer automation rules (or Config/Lambda custom remediation)
- Compute Optimizer gives recommendations only - except automation rules for EBS, it does not change resources
