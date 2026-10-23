# AWS GuardDuty

- It is a security service used for continuous monitoring of an account
- It can be integrated with supported data sources, constantly monitoring them
- It uses AI/ML and threat intelligence feeds for monitoring for suspicious activities (plus custom trusted IP lists and threat lists we upload)
- Identifies any unexpected and unauthorized activity and it tries to spot odd activities
- If it finds something, it can be configured to notify us or to do event-driven protection/remediation
- Supports multiple accounts (Master and Member accounts) - now called **administrator** and member accounts; with AWS Organizations use a **delegated administrator** and auto-enable for new accounts <span style="color: red;">EXAM</span>
- GuardDuty architecture:
    ![GuardDuty architecture](images/AmazonGuardDuty.png)

## Data Sources and Protection Plans

- **Foundational** (always on, no need to enable the logs ourselves - GuardDuty reads independent streams):
    - CloudTrail management events
    - VPC Flow Logs (EC2 ENIs)
    - Route 53 Resolver DNS query logs
- Optional **protection plans** (per account/region, enabled individually):
    - **S3 Protection**: CloudTrail S3 data events - suspicious access, exfiltration, disabled logging/public access
    - **EKS Protection**: EKS audit logs
    - **Runtime Monitoring**: OS-level events (process, file, network) via a GuardDuty security agent on EKS, ECS (Fargate and EC2) and EC2 - agent can be managed automatically (EKS add-on, Fargate sidecar, SSM for EC2)
    - **Malware Protection for EC2**: agentless scan of EBS volumes (snapshot) when a suspicious finding occurs, or on-demand
    - **Malware Protection for S3** (June 2024): scans new objects uploaded to selected buckets, can tag objects (e.g. `NO_THREATS_FOUND`) - use a bucket policy/tag-based access to quarantine. Can be enabled standalone (without full GuardDuty) <span style="color: red;">EXAM</span>
    - **Malware Protection for AWS Backup** (Nov 2025): scans EC2/EBS/S3 recovery points (incremental scans) - verify clean restore points before recovery from ransomware
    - **RDS Protection**: anomalous login activity on Aurora/RDS (MySQL, PostgreSQL)
    - **Lambda Protection**: network activity logs of Lambda functions (e.g. crypto-mining, C2 communication)
- 30-day free trial per plan; pricing based on volume of events/logs analysed

## Findings

- Format: `ThreatPurpose:ResourceTypeAffected/ThreatFamilyName.DetectionMechanism!Artifact` e.g. `CryptoCurrency:EC2/BitcoinTool.B!DNS`, `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS`
- Severity: Low / Medium / High / Critical
- **Extended Threat Detection** (Dec 2024, on by default, no extra cost): correlates multiple events across data sources and time into **attack sequence** findings (Critical) mapped to MITRE ATT&CK - e.g. `AttackSequence:IAM/CompromisedCredentials`, `AttackSequence:S3/CompromisedData`, EKS clusters (Jun 2025), EC2 instance groups and ECS clusters (Dec 2025)
- Findings go to EventBridge (near real-time, ~5 min for new findings; updates every 6 hours by default, configurable 15 min/1 h/6 h), Security Hub, Detective; export to S3 (KMS encrypted)
- Suppression rules: auto-archive expected findings (e.g. from a vulnerability scanner)
- Cannot be "disabled" by a member - administrator controls it; suspending GuardDuty stops analysis, disabling deletes findings

## Automated Response (Exam patterns)

- GuardDuty finding -> EventBridge rule -> Lambda / Step Functions / SSM Automation:
    - Isolate EC2 (swap SG to a quarantine SG, snapshot EBS for forensics)
    - Block malicious IP in WAF IP set or Network Firewall rule group / NACL
    - Revoke IAM session (`aws:TokenIssueTime` deny policy), disable access keys
    - Notify via SNS / chat
- Centralize: delegated admin + Security Hub aggregation across regions
- GuardDuty is regional - enable in **all regions** (even unused ones) to detect activity from stolen credentials <span style="color: red;">EXAM</span>
