# AWS Audit Manager

> **Status (2026):** AWS Audit Manager is in **maintenance mode** - no longer open to new customers/accounts since April 30, 2026. Existing customers can keep running assessments but can't expand to new regions/organizations, and no new frameworks or features are added. AWS recommends **AWS Config conformance packs** (and Security Hub CSPM standards) for compliance management.

- AWS Audit Manager helps us continually audit our AWS usage to simplify how we manage risk and compliance with regulations and industry standards
- Audit Manager provides prebuilt frameworks that structure and automate assessments for a given compliance standard or regulation
- We can create an assessment from any framework
- Audit Manager automatically runs resource assessments. These assessments collect data for the AWS accounts that we define as in scope for your audit
- After we complete an audit and we no longer need Audit Manager to collect evidence, we can stop evidence collection
- Evidence types: automated (from AWS Config rules, Security Hub controls, CloudTrail, API calls/resource configuration snapshots) and manual evidence (uploaded documents)
- Prebuilt frameworks: CIS, PCI DSS, HIPAA, SOC 2, GDPR, NIST 800-53, FedRAMP, AWS Foundational Security Best Practices, AWS generative AI best practices, etc. Custom frameworks and controls can be created
- Evidence stored in an S3 assessment report destination; assessment reports are generated for auditors (with integrity checksums)
- Multi-account: integrates with AWS Organizations via a delegated administrator
- Audit Manager vs Config vs Security Hub CSPM vs Artifact:
    - **Audit Manager**: collect and organize *evidence* for auditors over time
    - **Config**: record configuration and evaluate rules/conformance packs
    - **Security Hub CSPM**: security posture checks against standards, findings aggregation
    - **AWS Artifact**: download AWS's own compliance reports (SOC, ISO, PCI) and agreements (BAA) <span style="color: red;">EXAM</span>
