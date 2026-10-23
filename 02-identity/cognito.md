# Amazon Cognito

- Provides authentication, authorization and user management for mobile/web applications
- Customer/consumer identity (CIAM) - for **workforce** access to AWS accounts use IAM Identity Center instead
- Provides 2 main pieces of functionality:
    - **User Pools**: used for sign-in, provides a JSON Web Token (JWT) after authentication
        - **User Pools do not grant access to AWS services!**
        - User Pools provide user directory management and user profiles, sign-up and sing-in with customizable web UI, MFA and other security features
        - They allow social sign-in provided by Google, Apple, Facebook as well as sign in using other identity types such as SAML identity providers
    - **Identity Pools**: the aim of identity pool is to exchange a type of external identity for temporary AWS credentials, which can be used to access AWS resources
        - Unauthenticated Identities: identity pools can provides access to AWS services for guest users (unauthenticated users)
        - Federated Identities: authorize users to temporarily access AWS resources after they authenticated with Google, Facebook, Twitter, SAML 2.0 and event Cognito User Pools

## Cognito concepts

![Cognito Concepts](images/CognitoConcepts.png)

## User Pools in detail

- Tokens (JWT, OIDC):
    - **ID token**: user identity/claims (authentication) - default 1 h (5 min - 1 day)
    - **Access token**: scopes/groups, used to authorize API calls - default 1 h (5 min - 1 day)
    - **Refresh token**: get new ID/access tokens - default 30 days (1 h - 10 years); refresh token rotation supported
- **Managed login** (new hosted UI, branding editor) or **classic hosted UI**; or build our own UI with the SDK/Amplify
- **App clients**: per app configuration (OAuth flows, scopes, token validity, secret)
- OAuth 2.0 flows: authorization code (+PKCE), implicit (legacy), **client credentials** (machine-to-machine)
- **Resource servers + custom scopes**: authorize APIs with access token scopes
- **Lambda triggers**: pre sign-up, pre/post authentication, pre token generation (customize ID and access token claims), custom message, user migration (just-in-time migration from a legacy user store), custom auth challenge
- **Groups**: role-based access; each group can map to an IAM role (used by identity pools)
- MFA: SMS, TOTP, email OTP; **passwordless** (email/SMS OTP, **passkeys/WebAuthn**) as first factor
- Federation: social (Google, Facebook, Apple, Amazon), **SAML 2.0** and **OIDC** IdPs - user pool acts as a single token issuer for the app
- Other: custom domains (ACM cert in `us-east-1`), user import (CSV), adaptive authentication, compromised-credentials detection (Plus), AWS WAF web ACL association
- Multi-tenant SaaS: one user pool per tenant (strong isolation), or shared pool with custom attributes/groups per tenant, or app client per tenant

## Feature plans (user pools)

- **Lite**: basic sign-up/sign-in, social/SAML/OIDC federation, classic hosted UI, M2M, TOTP/SMS MFA, user migration, Lambda triggers. Low cost for small MAU counts
- **Essentials** (**default** for new user pools): everything in Lite + managed login with visual editor, passwordless (OTP, passkeys), email MFA, choice-based sign-in, **access token** customization
- **Plus**: Essentials + threat protection (formerly *advanced security features*): compromised-credential checks, adaptive/risk-based authentication, user activity logs and export
- Plan is chosen per user pool (`UserPoolTier`); billed per MAU (and separately for federated MAUs and M2M token requests)

## Identity Pools in detail

- Exchange a token (user pool, SAML, OIDC, social, or **developer authenticated identities**) for AWS credentials via STS (`AssumeRoleWithWebIdentity` under the hood)
- Role selection:
    - Default authenticated / unauthenticated roles
    - **Rule-based** mapping (claims -> role) or **token-based** (user pool group `cognito:preferred_role`)
- **Attributes for access control** (ABAC): map IdP claims to session tags -> `aws:PrincipalTag` in IAM policies
- Policy variables: `${cognito-identity.amazonaws.com:sub}` to scope access, e.g. per-user S3 prefix or DynamoDB partition key (`dynamodb:LeadingKeys`)
- Enhanced (simplified) flow is recommended; basic (classic) flow allows custom role selection by the client

## Integrations / when to use what

- **ALB**: authenticate users with a Cognito user pool (or any OIDC IdP) at the load balancer (`authenticate-cognito` action) - no auth code in the app
- **API Gateway**: Cognito user pool authorizer (REST), JWT authorizer (HTTP APIs); **IAM auth** with identity pool credentials
- **AppSync**: user pool or IAM (identity pool) authorization
- **CloudFront**: Lambda@Edge / CloudFront Functions validating JWTs
- **Amazon Verified Permissions**: fine-grained authorization (Cedar policies) using user pool tokens - Cognito authenticates, Verified Permissions authorizes
- Exam patterns:
    - Mobile app users need to upload directly to their own S3 prefix -> user pool + identity pool + IAM policy with `${cognito-identity.amazonaws.com:sub}`
    - Guest access to limited AWS resources -> identity pool unauthenticated role
    - Migrate users from a legacy DB without password reset -> user migration Lambda trigger
    - Microservice-to-microservice auth with OAuth -> client credentials grant + resource server scopes
- Multi-region: user pools are regional; **multi-Region replication** (June 2026, add-on for Essentials/Plus) syncs users and machine identities in near real time to a standby replica pool in another region
    - Primary pool = source of truth for writes (sign-up, password reset, admin config); secondary serves sign-in and token issuance during failover
    - Only eligible pools (modern Cognito infrastructure) in supported regions; otherwise DR = export/import users + rebuild (users must reset passwords)
