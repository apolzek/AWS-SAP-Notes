# STS

- Allows to assume roles across different accounts or same accounts
- Generates temporary credentials (`sts:AssumeRole*`)
- Temporary credentials are similar to access key. They expire and they don't directly belong to the identity which assumes the role
- Temporary credentials usually provide limited access
- Temporary credentials are requested by another identity (AWS or external - identity federation)
- Temporary credentials include the following:
    - `AccessKeyId`: unique ID of the credentials (temporary keys start with `ASIA`, long-term keys with `AKIA`)
    - `Expiration`: date and time of credential expiration
    - `SecretAccessKey`: used to sign the requests to AWS
    - `SessionToken`: unique token which must be passed with all the requests to AWS
- STS allows us to enable identity federation

## STS APIs (exam)

- `AssumeRole`: IAM users/roles, cross-account, role chaining - duration 15 min - role max session (1-12 h, default 1 h); **role chaining max 1 h**
- `AssumeRoleWithSAML`: Enterprise SAML 2.0 IdP federation - duration 15 min - role max session (up to 12 h), set by `SessionDuration` SAML attribute
- `AssumeRoleWithWebIdentity`: OIDC (Cognito identity pools, EKS IRSA, GitHub Actions, Google) - duration 15 min - role max session (up to 12 h)
- `GetSessionToken`: IAM user wanting MFA-protected temporary creds - duration 15 min - 36 h (default 12 h); root max 1 h
- `GetFederationToken`: Custom identity broker (legacy) - duration 15 min - 36 h (default 12 h); root max 1 h
- `AssumeRoot`: Management/delegated admin account performing root-only tasks in member accounts - duration max 15 min
- `GetWebIdentityToken`: AWS workloads getting a JWT for **external** services (outbound federation) - duration short-lived
- `GetCallerIdentity`: Who am I? Works even if all permissions are denied
- `DecodeAuthorizationMessage`: Decode encoded `UnauthorizedOperation` messages

- Recommended: Cognito identity pools / IAM Identity Center rather than `AssumeRoleWithWebIdentity` / custom brokers directly in mobile apps

## Assume a Role with STS

1. Define an IAM role within an account or cross-account
2. Define which principals can access the IAM role
3. Use the AWS STS (Secure Token Service) to retrieve the IAM role we have access to (`AssumeRole` API)
4. Temporary credentials can be valid from 15 minutes up to 12 hours (role `MaxSessionDuration`); default 1 hour

- Cross-account: the **trust policy** in account B must trust account A (or a specific role), **and** the identity policy in account A must allow `sts:AssumeRole` on the role ARN
- Trusting `arn:aws:iam::<account-A>:root` delegates the decision to account A's IAM admins
- **Role chaining**: assuming a role from a role session - session limited to **1 hour** regardless of max session duration
- Console switch role: session = min(role max duration, remaining user session)

## Session tags, source identity, session policies

- **Session tags**: key/value pairs passed when assuming a role (`AssumeRole`, SAML/OIDC attributes `https://aws.amazon.com/SAML/Attributes/PrincipalTag:*`), available as `aws:PrincipalTag` -> ABAC with federated users
    - Up to 50 session tags; **transitive tags** persist through role chaining
    - Requires `sts:TagSession` in the trust policy
- **Source identity** (`sts:SetSourceIdentity`): set once (e.g. corporate username) at first role assumption, **cannot be changed** in later chained sessions, logged in CloudTrail -> traceability of who did what across role chains
- **Session policies**: passed with the assume call to further restrict the session (max 2,048 chars, up to 10 managed ARNs). Effective = intersection of role policies and session policy
- Session policies + tags count against the packed session token size (`SessionTokenUtilization` in the response)

## STS endpoints

- Global endpoint `sts.amazonaws.com` + regional endpoints `sts.<region>.amazonaws.com`
- **Global endpoint change**: requests to the global endpoint from regions **enabled by default** (using Amazon DNS in a VPC) are now served **in the local region** instead of only `us-east-1`. Requests from opt-in regions or external DNS resolvers still go to `us-east-1`
    - Global endpoint requests still have `aws:RequestedRegion = us-east-1` and are logged in CloudTrail `us-east-1`
- Best practice: use **regional endpoints** (SDK setting `sts_regional_endpoints = regional`, default in current SDKs) for latency, resilience and to remove the `us-east-1` dependency
- Opt-in regions: global endpoint tokens are v1 by default (valid only in default regions); set the account to issue **v2 tokens** (valid in all regions) or use regional endpoints
- Regional STS endpoints can be deactivated per account (except `us-east-1` and the global endpoint)
- STS supports **interface VPC endpoints** (PrivateLink) - private workloads without internet/NAT can still get credentials
- Request quota: 600 RPS per account per region (shared by `AssumeRole`, `GetCallerIdentity`, `GetSessionToken`, etc.); cross-account `AssumeRole` counts only against the caller account; increase via Support

## Revoke IAM Role Temporary Credentials

- **Trust policy**: specifies who can assume a role
- Roles can be assumed by many identities
- Everybody who assumes a role, gets the same set of permissions
- Temporary credentials can not be cancelled, they are valid until they expire
- Temporary credentials can last for longer time
- In case of a credential leak if we change the permissions for the policy, we will affect all legitimate users - not a good idea for revoking access
- Solution: 
    - Revoke all existing sessions, by applying an `AWSRevokeOlderSessions` inline policy to the role. This will apply to all existing sessions, sessions created afterwards will not be affected
        - The policy denies `*` with condition `aws:TokenIssueTime` older than the revocation time (`DateLessThan`)
        - Legitimate users simply re-assume the role and get fresh credentials
    - We can not manually revoke credentials!
- Revoking sessions of an **IAM user** (`GetSessionToken`/`GetFederationToken`): attach a deny policy using `aws:TokenIssueTime`, deactivating/deleting the user's access keys does NOT invalidate temporary credentials already issued
- Instance role credentials leaked from an EC2 instance: revoke older sessions + use `aws:EC2InstanceSourceVPC` / `aws:EC2InstanceSourcePrivateIPv4` conditions (or GuardDuty `InstanceCredentialExfiltration` findings) to detect/block use outside the instance
