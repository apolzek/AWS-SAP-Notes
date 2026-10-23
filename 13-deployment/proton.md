# AWS Proton

> **Status (2026):** AWS Proton reached **end of support on October 7, 2026**. It stopped accepting new customers on October 7, 2025 and the console/Proton resources are no longer accessible after the end-of-support date. CloudFormation stacks already deployed by Proton keep running - only the Proton management/delivery layer is gone.
> - Alternatives: AWS Service Catalog (standardized products/portfolios), CloudFormation/CDK + CodePipeline, AWS Launch Wizard, internal developer platforms (e.g. Backstage), Terraform modules + Service Catalog Terraform support
> - On the exam: if a question still lists Proton, the concept is "platform team publishes versioned templates, developers self-serve" - Service Catalog is the long-lived equivalent

- It is:
    - Automated infrastructure as code provisioning and deployment of serverless and container-based applications:
        - AWS Proton service is a two-pronged automation framework
        - We create versioned **service templates** that define standardized infrastructure and deployment tooling for serverless and container-based applications
        - AWS Proton identifies all existing service instances that are using an outdated template version for us
        - We can request AWS Proton to upgrade them with one click
    - Standardized infrastructure:
        - Platform teams can use AWS Proton and versioned infrastructure as code templates
    - Deployments integrated with CI/CD:
        - AWS Proton automatically provisions the resources, configures the CI/CD pipeline, and deploys the code into the defined infrastructure
- Concepts:
    - **Environment templates**: shared infra (VPC, ECS cluster) owned by platform team
    - **Service templates**: per-service infra (Fargate service, Lambda) + optional pipeline
    - Templates written in CloudFormation (Jinja) or Terraform (self-managed provisioning)
    - Major/minor template versions; Proton flags out-of-date service instances
