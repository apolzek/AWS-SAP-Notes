# AWS Trusted Advisor

- Provides real time guidance to provision resources against AWS best practices
- It is an account level product, requires no agent to be installed
- Provides a number of checks and recommendations in 6 major areas:
  - Cost Optimization & Recommendations
  - Performance
  - Security
  - Fault Tolerance
  - Service Limit
  - Operational Excellence (added 2022)
- Total: hundreds of checks (~480+ in 2026), including checks powered by **AWS Config** managed rules, **Security Hub CSPM** controls and **Compute Optimizer** (opt-in) - not only the "classic" TA checks
- Trusted Advisor is not a free service, at least if we want to get out the most of it
- The free version is available if the account has **basic** or **developer** support plans
- Free version (2026 wording): **all Service Limits checks** + **selected Security and Fault Tolerance checks**, manual refresh only (no automatic refresh, no API)
- The free version provides 7 basic core checks <span style="color: #ff5733;">(Remember them for EXAM)</span> - classic list still used in exam material:
  - S3 bucket permissions (open access permissions)
  - Security Groups - specific ports unrestricted
  - IAM use
  - MFA on Root Account
  - EBS Public Snapshots
  - RDS Public Snapshots
  - 50 service limit checks: checks the 50 most common service limits and identifies any where we are over 80% of that limit
- Anything beyond these basic checks requires **Business Support+**, **Enterprise** or **Unified Operations** support plans (legacy Business and Enterprise On-Ramp until Jan 1, 2027)
- With those plans we get the full set of checks (older material says "further 115 checks" - now several hundred) with automatic refresh
- We also get access to the AWS Support API and the **Trusted Advisor API** (`trustedadvisor` - list recommendations, update lifecycle, org-wide aggregation)
- AWS Support API allows for programmatic access for AWS support functions:
  - We can get the names and identifiers for the checks AWS offers
  - We can request a Trusted Adviser check run against accounts and resources
  - Allows to get summaries and detailed information programmatically
  - Allows request for Truster Advisor refresh
- AWS Support API allows to programmatically open support ticket, and manage them
- With business and enterprise support we get CloudWatch integration => we can define event driven responses to actions
  - Today: check status changes are emitted to **EventBridge** (us-east-1) => Lambda/SNS/SSM Automation remediation; TA also publishes CloudWatch metrics (check status, service limit usage)
- **Organizational view**: aggregated TA report for all accounts of an AWS Organization (enable trusted access from the management account) - export as JSON/CSV
- **Trusted Advisor Priority** (Enterprise / Unified Operations): prioritized, context-aware recommendations curated by your account team (TAM), with lifecycle (accept, resolve, dismiss) and delegated admin
- Weekly email notifications to contacts; recommendations can be excluded (suppressed) per resource

## AWS Support Plans

> **Status (2026):** AWS restructured support plans (announced Dec 2025). **Developer**, **Business** (classic) and **Enterprise On-Ramp** are discontinued on **Jan 1, 2027** (Enterprise On-Ramp customers are auto-upgraded to Enterprise during 2026; legacy plans remain in GovCloud). New lineup: Basic, **Business Support+**, **Enterprise**, **Unified Operations**. Exam questions written earlier may still use the old names.

- Basic Support:
  - Is included for AWS customers and it is free
  - For Trusted Advisor with this support plan we get 7 core checks (see them above)
  - Account/billing support, AWS Health Dashboard, documentation, re:Post - no technical cases
- Developer (legacy, ends Jan 1, 2027):
  - For Trusted Advisor we get the same 7 base core checks (see them above)
  - Business-hours email access to Cloud Support Associates
- Business (legacy, ends Jan 1, 2027) / **Business Support+** (from $29/month per account):
  - We ge the full set of checks and recommendation
  - We get programmatic access to Trusted Advisor
  - 24/7 access to engineers; Business Support+ adds AI-powered contextual assistance and expert engagement within 30 min for critical issues
- Enterprise (minimum lowered to $5,000/month):
  - Same as business
  - Designated **Technical Account Manager (TAM)**, 15-min response for business-critical issues, Trusted Advisor Priority, Infrastructure Event Management, AWS Security Incident Response included
- **Unified Operations** (top tier, mission-critical): dedicated team (TAM, domain engineer, billing specialist), ~5-min response for critical incidents
- Support plans are **per account** - in an Organization each account needs its own plan (Enterprise is typically enabled for all accounts of the org with aggregated pricing)

## Service Quotas (vs Trusted Advisor)

- **Service Quotas** console/API: view default and applied quotas, current usage (for many services), and **request quota increases** directly (no support case needed)
- CloudWatch alarms on quota utilization (`AWS/Usage` namespace) - more precise than TA's 80% threshold
- **Quota request templates**: automatically request increases for **new accounts** in an Organization
- **Automatic quota management** (Oct/Nov 2025): notifications before reaching limits (email, Slack, AWS Health) and automatic quota adjustment based on usage
- Exam tip: "be alerted before hitting a limit" => Service Quotas + CloudWatch alarm (or TA service limit checks); "new accounts must start with higher limits" => quota request templates

## Good to Know

- We can check if an S3 bucket is made public, but we cannot check if objects are public in a bucket. Monitoring this we might use CloudWatch Events/S3 Events instead (today: S3 Block Public Access, IAM Access Analyzer for S3, Macie, AWS Config rules)
- Service Limits:
  - Limits can be monitored in Trusted Advisor (checks at 80% usage) **and in Service Quotas** (older material says "only in TA" - outdated)
  - Increases: request via **Service Quotas** (console/API) or a support case
- Trusted Advisor vs others:
  - TA: broad best-practice checks across 6 categories, account/org level
  - **Compute Optimizer**: ML-based rightsizing from utilization metrics (EC2, ASG, EBS, Lambda, ECS on Fargate, RDS/Aurora, licenses, idle resources)
  - **Cost Optimization Hub**: consolidates cost recommendations (Compute Optimizer rightsizing/idle, Savings Plans/RI purchases, Graviton migration) across accounts and regions with estimated savings
  - **Security Hub CSPM**: security posture against standards (CIS, FSBP, PCI) - TA shows some of these controls
  - **AWS Health**: account-specific events (maintenance, outages, retirement) - not best-practice checks
- See also [AWS Compute Optimizer](compute-optimizer.md) and [AWS Billing and Cost Management](billing.md)
