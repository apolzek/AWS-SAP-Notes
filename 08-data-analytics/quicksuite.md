# Amazon Quick Suite (formerly QuickSight)

> **Status (2026):** Amazon QuickSight became **Amazon Quick Sight**, the BI capability of **Amazon Quick Suite** (GA Oct 9, 2025). Existing QuickSight accounts, dashboards and APIs (`quicksight`) keep working. Exam questions still say "QuickSight" => same BI service

- Quick Suite extends QuickSight with AI-based research, collaboration, and more
- It is a business analytics and intelligence (BA/BI) service
- It is used for visualization and ad-hoc analysis
- Cost-effective, on-demand service, billed per session
  - Today: per-user pricing (Reader, Author, Author Pro / Reader Pro with generative AI) or **capacity pricing** (bulk reader sessions) for large/anonymous audiences
- It is able to discover and integrate with AWS data sources and supports a wide range of external data sources
- Supported data sources:
  - Athena, Aurora, Redshift, Redshift Spectrum
  - S3, AWS Iot
  - Jira, GitHub, Twitter, SalesForce
  - Microsoft SQL Server, MySQL, PostgreSQL
  - Apache Spark, Snowflake, Presto, Teradata
  - Also: OpenSearch, Databricks, Timestream, files (CSV/Excel) upload

## Quick Suite Components

- **Quick Sight**: BI dashboards, natural-language Q&A (formerly Amazon Q in QuickSight / QuickSight Q), generative BI, what-if analysis
- **Quick Research**: AI research agent combining internal data and public sources into reports
- **Quick Flows**: no-code automation of routine tasks for business users
- **Quick Automate**: multi-step enterprise workflows with monitoring and human-in-the-loop
- **Quick Index**: unified searchable knowledge base over documents and app data; **Spaces** and **chat agents** on top

## Quick Sight Key Concepts (exam)

- **SPICE** (Super-fast, Parallel, In-memory Calculation Engine): data imported into in-memory engine => fast dashboards, no load on the source; scheduled or incremental refresh. Capacity bought per GB
- Direct query mode: queries the source live (no SPICE)
- Editions: Standard vs **Enterprise** (Enterprise needed for: Active Directory/IAM Identity Center, **row-level security** (RLS), column-level security, VPC connections, encryption with KMS, embedding, hourly refresh)
- **Private data sources**: use a **VPC connection** (ENIs in our VPC) to reach RDS/Redshift in private subnets; for on-prem via VPN/Direct Connect
- Security: users/groups (IAM Identity Center, SAML, AD), RLS via dataset rules or tags, **Lake Formation** permissions honored with Athena
- **Embedded analytics**: embed dashboards in apps (registered users or anonymous users with capacity pricing)
- Paginated reports, alerts, anomaly detection, forecasting (ML insights)
- Cross-account access to S3/Athena via IAM role permissions on the Quick Sight service role
- Exam: "serverless BI dashboards on Athena/Redshift data with per-user security" => Quick Sight (Enterprise + RLS)
