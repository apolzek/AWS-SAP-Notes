# IAM: Identity and Access Management

- When accessing AWS, the root account should **never** be used. Users must be created with the proper permissions. IAM is central to AWS
- IAM is a **global** service (control plane in `us-east-1`, data plane replicated to every region) - IAM changes are eventually consistent, avoid putting IAM writes in critical/high-availability code paths
- **Users**: A physical person (or a legacy workload needing long-term keys)
    - Modern best practice: prefer **IAM Identity Center** (workforce) or **IAM roles** (workloads) over IAM users with long-term access keys
- **Groups**: Functions (admin, devops) Teams (engineering, design) which contain a group of users
    - Groups can not be nested, groups can not be a `Principal` in a policy (groups are not identities)
- **Roles**: Internal usage within AWS resources
    - **Cross Account Roles**: roles used to assumed by another AWS account in order to have access to some resources in our account
    - Role = permissions policy + **trust policy** (who can assume it). No long-term credentials, uses STS temporary credentials
    - **Service roles**: assumed by an AWS service (`ec2.amazonaws.com`, `lambda.amazonaws.com`) to act on our behalf
    - **Service-linked roles (SLR)**: predefined by the service, trust and permissions owned by the service, can not be modified. **SCPs and RCPs do NOT restrict SLRs**
    - **Instance profile**: container which passes a role to an EC2 instance (credentials via IMDS)
    - `iam:PassRole`: permission needed to hand a role to a service (EC2, Lambda, ECS, CloudFormation). Classic privilege-escalation vector - scope it to specific role ARNs
- **Policies (JSON documents)**: Defines what each of the above can and cannot do. **Note**: IAM has predefined managed policies
    - There are 3 types of policies:
        - AWS Managed
        - Customer Managed
        - Inline Policies
- **Resource Based Policies**: policies attached to AWS services such as S3, SQS

## IAM Roles vs Resource Based Policies

- When we assume a role (user, application or service), we give up our original permission and take the permission assigned to the role
- When using a resource based policy, principal does not have to give up any permissions
- Example: user in account A needs to scan a DynamoDB table in account A and dump it in an S3 bucket in account B. In this case if we assume a role in account B, we wont be able to scan the table in account A
- Resource-based policies are supported only by some services (S3, SQS, SNS, KMS, Lambda, Secrets Manager, ECR, API Gateway, EventBridge, Glacier, etc.). DynamoDB also supports resource-based policies on tables/streams (since 2024)
- Cross-account access with a resource-based policy: both the identity policy (account A) **and** the resource policy (account B) must allow the action

## Third-party access - External ID (confused deputy)

- When a third party (SaaS vendor) assumes a role in our account, require an `sts:ExternalId` condition in the trust policy
- External ID is not a secret, it prevents the **confused deputy** problem (vendor being tricked into using our role on behalf of another customer)
- For AWS services acting on our behalf, the equivalent protection is `aws:SourceArn` / `aws:SourceAccount` / `aws:SourceOrgID` conditions in resource/trust policies

## IAM Identity Providers (federation into IAM)

- **SAML 2.0 provider**: enterprise IdP (AD FS, Okta, Entra ID) -> `AssumeRoleWithSAML`
- **OIDC provider**: web identity (GitHub Actions, EKS IRSA, Google) -> `AssumeRoleWithWebIdentity`
    - AWS secures connections to well-known IdPs with its own trusted CA library, the thumbprint is no longer used for those
    - Always restrict the `sub` / `aud` claims in the trust policy (e.g. GitHub repo + branch), otherwise any repo could assume the role
- Prefer **IAM Identity Center** for human workforce access across many accounts; use IAM SAML/OIDC providers for per-account or workload federation

## IAM Roles Anywhere

- Lets workloads **outside AWS** (on-premises servers, other clouds) obtain temporary credentials using X.509 certificates
- Trust anchor = our CA (AWS Private CA or external CA) + profile (which roles can be assumed)
- Replaces long-term access keys on on-premises servers - common exam answer

## Outbound identity federation

- Lets AWS workloads get a signed JWT from STS (`sts:GetWebIdentityToken`) and present it to **external** services (OIDC relying parties) instead of storing third-party secrets
- Enabled per account (`EnableOutboundWebIdentityFederation`), creates an account-specific OIDC issuer URL
- Not available on the STS global endpoint, tokens can not be used to federate back into AWS

## IAM Access Analyzer

- **External access analyzer**: finds resources (S3, KMS keys, IAM roles, Lambda, SQS, Secrets Manager, SNS, EBS/RDS snapshots, ECR, EFS, DynamoDB...) shared outside the **zone of trust** (account or organization)
- **Internal access analyzer**: which principals **inside** the org/account can access selected critical resources (S3, DynamoDB, RDS)
- **Unused access analyzer** (paid): unused roles, unused access keys, unused passwords, unused service/action-level permissions; can generate recommended policies to remove unused permissions
- **Policy validation**: grammar + best-practice checks (100+ checks) when authoring policies
- **Custom policy checks** (paid, automated reasoning): `CheckNoNewAccess`, `CheckAccessNotGranted`, `CheckNoPublicAccess` - integrate in CI/CD to block risky policy changes
- **Policy generation**: generates a least-privilege policy from CloudTrail activity of a role/user
- Can be run organization-wide from a **delegated administrator** account

## Other IAM tools

- **Credential report**: account-level CSV of all users and status of their credentials (passwords, keys, MFA)
- **Last accessed information (Access Advisor)**: service and action-level last accessed data for users, roles, groups, policies - and for OUs/accounts in Organizations (helps refine SCPs)
- **ABAC (attribute-based access control)**: permissions based on tags (`aws:PrincipalTag`, `aws:ResourceTag`, `aws:RequestTag`) - scales better than RBAC with many resources/teams, new resources do not need policy changes

## Key quotas (per account)

- Users: 5,000 (**not adjustable**) - for more users use federation / Identity Center
- Groups: 300 (up to 500); groups per user: 10
- Roles: 1,000 (adjustable up to 10,000)
- Customer managed policies: 1,500 (adjustable up to 10,000)
- Managed policies attached: role 20 (up to 25), user 10 (up to 20), group 10
- Managed policy size: 6,144 chars; inline aggregate: user 2,048, group 5,120, role 10,240 chars
- Role trust policy: 2,048 chars (adjustable up to 8,192)
- Access keys per user: 2 (allows rotation); MFA devices per user: 8
- Versions per managed policy: 5

## Best practices

- One IAM User per person **ONLY**
- One IAM Role per Application
- IAM credentials should **NEVER** be shared
- Never write IAM credentials in your code. **EVER**
- Never use the ROOT account except for initial setup
- It's best to give users the minimal amount of permissions to perform their job
- Prefer temporary credentials (roles, Identity Center, Roles Anywhere) over long-term access keys; rotate any remaining keys
- Require MFA for humans (phishing-resistant passkeys/security keys preferred)
- Use permissions boundaries to safely delegate IAM administration to developers
- Use Access Analyzer + last accessed data to continuously remove unused permissions
- In an organization, centralize root access and delete member account root credentials (see `organizations.md`)
