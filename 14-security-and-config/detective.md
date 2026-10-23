# Amazon Detective

- GuardDuty, Macie and Security Hub are used to identify potential security issues or findings
- Sometimes security findings require deeper analysis to isolate the root cause and take actions
- Amazon Detective analyzes, investigates and quickly identifies the root cause of security issues or suspicious activities using ML and graphs
- Automatically collects and processes events from VPC Flow Logs, CloudTrail trails and GuardDuty to create an unified view (no need to enable/store these logs ourselves - Detective consumes independent streams)
- Produces visualization with details and context to get to the root cause of the issue
- Core data sources: VPC Flow Logs, CloudTrail management events, GuardDuty findings. Optional source packages: EKS audit logs, AWS security findings (from Security Hub - Inspector, Macie, Config, third-party)
- Behaviour graph: up to 1 year of aggregated data, ML baselines, entity profiles (IAM users/roles, EC2, IP addresses, S3 buckets, EKS clusters/pods)
- **Finding groups**: correlates related findings into a single group (attack chain, MITRE ATT&CK tactics) with generative AI summaries
- **Investigations** for IAM users/roles: indicators of compromise (unusual geolocations, impossible travel, high-volume API calls, TTPs)
- Multi-account: administrator account (delegated admin via Organizations) + member accounts, single behaviour graph per region
- Integrations: pivot from GuardDuty/Security Hub findings into Detective; can query Amazon Security Lake logs (CloudTrail/VPC Flow) for raw-log retrieval
- Prerequisite: GuardDuty must be enabled (for at least 48 hours) before enabling Detective
- Exam: GuardDuty *detects*, Detective *investigates root cause*, Security Hub *aggregates/prioritizes*, Inspector *scans vulnerabilities*, Macie *finds sensitive data in S3* <span style="color: red;">EXAM</span>
