# CI/CD

- CI/CD architecture:
    ![CI/CD architecture](images/CICD1.png)
- Branching architecture:
    ![Branching architecture](images/CICD2.png)
- Code pipeline:
    ![Code pipeline](images/CICD3.png)
- Each pipeline has stages
- Each pipeline should be linked to a single branch in a repository (V1 pipelines). V2 pipelines can use trigger filters on branches/tags/file paths (glob patterns) - one pipeline can serve feature branches or a monorepo folder
    ![Code deployment](images/CICD4.png)
- CodeBuild/CodeDeploy configuration files:
    - `buildspec.yml, appspec.[yml|json]`
    - Reference to these files: [https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file.html](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file.html)
    - buildspec is used to influence the way the build process occurs within CodeBuild
    - appspec allows the influence how the deployment process proceeds in CodeDeploy

> **Status (2026):** AWS CodeStar was discontinued (July 31, 2024). AWS Cloud9 closed to new customers (July 2024). Amazon CodeCatalyst and CodeGuru Reviewer closed to new customers (Nov 7, 2025). CodeCommit was closed to new customers in July 2024 but AWS **reversed** this - CodeCommit returned to full GA on Nov 25, 2025.

## AWS CodeCommit

> **Status (2026):** Fully GA again and open to new customers (since Nov 25, 2025). Between Jul 2024 and Nov 2025 it was closed to new customers - older study material may say "use GitHub/GitLab instead".

- Managed git service
- Basic entity of CodeCommit is a repository
- Authentication can be configured via IAM console. CodeCommit supports HTTPS (Git credentials), SSH (keys uploaded to IAM user) and HTTPS over GRC (`git-remote-codecommit`, uses IAM credentials/roles - recommended for federated users/Identity Center)
- Encrypted at rest with KMS (AWS managed key or customer managed key) and in transit
- Cross-account access: create an IAM role in the repo account, users in the other account assume it (no resource policies on repos)
- Approval rule templates: require N approvals on pull requests, can be associated with many repositories
- Triggers and notifications:
    - Notifications rules: can send notifications based on events happening in the repo, example: commits, pull request, status changes, etc. Notifications can be sent to SNS topics or AWS Chatbot (now *Amazon Q Developer in chat applications*: Slack, Teams, Chime)
    - Triggers: allow the generate event driven processes based on things that happen in the repo. Events can be sent ot SNS or Lambda functions
    - Repo events are also sent to EventBridge (preferred for new event-driven designs, e.g. trigger CodePipeline)
- Exam: prevent pushes to `main` -> IAM policy denying `codecommit:GitPush`/`PutFile`/`MergeBranches*` with condition `codecommit:References`

## AWS CodePipeline

- It is a Continuos Delivery tool
- Controls the flow from source code, through build towards deployment
- Pipelines are built from stages. Stages contain actions which can be sequential or parallel (`runOrder`)
- Movement between stages can happen automatically or it can require a manual approval (manual approval action can notify an SNS topic)
- Actions within stages can consume artifacts or they can generate artifacts
- Artifacts are just files which are generated and/or consumed by actions. Stored in an S3 artifact bucket (one per region; cross-region actions need an artifact bucket + KMS key in each region)
- Any changes to the sate of a pipeline, stages or actions generate events which are published to Event Bridge
- CloudTrail can be used to monitor API calls. Console UI can be used to view/interact with the pipeline
- Source providers: CodeCommit, S3, ECR, GitHub/GitHub Enterprise, GitLab, Bitbucket (via CodeConnections - formerly CodeStar Connections)
- Action providers: CodeBuild, CodeDeploy, CloudFormation (create/update stack, change sets, StackSets), ECS (rolling and blue/green via CodeDeploy), Elastic Beanstalk, Lambda invoke, Step Functions, Service Catalog, S3 deploy, AppConfig, Jenkins, third-party
- Cross-account deployments: pipeline in tools account, artifact bucket encrypted with a **customer managed KMS key** (AWS managed key can't be shared), key policy + bucket policy allow the target account, pipeline assumes a cross-account role <span style="color: red;">EXAM</span>

### Pipeline types: V1 vs V2

- **V2** (default for new pipelines):
    - Triggers: push / pull request events with filters on branches, Git tags, file paths (glob) - per-source-action
    - Pipeline-level variables (set at execution start) + action output variables (`#{namespace.variable}`)
    - Execution modes: `SUPERSEDED` (default, newer run overtakes older one), `QUEUED` (one at a time, in order), `PARALLEL` (independent runs, e.g. per feature branch)
    - Stage-level: rollback to last successful execution, automatic retry on failure, stage conditions (entry/on-success/on-failure rules, e.g. CloudWatch alarm check, deployment window, Lambda rule)
    - Pricing: per action execution minute (first 100 min/month free) - V1 is $1 per active pipeline/month
- **V1**: legacy, single branch per source action, no trigger filters/variables at pipeline level

## AWS CodeBuild

- CodeBuild is a build as a service product
- It is fully managed, we pay only for the resources consumed during builds (per build minute, by compute type)
- CodeBuild is an alternative to the solutions provided by third party solutions such as Jenkins
- CodeBuild uses Docker for build environments which can be customized by us (curated AWS images or custom image from ECR/Docker Hub)
- CodeBuild integrates with other AWS services such as KMS, IAM, VPC, CloudTrails, S3, etc.
- Architecturally CodeBuild gets source material from GitHub, CodeCommit, CodePipeline or even S3 (also Bitbucket, GitLab)
- It builds and tests code. The build can be customized via `buildspec.yml` file which has to be located in the root of the source <span style="color: red;">Remember the spelling of file and location for EXAM</span>
- CodeBuild output logs are published to CloudWatch Logs, metrics are also published to CloudWatch Metrics and events to Event Bridge (or CloudWatch Events)
- CodeBuild supports build environments such as Java, Ruby, Python, Node.JS, PHP, .NET, Go and many more
- Compute options: EC2 (Linux, Linux GPU, ARM, Windows, macOS), Lambda compute (fast start, max 15 min, no Docker-in-Docker), **reserved capacity fleets** (warm, no queueing, can use custom AMI)
- VPC access: configure VPC/subnets/SGs to reach private resources (RDS, internal artifact repos) - needs NAT for internet access <span style="color: red;">EXAM</span>
- Caching: S3 cache or local cache (Docker layer, source, custom) to speed up builds
- Can act as a managed self-hosted runner for GitHub Actions, GitLab and Buildkite
- Batch builds (build graph/matrix), test reports (JUnit, Cucumber), code coverage reports
- Default timeout 60 min, max 36 hours (per build)

### `buildspec.yml`

- It is used to customize the build process
- It has to be located in root folder of the repository (a different name/location can be set in the project, e.g. `buildspec-prod.yml`)
- It can contain four main phases:
    - `install`: used to install packages in the build environment
    - `pre_build`: sign-in to things or install code dependencies
    - `build`: commands run during the build process
    - `post_build`: used for packaging artifacts, push docker images, explicit notifications
- It can contain environment variables: shell, variables, parameter-store, secret-manager variables
- `Artifacts` part of the file: specifies what stuff to put where
- Other sections: `reports`, `cache`, `batch`; `finally` blocks run even if a phase fails
- Never put secrets in plain `env.variables` - use `parameter-store`/`secrets-manager` <span style="color: red;">EXAM</span>

## AWS CodeDeploy

- Is a code deployment as a service product
- It is an alternative for third-party services such as Jenkins, Ansible, Chef, Puppet or even CloudFormation
- It is used to deploy code, not resources (use CloudFormation for that)
- CodeDeploy can deploy code to EC2, on-premises, Lambda and ECS
- Besides code, it can deploy configurations, executables, packages, scripts, media and many more
- CodeDeploy integrates with other AWS services such as KMS, IAM, VPC, CloudTrail, S3
- In order to deploy code on EC2 and on-premises, CodeDeploy requires the presence of an agent (can be installed/updated via SSM Distributor). On-premises instances register with an IAM user or IAM role (recommended: IAM Roles Anywhere / STS)
- Revisions stored in S3 or GitHub
- Concepts: **Application** -> **Deployment group** (target set by tags, ASG, ECS service or Lambda alias) -> **Deployment configuration** -> **Revision**

### Deployment types and configurations

- **EC2/on-premises**:
    - In-place: instances are stopped/updated in place, uses `CodeDeployDefault.AllAtOnce`, `HalfAtATime`, `OneAtATime` or custom (min healthy hosts)
    - Blue/green: new instances (copy of ASG or existing tagged instances) behind ELB, traffic re-routed, old instances terminated or kept
- **Lambda** (always blue/green style - traffic shifting on an alias):
    - `Canary` (e.g. `LambdaCanary10Percent5Minutes`), `Linear` (e.g. `LambdaLinear10PercentEvery1Minute`), `AllAtOnce`
- **ECS** (blue/green with ALB/NLB, two target groups, production + optional test listener):
    - `ECSCanary10Percent5Minutes`, `ECSLinear10PercentEvery1Minutes`, `ECSAllAtOnce`
    - Since July 2025 ECS also has **native blue/green** (and since Oct 2025 native linear/canary) deployments in the ECS service itself, with bake time, Lambda lifecycle hooks and alarm-based rollback - no CodeDeploy needed
- Rollback: automatic on deployment failure or when a CloudWatch alarm fires (redeploys last known good revision as a new deployment) <span style="color: red;">EXAM</span>
- Triggers/notifications: SNS, EventBridge

### `appspec.[yaml|json]`

- It controls how deployments occur on the target
- Manages deployments: configurations + lifecycle event hooks
- Configuration section - has 3 important sections:
    - **Files**: applies to EC2/on-premises. Provides information about which files should be installed on the instance
    - **Resources**: applies to ECS/Lambda. For Lambda it contains the name, alias, current version and target version of a Lambda function. For ECS contains things like the task definition and container details (ports, traffic routing)
    - **Permissions**: applies to EC2/on-premises. Details any special permissions and how should be applies to files and folders from the files sections
- Lifecycle event hooks:
    - `ApplicationStop`: happens before the application is downloaded. Used for gracefully stop the application
    - `DownloadBundle`: agent copies the application to a temp location
    - `BeforeInstall`: used for pre-installation tasks
    - `Install`: agent copies the application from the temp folder to the final location
    - `AfterInstall`: perform post-install steps
    - `ApplicationStart`: used to restart/start services which were stopped during the `ApplicationStop` hook
    - `ValidateService`: verify the deployment was completed successfully<span style="color: red;">Remember for EXAM</span>
    - `DownloadBundle` and `Install` are reserved for the agent - they can't run scripts
    - With a load balancer (in-place or blue/green) extra hooks: `BeforeBlockTraffic`, `BlockTraffic`, `AfterBlockTraffic`, `BeforeAllowTraffic`, `AllowTraffic`, `AfterAllowTraffic`
- Lambda hooks: `BeforeAllowTraffic`, `AfterAllowTraffic` (hook = Lambda function that validates and calls `PutLifecycleEventHookExecutionStatus`)
- ECS hooks: `BeforeInstall`, `AfterInstall`, `AfterAllowTestTraffic`, `BeforeAllowTraffic`, `AfterAllowTraffic`

## Related services

- **AWS CodeArtifact**: managed artifact repository (npm, PyPI, Maven, NuGet, etc.), upstream repos + external connections to public registries, domain shared across accounts
- **AWS CodeConnections** (formerly CodeStar Connections): OAuth/app connection to GitHub, GitLab, Bitbucket used by CodePipeline, CodeBuild, CloudFormation Git sync
- **Amazon CodeGuru Reviewer** (closed to new customers Nov 2025) - use Amazon Inspector code security / Amazon Q Developer instead
