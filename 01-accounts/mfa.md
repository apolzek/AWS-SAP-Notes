# Multi-Factor Authentication (MFA)

- **Factor**: different piece of evidence which proves the identity
- Factors:
    - **Knowledge**: something we as users know: username, password
    - **Possession**: something we as users have: bank card, MFA device/app
    - **Inherent**: something we are, example: fingerprint, face, voice, iris
    - **Location**: a location (physical) or which network we are connected to (corporate wifi)
- More factors means more security, harder to bypass by an intruder

## MFA device types in AWS (IAM and root users)

- **Passkeys and security keys (FIDO2)**: synced passkeys (Apple, Google, Microsoft, password managers) or hardware keys (YubiKey). **Phishing resistant** - AWS recommended option
- **Virtual authenticator apps (TOTP)**: Google Authenticator, Authy, Microsoft Authenticator, etc.
- **Hardware TOTP tokens**: physical key fob / display card
- SMS MFA is **not** supported for IAM users (SMS OTP exists only in app-facing services such as Cognito user pools)
- Up to **8 MFA devices** (any mix of types) per root user and per IAM user - register a backup device

## Mandatory root MFA

- AWS phased in mandatory MFA for root users:
    - May 2024: Organizations **management account** root users
    - June 2024: **standalone** account root users
    - June 2025: **member account** root users
- Recommended for member accounts instead: **centralized root access management** - delete member account root credentials entirely (no password, no MFA to manage) and perform privileged tasks with `sts:AssumeRoot` from the management/delegated admin account

## MFA in IAM policies

- `aws:MultiFactorAuthPresent`: `true` if the request was made with MFA-authenticated temporary credentials
- `aws:MultiFactorAuthAge`: seconds since MFA authentication - enforce "MFA within the last hour" for sensitive actions
- Exam trap: long-term access keys (CLI/SDK) never carry MFA context, `aws:MultiFactorAuthPresent` is **absent** (not `false`)
    - Use `BoolIfExists` with `"aws:MultiFactorAuthPresent": "false"` in a `Deny` statement to also block requests without the key
    - `"Bool": {"aws:MultiFactorAuthPresent": "false"}` alone does NOT deny access key requests
- **MFA-protected API access (CLI)**: call `sts:GetSessionToken` (or `AssumeRole`) with `SerialNumber` + `TokenCode` to get temporary credentials that carry MFA context
- Role trust policies can require MFA to assume the role (`aws:MultiFactorAuthPresent` in the trust policy condition)

## MFA in other services

- **S3 MFA Delete**: requires MFA to permanently delete object versions or change versioning state; can only be enabled by the **root user** via CLI/API (not console); bucket must be versioned
- **IAM Identity Center**: MFA configured at the identity source - built-in directory (passkeys/security keys, authenticator apps, context-aware "always-on" or "only when sign-in context changes") or delegated to the external IdP
- **Cognito user pools**: SMS, TOTP, email OTP (Essentials/Plus), passkeys as first factor (Essentials/Plus)
- **Directory Service (Managed Microsoft AD, AD Connector)**: RADIUS-based MFA (e.g. for WorkSpaces)
