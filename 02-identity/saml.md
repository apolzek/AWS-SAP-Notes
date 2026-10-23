# SAML 2.0 Identity Federation

- On AWS, only AWS credentials can be used to access AWS resources, so an exchange has to take place
- Federation lets user outside of AWS to assume a temporary role for access AWS resources
- These users assume identity provider access role
- SAML 2.0 - Security Assertion Markup Language
- Most large identity providers (IdP) use the SAML 2.0 open standard
- SAML 2.0 allows to **indirectly** use on-premise identities with AWS (console and CLI)
- SAML 2.0 based identity federation is used when we have an enterprise based identity provider which is SAML 2.0 compatible
- SAML 2.0 based federation is ideal when we have an existing identity management team managing access to other services including AWS
- If we are looking to maintain a single source of truth and/or we have more than 5000 users, SAML 2.0 based federation is recommended to be used
    - 5,000 = hard limit of IAM users per account
- Federation is using IAM Roles and AWS Temporary Credentials
    - Session duration: default 1 hour, up to the role's maximum session duration (max 12 hours), requested by the IdP with the `https://aws.amazon.com/SAML/Attributes/SessionDuration` attribute

## SAML 2.0 Identity Federation Authentication Process - API Access

![SAML 2.0 Federation API](images/SAML2.0FederationAPI.png)

## SAML 2.0 Identity Federation Authentication Process - AWS Console Access

![SAML 2.0 Federation Console](images/SAML2.0FederationConsole.png)

## SAML 2.0 Federation

- We need to setup trust between AWS IAM and SAML (both ways)
    - In IAM: create a **SAML identity provider** with the IdP metadata XML
    - In the IdP: register AWS as a service provider / relying party (`https://signin.aws.amazon.com/saml`, or a regional sign-in endpoint for resilience)
    - Create IAM roles whose trust policy allows `sts:AssumeRoleWithSAML` from the SAML provider, with condition `SAML:aud`
- SAML 2.0 enabled web based, cross domain SSO
- Uses the STS API: `AssumeRoleWithSAML`
- It is the old way of doing federation, recommended way by AWS is to use **Amazon Single Sign On (SSO)**
    - Now **IAM Identity Center** (AWS SSO was renamed in 2022)

## SAML assertion attributes

- `https://aws.amazon.com/SAML/Attributes/Role`: role ARN + provider ARN pairs the user may assume (mapped from AD groups)
- `https://aws.amazon.com/SAML/Attributes/RoleSessionName`: identifier of the session (e.g. user email) shown in CloudTrail
- `https://aws.amazon.com/SAML/Attributes/SessionDuration`: requested session length
- `https://aws.amazon.com/SAML/Attributes/PrincipalTag:<key>`: **session tags** for ABAC; `TransitiveTagKeys` to keep tags on role chaining
- `https://aws.amazon.com/SAML/Attributes/SourceIdentity`: source identity for traceability across role chains
- Condition keys usable in trust/permission policies: `saml:sub`, `saml:aud`, `saml:iss`, `saml:namequalifier`, plus mapped attributes (e.g. `saml:edupersonorgdn`)

## Details and exam traps

- AWS acts as **service provider**; the IdP authenticates the user (AD FS, Okta, Entra ID, Ping...). AWS never sees the user's password
- **IdP-initiated** flow for the console (user signs in to the IdP portal and picks a role); for CLI, a helper tool/script obtains the assertion and calls `AssumeRoleWithSAML`
- `AssumeRoleWithSAML` does **not** require AWS credentials to call (the signed assertion is the proof)
- SAML assertion max size 100,000 characters (base64)
- IAM SAML providers support **encrypted SAML assertions** and multiple signing/encryption keys for certificate rotation
- IdP certificates expire - rotate metadata in IAM before expiry, otherwise all federated logins fail
- Users on-prem with **no SAML-compatible IdP** -> custom **identity broker** using `GetFederationToken` or `AssumeRole` (legacy pattern) - or deploy AD FS / use Identity Center with AD Connector
- Web/mobile users (social logins) -> not SAML, use **Cognito** (OIDC / web identity federation)

## When to use what

- **IAM Identity Center** (+ external IdP via SAML/SCIM): workforce access to many accounts - recommended default
- **Direct IAM SAML federation**: a few accounts, existing IdP role mapping per account, or requirements Identity Center doesn't meet (custom role per AD group per account)
- **Cognito user pools with SAML IdP**: enterprise users signing in to *our application* (B2B SaaS), not to AWS accounts
