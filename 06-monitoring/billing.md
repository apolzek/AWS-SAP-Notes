# AWS Billing and Cost Management

- Umbrella console for billing, cost visibility, budgeting and optimization: Bills, Payments, Cost Explorer, Budgets, Cost Anomaly Detection, Data Exports (CUR), Cost Optimization Hub, Billing Conductor, Pricing Calculator, Free Tier
- Billing data is global, but the **billing metrics** in CloudWatch (`EstimatedCharges`) live only in **us-east-1** (must enable "Receive Billing Alerts")
- IAM users/roles need the IAM access to billing activated by the root user, plus fine-grained IAM actions (`aws-portal:*` replaced by `billing:*`, `ce:*`, `budgets:*`, `cur:*`, `bcm-data-exports:*`, etc.)

## Consolidated Billing (AWS Organizations)

- The **management account** (payer) pays for all member accounts - one bill
- Usage is **aggregated** across accounts for volume pricing tiers (S3, data transfer, etc.)
- **RI and Savings Plans discount sharing**: by default, unused RI/SP benefits float to other accounts of the org (applied first to the owning account)
  - Sharing can be turned off per account in the management account (billing preferences) - e.g. isolate a business unit's commitments
- Member accounts can't see other accounts' costs (unless given access to the management account or via billing views)
- The management account should hold no workloads (best practice) and be tightly protected
- **Invoice configuration**: split the org bill into multiple invoices (invoice units, e.g. per business entity)
- **Billing Transfer** (2025): transfer billing/payment responsibility of an org to another org (e.g. a reseller/partner) while keeping the org structure

## AWS Cost Explorer

- Visualize, understand and manage costs and usage over time - filters/group by service, account, region, tag, cost category, usage type, purchase option
- Default history: last **13 months** (opt-in up to **38 months** of multi-year data) + **forecast up to 18 months** (Nov 2025, with AI-powered explanations of drivers)
- Granularity: monthly, daily; **hourly and resource-level** for last 14 days (opt-in, paid)
- Cost views: unblended, blended, amortized (spreads upfront RI/SP fees over the term), net (after discounts)
- Recommendations: **RI and Savings Plans purchase** recommendations (based on 7/30/60 days), **RI/SP utilization and coverage** reports, rightsizing recommendations (EC2 - powered by Compute Optimizer)
- API: `GetCostAndUsage`, `GetCostForecast` - charged per API request ($0.01)
- **Savings Plans Purchase Analyzer**: model different commitment levels (savings, coverage, utilization)

## AWS Budgets

- Set custom budgets and get alerted on **actual** or **forecasted** values
- Budget types:
  - **Cost budget**
  - **Usage budget** (e.g. GB of S3, EC2 hours)
  - **Savings Plans / RI utilization** budgets (alert when commitments are under-used)
  - **Savings Plans / RI coverage** budgets (alert when too much on-demand)
- Periods: daily, monthly, quarterly, annually; fixed, planned (per-period amounts) or **auto-adjusting** budgets (based on historical spend)
- Filters by account, service, tag, cost category, region, etc.; can be created by member accounts or centrally; **billing views** let budgets run on filtered cross-account data without management account access (2025)
- Notifications: up to 5 thresholds per budget (e.g. 80% forecasted, 100% actual), via email, **SNS** (=> Chatbot/Slack, Lambda)
- Data refresh: up to 3 times a day - budgets are **not real-time** (spend can still overshoot)
- **Budget Actions**:
  - Action types: apply an **IAM policy** (to user/group/role), apply an **SCP** (to an OU/account - from the management account), or **stop EC2 / RDS instances** (same account only)
  - Run automatically or after **manual approval** (workflow); up to 10 actions per threshold
  - IAM/SCP actions reset each budget period; instance-stop actions don't
  - Requires an IAM execution role for AWS Budgets
  - Exam tip: "automatically prevent developers from launching more resources when the sandbox budget is exceeded" => Budget action applying a deny SCP/IAM policy
- **Budget reports**: scheduled reports of budgets by email
- Pricing: budgets without actions are free; first 2 action-enabled budgets free, then $0.10/day each; budget reports $0.01 per report

## AWS Cost Anomaly Detection

- Free, ML-based service that detects **unusual spend** (vs historical patterns) and identifies **root causes** (up to top 10 by service, account, region, usage type)
- **Monitors**: AWS services, linked accounts, cost categories, cost allocation tags (AWS managed or customer managed)
- Alert subscriptions: individual alerts (SNS - near real time after detection) or daily/weekly summaries (email); threshold by $ or %
- Needs ~10 days of history; relies on Cost Explorer data (latency up to 24h) => not real-time
- Exam tip: "detect unexpected cost spikes without defining thresholds" => Cost Anomaly Detection (vs Budgets = known thresholds)

## Cost and Usage Report (CUR) / AWS Data Exports

- **AWS Data Exports**: create recurring exports of billing/cost data to **S3** (Parquet or CSV, gzip), selecting columns and filtering rows with SQL
- Export types / tables:
  - **CUR 2.0** (recommended): most granular cost data (hourly/daily/monthly, resource IDs, tags, RI/SP details), **fixed schema** (columns don't change monthly), nested columns for tags/product, optional **split cost allocation data** (ECS/EKS)
  - **FOCUS 1.0 / 1.2** with AWS columns: FinOps Open Cost and Usage Specification - standardized multi-cloud billing schema
  - **Cost optimization recommendations** (from Cost Optimization Hub)
  - **Carbon emissions**
  - Cost and Usage Dashboard (QuickSight-based, powered by Data Exports)
- **Legacy CUR** still supported (dynamic columns) - new setups should use CUR 2.0
- Typical analytics pipeline: Data Exports => S3 => Glue crawler/Athena => QuickSight (or CUDOS / Cloud Intelligence Dashboards)
- Data is refreshed up to 3 times a day; can be exported from the management account (whole org) or member accounts (own data)
- Exam tip: "most detailed, line-item cost data for custom analysis/chargeback" => CUR 2.0 via Data Exports + Athena

## Cost Optimization Hub

- Free feature that **consolidates and deduplicates** cost optimization recommendations across accounts and regions of an Organization
- Sources: Compute Optimizer (rightsizing, idle resources, Graviton migration, EBS/Lambda/ECS/RDS), Savings Plans and RI purchase recommendations, upgrades (e.g. previous-generation instances)
- Savings estimates take into account **your pricing** (RI, SP, private pricing discounts) and avoid double counting (e.g. rightsizing vs SP purchase)
- Group/filter by account, region, resource type, action type (rightsize, stop, delete, upgrade, purchase, migrate to Graviton), implementation effort
- Enable from the management account for the org; one **delegated administrator** member account possible
- Recommendations exportable via Data Exports; also visible in Trusted Advisor
- Exam tip: "single place to find and prioritize savings across the org" => Cost Optimization Hub

## AWS Billing Conductor

- Custom **pro forma billing** for showback/chargeback - mostly for **resellers/partners** and enterprises that re-bill internal business units with their own rates
- Concepts:
  - **Billing groups**: sets of accounts with a primary account (up to 1,000 accounts per group)
  - **Pricing plans / pricing rules**: global or per-service markup/discount, tiering
  - **Custom line items**: credits or fees (e.g. support fee, shared cost allocation)
- Produces pro forma Cost Explorer views and CUR for each billing group - does **NOT** change the actual AWS invoice
- Pricing: per account in billing groups (tiered); free with Billing Transfer AWS-managed pricing plans

## Billing Views and Dashboards

- **Custom billing views**: filtered (by account/tag) views of cost data that can be shared with member accounts or even **other organizations** (2025), combined into multi-source views (up to 20 payer accounts) - used by Cost Explorer and Budgets
- **Billing and Cost Management Dashboards**: customizable dashboards combining Cost Explorer and Savings Plans widgets, shareable, scheduled email delivery (2026)

## AWS Pricing Calculator

- Public calculator (no login) for estimates of new workloads
- **In-console (authenticated) Pricing Calculator**:
  - **Workload estimates**: estimate a specific workload, using your discounts and commitments (RI/SP/private pricing)
  - **Bill estimates**: model changes to the whole consolidated bill, incl. new **RI/SP purchases** (management/standalone account only)
  - Can import historical usage as a baseline

## AWS Free Tier (2025 change)

- Accounts created on or after **July 15, 2025**: choose **Free plan** or **Paid plan** at sign-up
  - $100 credits at sign-up + up to $100 more by completing onboarding activities
  - Free plan: up to **6 months** or until credits are used; restricted (no Marketplace, no large/dedicated offerings) and the account closes at the end unless upgraded to Paid
- Accounts created before: legacy 12-month free tier + always-free offers
- 30+ **always-free** services (e.g. Lambda 1M requests/month, DynamoDB 25 GB)
- Free Tier usage alerts (email) and AWS Budgets zero-spend budget template

## Other Cost Tools and Concepts

- **Cost Categories** and **cost allocation tags**: see [Cost Allocation Tags](cost-allocation-tags.md)
- **AWS Compute Optimizer**: see [AWS Compute Optimizer](compute-optimizer.md)
- **Trusted Advisor** cost optimization checks: see [AWS Trusted Advisor](trusted-advisor.md)
- **AWS Purchase Order Management**, **Payment profiles** (different payment methods per service provider/currency)
- **Customer Carbon Footprint Tool**: emissions estimates (now also via Data Exports)
- **Billing and Cost Management MCP server / Amazon Q** for natural-language cost analysis
- Data transfer cost basics (exam): inbound free; same AZ via private IP free; **cross-AZ ~$0.01/GB each direction**; cross-region and internet egress charged; use VPC endpoints (gateway endpoints for S3/DynamoDB are free) to avoid NAT Gateway processing charges

## Exam Cheat Sheet

| Requirement | Service |
| --- | --- |
| Alert when monthly spend is forecast to exceed $X | AWS Budgets (forecasted alert) |
| Automatically restrict permissions when budget exceeded | Budget Actions (IAM policy / SCP) |
| Detect unusual spikes automatically | Cost Anomaly Detection |
| Detailed line-item data for Athena/QuickSight | CUR 2.0 / Data Exports |
| Multi-cloud standardized billing data | FOCUS export |
| Visualize trends, RI/SP recommendations, forecasts | Cost Explorer |
| Org-wide prioritized savings recommendations | Cost Optimization Hub |
| Rightsizing based on utilization | Compute Optimizer |
| Re-bill customers/BUs with custom rates | Billing Conductor |
| Share unused RIs/SPs across accounts | Consolidated billing (sharing on by default) |
| Allocate shared EKS/ECS cluster cost | Split cost allocation data (CUR 2.0) |
| Estimate cost of a new architecture | Pricing Calculator |
