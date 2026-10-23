# AWS Control Tower

- Control Tower's job is to allow quick and easy setup of a multi-account environment
- Control Tower orchestrates other AWS services to provide this functionality
- It is using Organizations, IAM Identity Center, CloudFormation, AWS Config and more
    - Also CloudTrail (organization trail), S3 (log buckets), Service Catalog (Account Factory), AWS Backup (optional), Security Hub (Service-Managed Standard)
- It is essentially an evolution of AWS Organizations adding more features, intelligence and automation
- No extra charge for Control Tower itself; we pay for underlying services (Config rules/recordings, CloudTrail, S3, KMS...)
- Supports up to 10,000 accounts per landing zone (Organizations limit still applies)

## Control Tower Features

- **Landing Zones**: it is the multi-account environment
    - Provides via other AWS services:
        - SSO/ID Federation
        - Centralized logging and auditing
- **Guard Rails**: used to detect/mandate rules and standards across all accounts
    - Renamed to **Controls**; the catalog of controls is now the **Control Catalog** (formerly Controls Library)
- **Account Factory**: automates and standardizes new account creation
- **Dashboard**: single page oversight of the entire environment
- When Control Tower is set up, it creates to OUs:
    - Foundational OU (security)
    - Custom OU (sandbox)
    - Landing zone 4.0 (Nov 2025) no longer requires a Security OU or a fixed structure - only requirement is that the hub accounts (log archive, audit) are in the same OU
- Inside the Foundational OU Control Tower creates two accounts:
    - Audit Account: for users who needs access to audit information provided by Control Tower
    - Log Archive Account: for users who need access to all log information to all enrolled accounts within the Landing Zones
    - We can bring existing accounts as Log Archive / Audit (instead of creating new ones)
- Within the Custom OU Account Factory will provision AWS accounts in an automated way
- For these new accounts we have bases templates for configurations:
    - Account Baseline template
    - Network Baseline template (cookie cutter)
- Control Tower utilizes CloudFormation under the hood to implement all of these automation
- It also uses both AWS Config and SCPs to implement account Guard Rails
    - Detective controls are now deployed as **service-linked AWS Config rules** (no StackSets, can only be changed through Control Tower)

## Landing Zones

- It is a feature designed that anyone should be able to implement a *well architected* multi-account environment
- Home Region: the region where we initially deploy the product
    - Can not be changed after setup; additional **governed regions** can be added
    - **Region deny** control: block API calls in non-governed regions (with exceptions for global services)
- A Landing Zone creates the Security and Sandbox OUs, we can create other OUs and accounts as well
- Landing Zones utilizes the IAM Identity Center for multiple-accounts and ID Federation
    - Identity Center integration is **optional** - we can opt out and use self-managed identity (e.g. existing IdP/IAM federation)
- A Landing Zone provides monitoring and notifications via CloudWatch and SNS
- We can allow end-users to provision new accounts from a Landing Zone using Service Catalog
- Landing zones are **versioned** (2.x, 3.x, 4.0); some features require updating the landing zone
    - 3.0: organization-level CloudTrail trail (instead of per-account trails)
    - 4.0 (Nov 2025): **optional service integrations** (Config, CloudTrail, Security Roles, Backup can be enabled/disabled individually), dedicated S3 buckets/SNS topics per service, flexible OU structure, service-linked Config aggregator in the Config hub account, **controls-only** minimal landing zone (enable controls without the full `AWSControlTowerBaseline`)
- Can be deployed over an **existing organization**: register existing OUs, enroll existing accounts
- **Automatic enrollment** (2025): moving an account into a registered OU applies that OU's baseline and controls automatically
- **Drift**: changes made outside Control Tower (moving accounts, modifying SCPs, deleting roles) - detect and repair (re-register OU, reset landing zone)
- Landing zone can be managed by APIs/CloudFormation (`CreateLandingZone`, `EnableBaseline`, `EnableControl`)

## Guardrails

- They are rules for multi-account governance
- There are 3 different types of rules:
    - Mandatory
    - Strongly Recommended
    - Elective
- Guardrails function in two different ways:
    - Preventive: stop us from doing things
        - They are implement using SCPs
            - Newer preventive controls can also be implemented with **RCPs** and **declarative policies** (EC2)
        - They are enforced or not enabled
        - Example: allow or deny regions; disallow bucket policy changes in the Org
    - Detective: compliance check
        - Uses AWS Config Rules to detect compliance violations
        - These types of Guardrails are either *clear*, *in violation* or *not enabled*
        - Example: check if CloudTrail is enabled; check if EC2 instances have public IPv4 attached
        - Control Catalog now includes hundreds of AWS Config managed rules and Security Hub controls that can be enabled from Control Tower
    - **Proactive**: checks resources **before** they are provisioned
        - Implemented with **CloudFormation Hooks**
        - Only applies to resources deployed through CloudFormation
        - Example: block a CloudFormation stack creating an unencrypted RDS instance
- Controls are mapped to frameworks (NIST, PCI DSS, CIS, etc.) in the Control Catalog (`ListControlMappings`)
- Controls are applied per **OU** (inherited by enrolled accounts); some can be enabled per account via API
- Exam trap: preventive (SCP) controls do not apply to the management account

## Account Factory

- Allows automated account provisioning
- This can be done by either cloud administrators or end users (with appropriate permissions)
- Guardrails are automatically applied to the provisioned accounts
- Account admin can be given to a named user who provisions the account or to another user
- Accounts can be configured with standard account and network configurations
- Can be fully integrated with a businessess SDLC
- Implemented as a **Service Catalog** product (provisioned product per account)
- **Account Factory Customization (AFC)**: blueprints (CloudFormation/Terraform via Service Catalog) applied at account creation/update
- **Account Factory for Terraform (AFT)**: GitOps pipeline (CodePipeline/CodeBuild) for account requests and customizations in Terraform
    - Separate AFT management account; supports Terraform Community Edition, HCP Terraform / Terraform Enterprise
    - Global customizations, account customizations, account provisioning customizations (Step Functions); plan-only runs (2026)
- **Customizations for Control Tower (CfCT)**: solution that deploys CloudFormation templates and SCPs/RCPs to OUs/accounts on lifecycle events (CloudFormation-based alternative to AFT)
- Lifecycle events (`CreateManagedAccount`, `UpdateManagedAccount`...) are published to EventBridge - trigger custom automation after account creation

## Control Tower vs Organizations vs Landing Zone Accelerator

- **Organizations**: building blocks (accounts, OUs, SCPs/RCPs) - we build governance ourselves
- **Control Tower**: managed, opinionated landing zone + controls + account vending on top of Organizations
- **Landing Zone Accelerator on AWS (LZA)**: AWS solution deployed on top of Control Tower for regulated environments (extensive networking/security config as code)
- Exam pattern: "set up a governed multi-account environment quickly with least operational overhead" -> Control Tower; "Terraform-based account vending" -> AFT
