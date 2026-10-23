# AWS Config

- AWS Config has 2 main jobs:
    - Primary: record configuration changes over time on AWS resources. Every time a configuration is changed on a resource a configuration item is created which stores the change at that specific point in time
    - Secondary: auditing of changes, compliance with standards
- Config does not prevent changes from happening! It is not a permissions product or a protections product. Even if we define standards for resources, Config can check the compliance against those standards, but it does not prevent us from braking those standards
- Config is a regional service, supports cross-region and cross-account aggregations
- Changes can generate SNS notifications and near-realtime events via EventBridge and Lambda
- Config stores changes historically in a consistent format in an S3 product bucket
- Config recording has to be manually enabled!
- Recording frequency: **continuous** (every change) or **periodic** (daily snapshot - cheaper, useful for frequently changing resources); recording can include all resource types or a selected list, and global resources (IAM) should be recorded in one region only to avoid duplicates
- Pricing: per configuration item recorded + per rule evaluation + per conformance pack evaluation
- Config is not real-time preventive: for prevention use SCPs/RCPs, IAM, CloudFormation Hooks, Control Tower proactive controls
- Config Rules:
    - Can be AWS managed ones or user defined using Lambda
    - Resources are evaluated against these rules determining if there are compliant or non-compliant
    - Custom Rules use Lambda, the function does the evaluation and returns the information back to Config
    - Custom policy rules: written in **CloudFormation Guard** DSL, no Lambda needed
    - Trigger types: configuration change (when a resource changes) or periodic (1, 3, 6, 12, 24 hours)
    - Evaluation modes: **detective** (deployed resources) and **proactive** (evaluate resource configuration *before* deployment - used by CloudFormation Hooks / Control Tower proactive controls)
    - Organization rules: deploy a rule to all accounts from management/delegated admin account
- Config can be integration to EventBridge which can be used to invoke Lambda functions for automatic remediation
- Config can also have integration with SSM to remediate issues
- AWS Config architecture:
    ![AWS Config architecture](images/AWSConfig.png)

## Multi-Account and Multi-Region Aggregation

- **Aggregator**: collects configuration and compliance data from multiple accounts and regions into one aggregator account (read-only view, no rule deployment)
    - Source = individual accounts (require authorization in each source account) or an entire AWS Organization (no per-account authorization) <span style="color: red;">EXAM</span>
- Advanced queries: SQL-like queries on current configuration state (single account or across an aggregator), also natural language query generator
- Configuration history and snapshots delivered to S3 (central bucket possible across accounts) + SNS topic
- Config + Security Hub CSPM: Security Hub controls are implemented as service-linked Config rules - Config recording must be enabled for Security Hub CSPM

## AWS Config Remediations

- Nom-compliant resources can be remediated automatically using System Manager Automation documents
- AWS Config provides a set of managed automation documents with remediation actions. We can create custom remediation documents
- Remediation can be manual or **automatic** with retries (max attempts within a time period); uses an IAM role passed to SSM Automation
- Example: `s3-bucket-public-read-prohibited` non-compliant -> `AWS-DisableS3BucketPublicReadWrite`; unrestricted SSH SG -> `AWS-DisablePublicAccessForSecurityGroup`
- Custom remediation via EventBridge rule on compliance change -> Lambda/Step Functions

## Conformance Packs

- A conformance pack is a collection of AWS Config rules and remediation actions that can be deployed as a single entity in an account/region or across an AWS Organization
- Conformance packs are created by authoring a YAML template that contains a list AWS Config rules and remediation actions
- Conformance packs can be deployed organization-wide (`PutOrganizationConformancePack`) from management or delegated admin account; AWS provides sample templates (Operational Best Practices for CIS, PCI DSS, HIPAA, NIST, etc.)
- Compliance score per conformance pack; recommended replacement for AWS Audit Manager (maintenance since Apr 2026)
- Config vs CloudTrail: Config = *what* the resource looks like and how it changed (configuration timeline, relationships); CloudTrail = *who* made the API call and when <span style="color: red;">EXAM</span>
