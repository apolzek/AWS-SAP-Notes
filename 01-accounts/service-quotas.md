# AWS Service Quotas

- Defines how much of a "thing" we can use inside of an AWS account
- Example:
    - Number of EC2 instances at a certain times per region
    - Number of IAM users per AWS accounts
- Services usually have a default per region quota
- Global services may have a per account quota instead per region
    - Global services quotas (IAM, Organizations, CloudFront, Route 53) are viewed/requested in `us-east-1`
- Most services quotas can be increased as needed
- Some service quotes can not be changed, example: number of IAM users per account (5000)
- The higher the increase, the more time is needed to get the change-request approved
    - Some increases are auto-approved within minutes (e.g. IAM roles up to 10,000, customer managed policies up to 10,000)
- Service endpoint and quotas: [https://docs.aws.amazon.com/general/latest/gr/aws-service-information.html](https://docs.aws.amazon.com/general/latest/gr/aws-service-information.html)
- **Service Quotas**: 
    - From the console we can go to *Service Quotas* page, where we can create dashboards for quotas we want to monitor
    - We can request quota changes from this service for certain services
    - *Quota request template*: we can predefine quota value request for new accounts in an AWS organization
        - Configured from the management account (Organizations trusted access required); requests are submitted automatically when **new accounts are created** in the org (not applied to existing accounts)
    - We can create a CloudWatch Alarm based on a particular service quota
- Legacy method to increase quotas: create a support ticket selecting service quota increase
- We can request service quota increase from the CLI as well. Reference API: [https://awscli.amazonaws.com/v2/documentation/api/latest/reference/service-quotas/request-service-quota-increase.html](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/service-quotas/request-service-quota-increase.html)

## Quota types

- **Applied quota**: current value for our account (after increases); **AWS default quota**: baseline (`GetAWSDefaultServiceQuota`)
- **Adjustable** vs **non-adjustable (hard limits)**: exam trap - if the quota is hard, the answer is architectural (more accounts, federation instead of IAM users, sharding, etc.)
- **Resource quotas** (e.g. number of VPCs) vs **rate quotas / API throttling** (e.g. STS 600 RPS per account per region, EC2 API token bucket). Throttling errors -> retry with exponential backoff + jitter, caching, request increase
- Some quotas are per account **per region**, multi-region DR designs must raise quotas in the DR region **in advance** (pilot light / warm standby need capacity when failover happens)

## Monitoring quotas

- **CloudWatch usage metrics** (`AWS/Usage` namespace) for supported quotas; Service Quotas console can create CloudWatch alarms at a % of the quota
- **Service Quotas Automatic Management**: monitors utilization and notifies at 80% / 95% (email, Slack, console mobile app, AWS Health, EventBridge)
    - *Notify Only* mode or *Notify and Auto-Adjust* mode (automatically submits increase requests for adjustable quotas)
- **Trusted Advisor** "Service Limits" checks (full set of checks requires Business/Enterprise support); can be aggregated org-wide via the Trusted Advisor organizational view
- Automate with EventBridge: quota-threshold event -> Lambda -> `RequestServiceQuotaIncrease`

## Multi-account considerations

- Quotas are per account: one of the reasons for a **multi-account strategy** is to isolate quota consumption (noisy neighbor) between workloads
- AWS Organizations itself has quotas: 10 accounts by default (adjustable), 5 OU levels, 2,000 OUs, max 10 SCPs per entity
- Some services have org-level limits independent of the org account count (e.g. Control Tower 10,000 accounts, IAM Identity Center 7,000 accounts per instance by default)
