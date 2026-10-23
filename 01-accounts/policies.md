# Policies

- IAM policies define permissions for an action regardless of the method that you use to perform the operation

## Policy types

- **Identity-based policies**: attach managed and inline policies to IAM identities (users, groups to which users belong, or roles). Identity-based policies grant permissions to an identity
- **Resource-based policies**: attach inline policies to resources. The most common examples of resource-based policies are Amazon S3 bucket policies and IAM role trust policies. Resource-based policies grant permissions to a principal entity that is specified in the policy. Principals can be in the same account as the resource or in other accounts
- **Permissions boundaries**: use a managed policy as the permissions boundary for an IAM entity (user or role). That policy defines the maximum permissions that the identity-based policies can grant to an entity, but does not grant permissions. Permissions boundaries do not define the maximum permissions that a resource-based policy can grant to an entity
- **Organizations SCPs**: use an AWS Organizations service control policy (SCP) to define the maximum permissions for account members of an organization or organizational unit (OU). SCPs limit permissions that identity-based policies or resource-based policies grant to entities (users or roles) within the account, but do not grant permissions
- **Organizations RCPs**: resource control policies define the maximum permissions for **resources** in member accounts (S3, KMS, STS, SQS, Secrets Manager, etc.), applied to any caller including principals outside the org. Do not grant permissions
- **Access control lists (ACLs)**: use ACLs to control which principals in other accounts can access the resource to which the ACL is attached. ACLs are similar to resource-based policies, although they are the only policy type that does not use the JSON policy document structure. ACLs are cross-account permissions policies that grant permissions to the specified principal entity. ACLs cannot grant permissions to entities within the same account
    - S3 ACLs are disabled by default on new buckets (Object Ownership = *Bucket owner enforced*); AWS recommends keeping them disabled
- **Session policies**: pass advanced session policies when you use the AWS CLI or AWS API to assume a role or a federated user. Session policies limit the permissions that the role or user's identity-based policies grant to the session. Session policies limit permissions for a created session, but do not grant permissions. For more information, see Session Policies
    - Max 1 inline JSON policy + up to 10 managed policy ARNs, combined 2,048 characters
- **VPC endpoint policies**: resource-style policy on a gateway/interface endpoint limiting what can be done through it (key piece of a data perimeter)

## Policies Deep Dive

- Anatomy of a policy: JSON document with `Effect`, `Action`, `NotAction` (inverse condition of `Action`), `Resource`, `Conditions` and `Policy Variables`
    - Resource-based policies also have `Principal` / `NotPrincipal`
    - `Version` should always be `"2012-10-17"` (required for policy variables)
- Priority order of permissions in AWS is: deny (explicit) > allow > deny (implicit). A policy always assumes a default (implicit) deny => if we do not allow explicitly to do something, we wont be able to do it
- An explicit `DENY` has always precedence over `ALLOW`
- `NotAction` with `Allow` is risky: it allows **everything else**, including new services - prefer `NotAction` with `Deny`
- Best practice: use least privilege for maximum security
    - Access Advisor: a tool for seeing permissions granted and when last accessed
    - Access Analyzer: used of analyze resources shared with external entities (also: internal access, unused access, policy validation, custom policy checks, policy generation from CloudTrail)
- Common Managed Policies:
    - `AdministratorAccess`
    - `PowerUserAccess`: does not allow anything regarding to IAM, organizations and account (with some exceptions), otherwise similar to admin access
    - `ReadOnlyAccess` / `ViewOnlyAccess` (view-only does not read data, e.g. no `s3:GetObject`)
    - Job function policies: `Billing`, `DatabaseAdministrator`, `NetworkAdministrator`, `SecurityAudit`, `SupportUser`, `SystemAdministrator`
- IAM policy condition:

    ```
    "Condition": {
        "{condition-operator}": {
            "{condition-key}": "{condition-value}"
        }
    }
    ```

- Operators:
    - String: `StringEquals`, `StringNotEquals`, `StringLike`, etc.
    - Numeric: `NumericEquals`, `NumericNotEquals`, `NumericLessThan`, etc.
    - Date: `DateEquals`, `DateNotEquals`, `DateLessThan`, etc.
    - Boolean
    - IpAddress/NotIpAddress:
        - `"Condition": {"IpAddress": {"aws:SourceIp": "192.168.0.0/16"}}`
        - `aws:SourceIp` only contains **public** IPs; requests through a VPC endpoint use `aws:VpcSourceIp` / `aws:SourceVpc` / `aws:SourceVpce` instead
    - ArnEquals/ArnLike
    - Null
        - `"Condition": {"Null": {"aws:TokenIssueTime": "true"}}` - `true` means the key is **absent** (e.g. request made with long-term credentials)
    - `...IfExists` suffix: evaluate only if the key is present (e.g. `BoolIfExists`)
    - Multi-valued keys: `ForAnyValue:` / `ForAllValues:` (e.g. `aws:TagKeys`). Trap: `ForAllValues` returns true when the key is missing/empty
    - Logic: multiple keys in a condition block = AND; multiple values for one key = OR
- Policy Variables and Tags:
    - `${aws:username}`: example `"Resource:["arn:aws:s3:::mybucket/${aws:username}/*"]`
    - AWS Specific:
        - `aws:CurrentTime`
        - `aws:TokenIssueTime`
        - `aws:PrincipalType`: indicates if the principal is an account, user, federated or assumed role
        - `aws:SecureTransport`
        - `aws:SourceIp`
        - `aws:UserId`
        - `aws:RequestedRegion`: region of the API call (used in region-restriction SCPs)
        - `aws:MultiFactorAuthPresent`, `aws:MultiFactorAuthAge`
        - `aws:CalledVia`, `aws:ViaAWSService`: request made by a service on behalf of the principal (e.g. CloudFormation, Athena)
        - `aws:SourceIdentity`: identity set at first role assumption, persists through role chaining
    - Organization keys (data perimeter):
        - `aws:PrincipalOrgID`, `aws:PrincipalOrgPaths`: caller belongs to our org/OU - replaces listing every account ID in resource policies
        - `aws:ResourceOrgID`, `aws:ResourceOrgPaths`, `aws:ResourceAccount`: target resource belongs to our org
        - `aws:SourceOrgID`, `aws:SourceAccount`, `aws:SourceArn`: for AWS service principals (confused deputy protection)
        - `aws:PrincipalIsAWSService`: exempt AWS service principals from perimeter denies
    - Network keys: `aws:SourceVpc`, `aws:SourceVpce`, `aws:VpcSourceIp`
    - Service Specific:
        - `ec2:SourceInstanceARN`
        - `s3:prefix`
        - `s3:max-keys`
        - `sns:EndPoint`
        - `sns:Protocol`
    - Tag Based:
        - `iam:ResourceTag/key-name`
        - `iam:PrincipalTag/key-name`
        - Global: `aws:ResourceTag/key`, `aws:RequestTag/key`, `aws:TagKeys`, `aws:PrincipalTag/key` (ABAC)

## Data perimeter

- Combination of policies ensuring only **trusted identities** access **trusted resources** from **expected networks**
    - Identity perimeter: RCPs / resource policies with `aws:PrincipalOrgID`
    - Resource perimeter: SCPs / VPC endpoint policies with `aws:ResourceOrgID`
    - Network perimeter: SCPs / resource policies with `aws:SourceVpc`, `aws:SourceVpce`, `aws:SourceIp`
- Exam pattern: "prevent data exfiltration to buckets outside the org" -> VPC endpoint policy/SCP with `aws:ResourceOrgID`; "prevent outsiders accessing our buckets even if misconfigured" -> RCP

## Permission Boundaries

- Only IDENTITY permissions are impacted by boundaries - any resource policies are applied full
    - Nuance (same account): a resource policy granting access to an **IAM user ARN** or an **assumed-role session ARN** is not limited by the boundary; one granting access to the **role ARN** is limited by an implicit deny in the boundary/session policy
- Permission boundaries can be applied to IAM Users and IAM Roles
- Permission boundaries don't grant access to any action. They define maximum permissions an identity can receive
- Use cases for permission boundaries:
    - Delegation problem: if we give elevated permissions to an user, he/she could promote itself to have administrator permissions or could create another user with administrator permissions
    - Solution is to have a boundary which forbids changing its onw user's permissions and forbid creating other users/roles with elevated permissions
    - Implementation: allow `iam:CreateRole` only with condition `iam:PermissionsBoundary` = the boundary ARN, and deny changing/deleting the boundary itself
- Boundary vs SCP: boundary is per user/role inside an account (delegation by the account admin); SCP is per account/OU (governance by the org)

## Policy Evaluation Logic

- Components involved in a policy evaluations:
    - Organization SCPs
    - Organization RCPs
    - Resource Policies
    - IAM Identity Boundaries
    - Session Policies
    - Identity Policies
- Evaluation order (same account):
    1. Explicit deny in any policy -> **Deny**
    2. SCPs must allow (principal side) and RCPs must allow (resource side) -> otherwise implicit deny
    3. Resource-based policy allows -> Allow (for most cases, even without identity policy, unless boundary/session policy apply to a role session)
    4. Identity policy must allow, permissions boundary must allow, session policy must allow
- Different accounts: **both** the identity policy in the caller account **and** the resource policy in the resource account must allow (plus SCPs of the caller account and RCPs of the resource account)
- Special cases: IAM role trust policies and KMS key policies must explicitly allow - an identity policy alone is not enough for KMS unless the key policy delegates to IAM (`"Principal": {"AWS": "arn:aws:iam::<account>:root"}`)
- Policy evaluation logic - same account:
    ![policy evaluation logic - same account](images/PolicyEvaluation1.png)
- Policy evaluation logic - different account:
    ![policy evaluation logic - different account](images/PolicyEvaluation2.png)

## AWS Policy Simulator

- When creating new custom policies you can test it here:
  - https://policysim.aws.amazon.com/home/index.jsp
  - This policy tool can you save you time in case your custom policy statement's permission is denied
  - Simulates identity policies, permissions boundaries and (optionally) resource policies and SCPs; does not fully simulate RCPs/session policies - validate with CloudTrail `AccessDenied` events
- Alternatively, you can use the CLI:
    - Some AWS CLI commands (not all) contain `--dry-run` option to simulate API calls. This can be used to test permissions.
    - If the command is successful, you'll get the message: `Request would have succeeded, but DryRun flag is set`
    - Otherwise, you'll be getting the message: `An error occurred (UnauthorizedOperation) when calling the {policy_name} operation`
- `AccessDenied` messages now include the **policy type** that caused the deny (SCP, RCP, permissions boundary, session policy, identity/resource policy, explicit vs implicit) - speeds up troubleshooting
- Encoded authorization messages (EC2) can be decoded with `sts:DecodeAuthorizationMessage`
- IAM Access Analyzer **custom policy checks** can block risky policy changes in CI/CD before deployment
