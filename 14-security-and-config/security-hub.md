# AWS Security Hub

> **Status (2026):** In 2025 the original service was renamed **AWS Security Hub CSPM** (cloud security posture management - security standards/controls, ASFF findings). A **new AWS Security Hub** (preview re:Inforce Jun 2025, GA Dec 2025) sits on top: unified security operations that correlates GuardDuty, Inspector, Macie and Security Hub CSPM signals into exposure findings, using **OCSF**. The notes below (central configuration, standards) apply to Security Hub CSPM unless stated otherwise.

- Provides a comprehensive view of our security state in AWS
- Helps us assess our AWS environment against security industry standards and best practices
- Security Hub collects security data across AWS accounts, AWS services, and supported third-party products and helps us analyze our security trends and identify the highest priority security issues
- Security Hub needs to be enabled to be used. There are two ways to enable AWS Security Hub, by integrating with AWS Organizations or manually
- Organization can have a delegated administrator for the Security Hub

## Central Configuration

- It is a feature that helps us set up and manage Security Hub across multiple AWS accounts and regions
- We must integrate Security Hub and Organizations to use central configuration. Also, we must designate a home Region to use central configuration. The home Region is the Region from which the delegated administrator configures the organization
- The delegated administrator account can create AWS Security Hub configuration policies to configure Security Hub, security standards, and security controls in your organization
- Policy configurations:
    - Configuration policies must be associated to take effect
    - An account or OU can be associated with only one configuration policy
    - Configuration policies are complete: Configuration policies provide a complete specification of settings
    - Configuration policies can't be reverted: There's no option to revert a configuration policy after you associate it with accounts or OUs
    - Configuration policies take effect in your home Region and all linked Regions
    - Configuration policies are resources: they have an ARN
- Types of configuration polices:
    - Recommended configuration policy - provided by AWS
    - Custom configuration policy

## Security Hub CSPM - Standards, Controls and Findings

- Requires **AWS Config** recording enabled in each account/region (controls are service-linked Config rules) <span style="color: red;">EXAM</span>
- Security standards: AWS Foundational Security Best Practices (FSBP), CIS AWS Foundations Benchmark, PCI DSS, NIST SP 800-53, AWS Resource Tagging Standard, Service-managed standard (Control Tower)
- Findings in **ASFF** (AWS Security Finding Format) from: Config, GuardDuty, Inspector, Macie, IAM Access Analyzer, Firewall Manager, Systems Manager Patch Manager, Health, third-party (Partner products) and custom (`BatchImportFindings`)
- **Cross-region aggregation**: link regions to an aggregation (home) region to see all findings in one place; with Organizations delegated admin -> single pane across accounts and regions
- **Automation rules**: automatically update/suppress findings (severity, workflow status, notes) based on criteria
- Response: findings -> **EventBridge** (all findings automatically, or **custom actions** for manual "send to remediation") -> Lambda / SSM Automation / Step Functions. AWS solution: *Automated Security Response on AWS* <span style="color: red;">EXAM</span>
- Insights: grouped findings (e.g. resources with most findings); security score per standard
- Pricing: per security check + per finding ingested

## New Security Hub (2025+)

- Builds on Security Hub CSPM + GuardDuty + Inspector + Macie (each still enabled separately)
- **Exposure findings**: correlates signals - e.g. internet-reachable EC2 + critical CVE + IAM role with access to sensitive S3 data -> one prioritized exposure finding
- **Attack path** graph visualizing how an attacker could chain vulnerabilities/misconfigurations to reach critical resources
- Findings normalized in **OCSF** (Open Cybersecurity Schema Framework - same schema as Security Lake); ASFF and OCSF are not compatible
- Threat trends dashboards, resource inventory, automated response and ticketing integrations (Jira, ServiceNow)

## Related Services

- **Amazon Security Lake**: centralizes security logs (CloudTrail, VPC Flow Logs, Route 53 Resolver, Security Hub findings, EKS audit, WAF, third-party) into an S3-based data lake in **OCSF** / Apache Iceberg format across accounts and regions; subscribers query with Athena/OpenSearch or SIEM tools (data access or query access via Lake Formation) <span style="color: red;">EXAM</span>
- **IAM Access Analyzer**: external access findings (resources shared outside the zone of trust - account/org), **internal access** findings, **unused access** findings (unused roles, keys, permissions - paid), policy validation, custom policy checks, policy generation from CloudTrail activity
- **Amazon Macie**: discovers sensitive data (PII, credentials) in S3 using managed/custom data identifiers; automated sensitive data discovery sampling; bucket security posture (public, unencrypted, shared)
- **AWS Verified Access**: VPN-less, zero-trust access to internal HTTP(S) (and TCP since 2025) applications; evaluates each request with identity (IAM Identity Center/OIDC) and device posture (CrowdStrike, Jamf, JumpCloud) using Cedar policies; WAF can be attached
- Exam mapping: aggregate/prioritize findings + compliance checks -> Security Hub; centralize raw logs long-term in open format -> Security Lake; investigate -> Detective
