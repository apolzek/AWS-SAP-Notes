# Cost Allocation Tags

- Tags that we can enable to provide additional information for any billing report in AWS
- Cost Allocation Tags need to be **activated** in the Billing console (Cost allocation tags page) - in an Organization only the **management (payer) account** can activate them, and activation applies to all member accounts
- Tagging the resource is not enough: a resource tag only shows up in billing data after the tag key is activated as a cost allocation tag
- There 2 different form of Cost Allocation Tags:
  - AWS generated - example: `aws:createdBy` or `aws:cloudformation:stack-name`. These are added automatically by AWS if cost allocation tags are enabled
  - User defined tags: `user:something`
- Both type of tags will be visible in AWS cost reports and can be used as a filter
- Cost Allocation Tags appear only in the Billing Console
- After enabling Cost Allocation Tags, it can take up to 24 hours to be visible and active
- Cost Allocation Tags are not added retroactively by default - but since Mar 2024 the management account can request a **backfill** of up to **12 months** (only for tags that were actually on the resources at that time; one request per 24h)
- Tag keys are case sensitive (`CostCenter` != `costcenter`) - enforce consistency with **Tag Policies** (AWS Organizations) and prevent untagged resources with SCPs (`aws:RequestTag`) / AWS Config `required-tags`
- Not all resources/services support tags or cost allocation (e.g. some data transfer charges) - use Cost Categories / account structure for those

## Related Cost Allocation Features

- **Cost Categories**: rules that map costs to business dimensions (team, project, environment) based on accounts, tags, services, charge types - including **split charges** to allocate shared costs (e.g. shared networking split proportionally); usable in Cost Explorer, Budgets, CUR/Data Exports
- **Split cost allocation data** for **ECS and EKS**: allocates EC2 instance costs to individual tasks/pods (by CPU/memory reservation/usage) in CUR 2.0 - showback for shared container clusters
- **Account-based allocation**: a multi-account strategy (one account per workload/team/env) is the most reliable allocation method - tags complement it
- Tags appear in Cost Explorer, Budgets (filters), CUR 2.0 / Data Exports (`resource_tags` column), Cost Anomaly Detection monitors (by cost allocation tag)
- AWS-generated `aws:createdBy` tag must be activated and only applies to resources created after activation
- Exam tip: "chargeback/showback per department across a multi-account org" => activate cost allocation tags in the management account + Cost Categories + CUR 2.0 into S3/Athena/QuickSight; "costs from shared EKS clusters per team" => split cost allocation data
- See also [AWS Billing and Cost Management](billing.md)
