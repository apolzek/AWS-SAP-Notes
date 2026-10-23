# Amazon Macie

- It is a data security and data privacy service
- Macie is a service to discover, monitor and protect data stored in S3 buckets
- Once enabled and pointed to buckets, Macie will automatically discover data and categorize it as PII, PHI, Finance etc.
- Macie is using data identifier. There are 2 types of data identifier:
  - Managed Data Identifier: built-in, can use machine learning, pattern matching to analyze and discover data. It is designed to detect sensitive data from many countries
  - Custom Data Identifier: created by clients, they are proprietary to accounts and they are regex based
- Discovery Jobs: these jobs will use data identifiers to manage and search for sensitive content. They will generate findings which can be used for integration with other AWS services in order to do automatic remediation
  - Findings are published to **Amazon EventBridge** (directly) and optionally to **AWS Security Hub**; EventBridge => Lambda/Step Functions/SNS for remediation
  - Findings are retained in Macie for 90 days
- Macie uses multi account architecture: one account is the administrator account which can used to manage Macie within the member accounts to discover sensitive data
- This multi-account structure can be done with AWS Organizations (recommended, **delegated administrator** account, auto-enable for new member accounts) or by explicitly inviting accounts in Macie (max 1,000 accounts via invitation)
- Macie is a **regional** service => must be enabled in every region where S3 data lives
- Scope: **S3 general purpose buckets only** (not EBS, RDS, DynamoDB, EFS); analyzes many file formats (text, CSV/JSON, Parquet/Avro, PDF, Office docs, compressed archives)
- Pricing: per bucket monitored (inventory/evaluation) + per GB of data inspected => use sampling / automated discovery to control cost

![Macie](images/Macie.png)

## Automated Sensitive Data Discovery

- Continuous, sampling-based analysis of all S3 buckets in the account/organization (enabled by default for new Macie accounts, can exclude buckets/accounts)
- Daily selection of representative objects => cost-effective, broad visibility without full scans
- Produces a **sensitivity score** per bucket and an interactive data map/heatmap
- Typical pattern: automated discovery to find "hot spots", then targeted discovery jobs for full analysis of those buckets
- Exam: "continuously and cost-effectively identify which buckets in the org contain PII" => Macie automated sensitive data discovery

## S3 Security Posture Monitoring

- Macie maintains an inventory of all S3 buckets: public access, encryption, shared/replicated with external accounts, size and object counts
- Generates **policy findings** automatically (no job needed) when bucket security decreases

## Macie Identifiers

- **Data Discovery Jobs**: analyzes data in order to determine wether the objects contain sensitive data. This is done using data identifiers
  - One-time or scheduled (daily/weekly/monthly), can be scoped by bucket, prefix, tags, object criteria, sampling depth
  - Detailed results (sensitive data discovery results) are written to an S3 bucket we configure, encrypted with a KMS key - required for long-term audit
- **Managed Data Identifiers**:
  - Created and managed by AWS
  - Can be used to detect a growing list of common sensitive data types: credentials, financial data, health data, personal identifiers (addresses, passport nb, etc.)
- **Custom Data Identifiers**:
  - Can be created by us, AWS account users/owners
  - Regex: define patterns to match in data
  - Keywords: optional sequences that need to be in proximity to regex match
  - Maximum Match Distance: how close keywords are to regex pattern
  - Ignore Words: We can also include ignore words in the regex match
- **Allow Lists**: text or regex that Macie must ignore (e.g. test credit card numbers, public company addresses) => reduce false positives

## Macie Findings

- Macie will produce 2 types of findings:

  - **Policy Findings**: are generated when the policies or settings are changed in a way that reduces the security of the bucket after Macie is enabled
  - **Sensitive Data Findings**: generated when sensitive data is identified based on identifiers

- Types of policy findings:
  - `Policy:IAMUser/S3BlockPublicAccessDisabled`: all bucket-level block public access settings were disabled for the bucket
  - `Policy:IAMUser/S3BucketEncryptionDisabled`: default encryption settings for the bucket were reset to default Amazon S3 encryption behavior, which is to encrypt new objects automatically with an Amazon S3 managed key
  - `Policy:IAMUser/S3BucketPublic`: an ACL or bucket policy for the bucket was changed to allow access by anonymous users or all authenticated AWS Identity and Access Management (IAM) identities
  - `Policy:IAMUser/S3BucketSharedExternally`: an ACL or bucket policy for the bucket was changed to allow the bucket to be shared with an AWS account that's not part of your organization
  - `Policy:IAMUser/S3BucketReplicatedExternally`: replication was enabled and configured to replicate objects to a bucket in an AWS account outside the organization
  - `Policy:IAMUser/S3BucketSharedWithCloudFront`: the bucket policy allows sharing with a CloudFront OAI/OAC
- Types of sensitive data findings:
  - `SensitiveData:S3Object/Credentials`: object contains sensitive credentials data, such as AWS secret access keys or private keys
  - `SensitiveData:S3Object/CustomIdentifier`: object contains text that matches the detection criteria of one or more custom data identifiers
  - `SensitiveData:S3Object/Financial`: object contains sensitive financial information, such as bank account numbers or credit card numbers
  - `SensitiveData:S3Object/Multiple`: object contains more than one category of sensitive data
  - `SensitiveData:S3Object/Personal`: object contains sensitive personal information—personally identifiable information (PII) such as passport numbers or driver's license identification numbers, personal health information (PHI) such as health insurance or medical identification numbers, or a combination of PII and PHI
- Severity: Low / Medium / High, based on type and amount of sensitive data
- Macie can retrieve and reveal **samples** of sensitive data found in an object (requires configuration + KMS key) to help investigate findings

## Macie vs Other Services

- Macie: sensitive data (PII/PHI/credentials) **inside S3 objects** + S3 bucket posture
- GuardDuty (S3 Protection): threat detection on S3 API activity (anomalous access, exfiltration) and malware protection for S3 uploads
- Security Hub: aggregates findings from Macie, GuardDuty, Inspector, etc.
- Comprehend: PII detection in text passed to its API (not a storage scanner)
- Glue / DataBrew / Bedrock Guardrails: PII detection/redaction in data pipelines or GenAI prompts
