# IAM Identity Center (Successor to AWS Single Sing-On - SSO)

- A way to use existing enterprise identity store with AWS
- Allows to centrally manage SSO access to multiple AWS accounts and external business applications as well
- Replaces the historical uses cases provided by SAML 2.0
- Flexible Identity store: where identities are stored. Allows for external identities to be swapped with AWS credentials
- IAM Identity Center supports the following type of identity stores:
    - Built-in identity store
    - AWS Managed Microsoft AD
    - On-premise Microsoft AD (Two way trust or AD Connector)
    - External Identity Provider - SAML 2.0
        - Okta, Microsoft Entra ID, Ping, Google Workspace, OneLogin, JumpCloud...; users/groups provisioned automatically with **SCIM**
- Only **one identity source** at a time per instance (changing it can delete existing users/assignments)
- IAM Identity Center is preferred by AWS for any "workforce" (enterprise) identity federation over the traditional SAML 2.0 based identity federation
- No additional cost

## IAM Identity Center Architecture

![IAM Identity Center](images/AWSSSO.png)

- A requirement of IAM Identity Center is that we should have a valid AWS Organization created
    - True for an **organization instance** (multi-account access). An **account instance** can be enabled in a standalone or member account, but only for application access (no AWS account access)

## Permission sets

- A permission set = collection of policies (AWS managed, customer managed by name, inline, permissions boundary) + session duration (1-12 h)
- Assigning *user/group + permission set + account* creates an IAM role `AWSReservedSSO_<permissionset>_<id>` in the target account (managed by Identity Center - do not edit)
- Customer managed policy references: the policy must exist with the same name/path in each target account
- Users get credentials through the **AWS access portal** (console or `aws sso login` for CLI/SDK short-term credentials)
- **ABAC**: attributes from the identity source (department, cost center) passed as session tags (*attributes for access control*) -> one permission set using `aws:PrincipalTag` instead of many
- Exam trap: resource policies (e.g. S3 bucket policy) referencing Identity Center users must use the role ARN pattern `arn:aws:iam::<acct>:role/aws-reserved/sso.amazonaws.com/<region>/AWSReservedSSO_*` or `aws:PrincipalTag`/`aws:userid` conditions

## Organization instance vs account instance

- **Organization instance** (recommended): enabled in the management account, one per organization
    - Multi-account permissions, access portal for AWS accounts, SAML 2.0 customer managed apps, delegated admin, customer managed KMS key, multi-region replication
- **Account instance**: bound to a single account and region (standalone or member account)
    - Only for AWS managed applications / OIDC apps (e.g. Amazon Q Developer, Amazon Quick) - **no** AWS account access, no SAML apps, no delegated admin
    - Useful for isolated application teams or quick tests
- **Delegated administrator**: register a member account to manage Identity Center - it can **not** manage permission sets assigned to the management account (protects the management account)

## Multi-region

- Identity Center is deployed in a single **home region**, historically a single point of failure for sign-in
- **Multi-Region replication** (organization instances): replicate workforce identities, permission sets, assignments and sessions from the primary region to additional regions; users can access accounts and apps through additional-region access portals if the primary is impaired
    - Feb 2026: external IdP instances; Jul 2026: also Identity Center directory; requires a **multi-region customer managed KMS key**
- Without replication: break-glass IAM users/roles in critical accounts for regional outages of Identity Center

## Trusted identity propagation

- Propagates the **user's identity** (not just a role) from a client application to AWS services, so data access can be authorized and audited **per user**
- Supported by e.g. Amazon Redshift, Athena, EMR, Lake Formation, **S3 Access Grants**, Amazon Q Business, QuickSight
- Requires an organization instance and an OAuth 2.0 trusted token issuer for external apps
- Exam pattern: "BI users must access data with their corporate identity and fine-grained per-user permissions logged in CloudTrail" -> Identity Center + trusted identity propagation

## Other features

- **MFA**: built-in directory supports passkeys/security keys, authenticator apps; can require MFA every sign-in or on context change. With external IdP, MFA is done at the IdP
- **Session management**: configurable access portal session duration (default 8 h), sessions can be revoked by admins
- **Customer managed applications**: SAML 2.0 and OAuth 2.0 apps (SSO to third-party SaaS)
- **Temporary elevated access**: via partner solutions or the AWS TEAM solution (just-in-time permission set assignment)
- Quota: up to 7,000 accounts per organization instance by default

## Identity Center vs IAM SAML federation vs Cognito

- **Identity Center**: workforce users, many accounts, central assignments, SCIM provisioning - default answer for multi-account human access
- **IAM SAML/OIDC federation per account**: legacy or special cases (one account, custom role mapping, workloads like GitHub Actions)
- **Cognito**: customers of our applications (CIAM), not workforce access to AWS accounts
