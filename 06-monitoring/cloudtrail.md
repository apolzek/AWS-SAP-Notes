# CloudTrail

- Is a product which logs API actions which affects AWS accounts (example: stop EC2 instance, delete S3 bucket)
- It logs API calls/activities as a CloudTrail Event, actions taken by user, role or service
- CloudTrail by default logs the last 90 days in Event History. It is enabled by default at no cost
- Trails: used to customize CloudTrail history
- Trails (and CloudTrail Lake) can log 4 types of events:
  - **Management Events**: provide information about management operations performed on resources in AWS accounts (control plane operation). Example: create an EC2 instance, terminate an EC2 instance
  - **Data Events**: contain information about resource operations performed on or in a resource, example: objects uploaded to S3, Lambda function being invoked
  - **Network Activity Events** (GA 2025): API calls made **through VPC endpoints** from a private VPC to AWS services (e.g. S3, KMS, EC2, Secrets Manager, CloudTrail) - including calls **denied by the VPC endpoint policy** => detect data exfiltration attempts / access from outside the org (network perimeter)
  - **Insights Events**: CloudTrail Insights analyzes our normal patterns of API call volume and API error rates, also called the _baseline_, and generates Insights events when the call volume or error rates are outside normal patterns
    - Originally only for write management events; since Nov 2025 also for **data events**; baseline takes up to 36h; extra cost
- By default CloudTrail only logs **management events** (data events are way higher volume)
  - Management events can be split into read/write; exclude noisy sources (KMS, RDS Data API events)
  - Data events and network activity events must be explicitly enabled and are charged (~$0.10 per 100,000 events)
  - **Advanced event selectors**: fine-grained filtering (by `eventName`, `resources.ARN`, `readOnly`, `userIdentity.arn`, etc.) - reduce cost by logging only the data events that matter
  - **Aggregated events** (Nov 2025): 5-minute summaries of high-volume data events (top actions, error rates, access frequency) delivered to the trail bucket - cheaper downstream analysis
- CloudTrail is a regional service, but when we create a Trail, it can configured to operate in two ways:
  - One region: only will log events from the region in which it is created
  - All regions: collection of trails from all regions
- A one region trail can be configured to log global services events as well
- New trails created in the console are always **multi-region** (single-region trails only via CLI/API); multi-region trails also capture new regions automatically - best practice for security
- Most services log events in the region where the event occurred, but a small number of services (IAM, STS, CloudFront) log events globally (us-east-1). **A trail needs to be enabled in that region to capture those events!**
- The default event history is limited to 90 days, with a trail we can be much more flexible
- A trail can store the events in a defined S3 buckets indefinitely, and these logs can be parsed by other tooling (log entries are stored in JSON format)
- CloudTrail can be integrated with CloudWatch Logs, where we can use Metric Filters for example (e.g. alarm on root login, `ConsoleLogin` failures, SG/NACL/IAM policy changes - CIS benchmark alarms)
- **EventBridge** receives CloudTrail management events (and data events if logged) - react to API calls in near real time (e.g. Lambda auto-remediation when someone makes a bucket public)
- Query trail logs in S3 with **Athena** (partition projection) - the cheap/default way for forensic analysis
- Security of the trail:
  - **Log file integrity validation**: hourly digest files (SHA-256 hashes + signature) - prove logs weren't modified/deleted (`validate-logs`)
  - Encrypt with **SSE-KMS** (default SSE-S3), S3 bucket in a dedicated log archive account, **S3 Object Lock** / MFA delete, bucket policy denying deletes
  - SNS notification on each log file delivery (optional)
- Event History: management events only, 90 days, per region, can be searched by attributes (not data events)

![CloudTrail](images/CloudTrail.png)

- Organizational Trail:
  - A trail created in the management account of an organization storing all events across every account in the organization
  - In case we create a trail from the management account of an organization, we can enable it to bring events from all of the accounts from the organization
  - Can also be created/managed by a **delegated administrator** account (up to 3)
  - Member accounts can **see** the org trail but **cannot modify or delete** it - log path: `s3://bucket/AWSLogs/o-orgid/account-id/...`
  - New accounts joining the org are logged automatically; Control Tower sets up an org trail to the Log Archive account
- CloudTrail does not offer real time logging => log files are delivered on average within ~5 minutes of the API call (not guaranteed; older material says up to 15 min)
- CloudTrail pricing:
  - 90 days history enabled by default in every AWS account and free
  - One copy of management events is free for every region in an AWS account
  - Other trails created are charged
  - Data events, network activity events and Insights events are charged per event analyzed/delivered

## CloudTrail Lake

> **Status (2026):** CloudTrail Lake is **closed to new customers since May 31, 2026**. Existing customers can keep using it (only critical bug fixes/security updates). Trails, Insights and aggregated events are not affected. AWS recommends CloudWatch (CloudWatch Logs + Logs Insights / centralization) as the alternative.

- Managed data lake for CloudTrail events: **event data stores** (immutable, default 7 years retention, up to ~10 years), queried with **SQL**
- Can ingest CloudTrail events, AWS Config configuration items, Audit Manager evidence and non-AWS events (integrations)
- Organization event data stores; federation to Glue Data Catalog / Athena; dashboards
- Exam (older questions): "SQL queries over 7 years of API activity without building an Athena pipeline" => CloudTrail Lake; today prefer trail + S3 + Athena, or CloudWatch Logs

## CloudTrail Exam Tips

- "Who deleted the resource?" => CloudTrail (Event History if < 90 days, otherwise trail in S3 / Athena)
- "Detect API calls in real time and remediate" => EventBridge rule on CloudTrail events + Lambda/SSM Automation
- "Detect unusual spikes of API activity" => CloudTrail Insights (or GuardDuty for threat detection)
- "Prove logs were not tampered with" => log file integrity validation + Object Lock
- "Log S3 object-level access" => S3 data events (or S3 server access logs - cheaper, best effort)
- "Detect access via VPC endpoints from outside the organization" => network activity events
- CloudTrail is not a performance tool (CloudWatch) and not a configuration history tool (AWS Config)
