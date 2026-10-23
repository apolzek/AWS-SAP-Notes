# Service Catalog

- A service catalog is a document or database crate by the IT team containing an organized collection of products
- Used when different teams in the business use a service-charge module
- Key product information: Product Owner, Cost, Requirements, Support Information, Dependencies
- Defines approval of provisioning from IT and customer side
- Designed for managing cost and scale service delivery

## AWS Service Catalog

- Portal for end users who can launch predefined products by admins
- End user permissions can be controlled
- Admins can define those products using CloudFormation (or Terraform - HashiCorp Terraform Cloud/Enterprise or open-source Terraform via an engine) and the permissions required to launch them
- Admins build products into portfolios which are made visible to the end users
- Service Catalog architecture:
    ![Service Catalog architecture](images/AWSServiceCatalog.png)

## Key Concepts

- **Product**: a CloudFormation/Terraform template + versions (provisioning artifacts). Can be sourced from Git (CodeConnections) to auto-sync new versions
- **Portfolio**: collection of products + configuration (constraints, access). Access granted to IAM users/groups/roles (or principal name patterns, useful with Identity Center permission sets)
- **Provisioned product**: a launched stack instance owned by the end user
- Portfolio sharing:
    - Account-to-account share or **AWS Organizations share** (to OUs/org) - recommended at scale <span style="color: red;">EXAM</span>
    - Imported portfolios: recipients can add their own constraints/launch roles but can't change the products
    - Can use a delegated administrator account instead of the management account

## Constraints

- **Launch constraint**: IAM role that Service Catalog assumes to provision the product. End users only need Service Catalog permissions, not permissions on EC2/RDS/etc. - least privilege <span style="color: red;">EXAM</span>
    - Can also use a local role name (`LocalRoleName`) so the same constraint works in every account the portfolio is shared to
- **Template constraint**: limit allowed parameter values (e.g. only `t3.micro`/`t3.small` instance types)
- **Notification constraint**: send stack events to SNS
- **Tag update constraint**: allow/deny end users updating tags after launch
- **StackSet constraint**: deploy the product across accounts/regions with CloudFormation StackSets
- **TagOptions**: enforce/apply key-value tags (cost allocation) on provisioned products

## Other Features

- **Service actions**: let end users run SSM Automation documents against provisioned products (e.g. restart EC2, backup) without direct permissions
- **AppRegistry**: repository of applications and their associated resources (CloudFormation stacks, resource tag queries) + attribute groups (metadata such as owner, cost center, criticality). Foundation for *myApplications* in the console (cost, health, security per application)
- **Self-service for developers**: combine with Control Tower **Account Factory** (itself a Service Catalog product) and **Account Factory for Terraform (AFT)** for account vending
- Integrates with CloudFormation **StackSets**, Config (track provisioned products), EventBridge, CloudTrail
- Pricing: per API call above a free tier (1,000 calls/month)

## Exam Scenarios

- "Users must launch only approved, compliant resources without having admin permissions" -> Service Catalog + launch constraint
- "Share standardized products with all accounts in the organization" -> Organizations portfolio share
- "Restrict which instance types users can pick" -> template constraint
- "Platform team publishes versioned templates for developers" -> Service Catalog (AWS Proton reached end of support Oct 2026)
