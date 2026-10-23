# AWS Organizations

- Standard AWS account: it is an account which is not in an AWS Organization
- We create an AWS Organization from a standard AWS account
- The organization is not created in this account, we just use the account to create the organization. The standard account then becomes the **Management Account** (used to be called *Master Account*)
- Using the Management Account we can invite other accounts into the organization
- When a standard account joins an organization, it will change to **Member Account** of that organization
- Organizations have 1 Management Account and 0 or more Member Accounts
- An account can be member of only **one** organization at a time; the management account can not be changed (to change it, delete the org or migrate accounts to a new org)
- We can create a structure of AWS accounts in an organization. We can group accounts by things such as business units, function or development stage, etc.
- This structure is hierarchical, it is an inverted tree
- At the top of this tree is the root container of the organization (just a container within the organization, NOT to be confused with the root user)
- This root container can contain other containers, this containers are known as **Organizational Units (OU)**
- OUs can contains accounts (Management/Member accounts) or other OUs
- Organizations is a global service hosted in `us-east-1`

## Feature sets

- **Consolidated billing only**: just billing features, no policies
- **All features** (default and recommended): consolidated billing + SCPs, RCPs, declarative/management policies, trusted access, delegated admin
    - Moving from consolidated billing to all features requires every **invited** member account to approve (accounts created by the org approve automatically)

## Key quotas

- Accounts: 10 by default (adjustable, up to tens of thousands on request); invitations count against it
- OUs: 2,000 per org; nesting: **5 levels** of OUs under the root
- Concurrent account creations: 5; new created account must exist 4 days (min) before it can be removed
- Account closures: max 20% of member accounts (min 250, max 1,000) per 30 days; closed accounts keep counting against the quota until permanently closed (90-day post-closure period)
- Invitation handshake expires after 15 days

## Consolidated Billing

- It is an important feature of AWS Organizations
- The individual billing method of each account from the organization is removed, the member accounts pass their billing through the Management Account (**Payer Account**)
- Using consolidated billing we get a single monthly bill. This covers the Management Account and all the Member Accounts of the Organization
- When using organization reservation benefits and discounts are pooled, meaning the organization can benefit as a whole for the spending of each AWS account within the org
    - Volume pricing tiers (e.g. S3, data transfer) are aggregated across accounts
    - RI and Savings Plans sharing is on by default; can be turned off per account (billing preferences) from the management account
- **Billing transfer**: a management account can transfer billing responsibility for its organization to another organization's management account (e.g. resellers/partners), while keeping governance in its own org

## Best Practices

- Have a single account into which users can log into and assume IAM roles in order to access other accounts from the org
- The account with all the identities may be the Management Account or it can be another Member Account (*Login Account*)
- Modern recommendation: use **IAM Identity Center** (delegated admin in a member account) instead of a hand-made login account
- Keep the management account **empty of workloads** - SCPs and RCPs do not apply to it, so it must be tightly controlled
- Use delegated administrators for security services (GuardDuty, Security Hub, Config, Access Analyzer, IAM Identity Center, CloudFormation StackSets, IAM root access management) so they run outside the management account
- Typical OU structure: Security (Log Archive, Audit/Security Tooling), Infrastructure (Network, Shared Services), Workloads (Prod/Non-Prod), Sandbox, Policy Staging, Suspended

## `OrganizationAccountAccessRole`

- This is an IAM role used to access the newly added/created account in an organization
- This role will be created automatically if we create the account from an existing organization
- This role has to be created manually in the member account if the account was invited into the organization
- It grants `AdministratorAccess` and trusts the management account - restrict who in the management account can assume it

## Trusted access and delegated administrator

- **Trusted access**: allows an AWS service to operate across the org (creates service-linked roles in member accounts). Enable it from the service console or `EnableAWSServiceAccess`
- **Delegated administrator**: register a member account to administer a service for the whole org (`RegisterDelegatedAdministrator`)
- Organizations itself supports a **resource-based delegation policy** so a member account can manage Organizations policies (SCPs, tag policies, etc.) or view org structure without using the management account

## Moving accounts between organizations

- Remove the account from the old org (`LeaveOrganization` / `RemoveAccountFromOrganization`) and accept the invite from the new one
- Account needs standalone billing information (payment method) to leave; accounts created in the org must exist 4+ days (older docs: 7) before removal
- Plan for: SCPs/RCPs no longer applied, RAM shares to the old org are lost, RIs/Savings Plans sharing stops, Control Tower enrollment

## Root access management (centralized root)

- Feature of IAM + Organizations (launched Nov 2024), enabled from the management account or the IAM delegated admin
- **Root credentials management**: delete root password, access keys, signing certificates and MFA of **member accounts**; accounts created afterwards have **no root credentials** by default
- **Privileged root actions**: perform root-only tasks on a member account with short-lived root sessions via `sts:AssumeRoot` (max 15 minutes), scoped by AWS managed task policies:
    - `IAMAuditRootUserCredentials`, `IAMCreateRootUserPassword` (allow password recovery), `IAMDeleteRootUserCredentials`
    - `S3UnlockBucketPolicy` (remove a bucket policy that denies everyone), `SQSUnlockQueuePolicy`
- Removes the need to manage root MFA devices for hundreds of member accounts - common exam answer for "secure root at scale"

# Service Control Policies (SCP)

- They are a feature of AWS Organizations used to restrict AWS accounts
- They are JSON documents
- They can be attached to the root of the organization, to one or more OUs or to individual AWS accounts
- SCPs inherit down through the organization tree
- The Management Account is special: even if it has SCPs attached (directly or through an OU) it wont be affected by the SCP
- SCPs are account permission boundaries:
    - They limit what the account (including the root user of the account) can do
    - We can never restrict a root user from an account, but we can restrict the account itself, hence these restrictions will apply to the root user as well
- **SCPs don't grant any permissions!** This are just a boundary to limit what is and is not allowed in an account
- SCPs apply only to IAM principals (users, roles, root) of **member accounts**. They do NOT affect:
    - The management account
    - **Service-linked roles**
    - Principals from **other accounts** accessing our resources (use RCPs for that)
- Limits: max **10 SCPs** directly attached per root/OU/account (was 5), max size **10,240 characters** (was 5,120), at least 1 SCP attached to each entity
- **Full IAM policy language** (2025): `Condition`, `NotAction`, `NotResource` can now be used in `Allow` statements too (previously only in `Deny`). Still unsupported: `Principal` / `NotPrincipal` (use `aws:PrincipalArn` condition instead)
- SCPs can be used in two ways:
    - Deny list (default): allow by default and block access to certain services
        - `FullAWSAccess`: policy applied by default to the org an all OUs when we enable SCPs. This policy means tha by default nothing is restricted
        - SCPs don't grant permissions, but when they are enabled, there is a default deny for everything. This is why the `FullAWSAccess` policy is needed
        - SCP priority rules:
            1. Explicit Deny
            2. Allow
            3. Default (implicit) deny
        - Benefits of deny lists is that as AWS is extends the list of service offerings, new services will be available for accounts (low admin overhead)
    - Allow list: block by default and allow certain services
        - To implement allow lists:
            1. Remove the `FullAWSAccess` policy
            2. Add any services which should be allowed in a new policy
        - Allow lists are more secure, but they require more admin overhead
        - Allow must exist at **every level** of the path (root -> OU -> account); a deny at any level wins
- Common SCP use cases (exam):
    - Deny regions outside an approved list (`aws:RequestedRegion` + `NotAction` for global services such as IAM, CloudFront, Route 53, Support)
    - Prevent leaving the organization (`organizations:LeaveOrganization`)
    - Prevent disabling CloudTrail, GuardDuty, Config, Security Hub
    - Protect a security/break-glass role from modification (exempt with `aws:PrincipalArn`)
    - Require IMDSv2, deny unencrypted EBS/S3, restrict instance types
- SCPs **do not** support "effective policy" view; test in a Policy Staging OU first, check CloudTrail for `AccessDenied`

# Resource Control Policies (RCP)

- Launched Nov 2024. Authorization policy type which sets the **maximum permissions on resources** in member accounts, regardless of who calls (including principals **outside** the org)
- SCP = boundary on **principals** of our org; RCP = boundary on **resources** of our org. Together they form a **data perimeter**
- Like SCPs: never grant permissions, don't apply to the management account, don't apply to service-linked roles, don't apply to AWS managed KMS keys
- `RCPFullAWSAccess` is attached automatically to every entity and can not be detached; max **5 RCPs** per entity (including `RCPFullAWSAccess`), max **5,120 characters**
- Supported services (growing list): S3, STS (role trust), KMS, SQS, Secrets Manager, DynamoDB, ECR, Cognito, CloudWatch Logs, OpenSearch Serverless, CloudFront, EventBridge, Transfer Family and many more - check docs for the current list
- Typical use cases:
    - Only identities from our org can access our S3 buckets/KMS keys (`aws:PrincipalOrgID` with exception for AWS service principals `aws:PrincipalIsAWSService`)
    - Enforce HTTPS (`aws:SecureTransport`) or minimum TLS on all S3 buckets org-wide
    - Prevent confused deputy for AWS services (`aws:SourceOrgID`)
    - Restrict which external accounts can assume our roles (STS)
- Exam trap: a developer writes a bucket policy allowing `Principal: "*"` - an RCP denying non-org principals still blocks it (SCP could not, because the caller is outside our org)

# Declarative policies (EC2 policies)

- Launched Dec 2024 (first type: EC2 policies; Organizations now groups tag, backup, chat, AI opt-out, etc. under the "declarative" category too). Enforce a **desired service configuration** at the service control plane (not at API authorization), maintained even when the service adds new APIs
- Apply to the management account too, also govern service-linked roles; support custom error messages and an **account status report**
- EC2 declarative policy attributes: VPC Block Public Access, serial console access, image (AMI) block public access, **Allowed AMIs**, IMDS defaults (IMDSv2), snapshot block public access
- Detaching a declarative policy rolls the attribute back to its previous state
- Use declarative policies when "the configuration must stay enforced and no one, not even admins or new APIs, can bypass it"

# Other organization policies (declarative category)

- **Tag policies**: standardize tag keys/values (case, allowed values); can enforce on specific resource types (prevents non-compliant tagging operations); report non-compliance. Do NOT force tags to exist - use SCP with `aws:RequestTag` / `Null` for that
- **Backup policies**: centrally deploy AWS Backup plans to accounts
- **AI services opt-out policies**: opt out of AWS AI services (Rekognition, Lex, etc.) using our content for service improvement
- **Chat applications policies** (formerly chatbot policies): control which Slack/Teams workspaces/channels can access accounts through Amazon Q Developer in chat applications
- Newer types: Security Hub, Amazon Inspector, GuardDuty, Amazon Bedrock (enforce Guardrails), S3, upgrade rollout policies
- These policies support **effective policy** view (inheritance operators `@@assign`, `@@append`, `@@remove`), unlike SCPs/RCPs
- Unlike SCPs/RCPs, management policies **do** apply to the management account
