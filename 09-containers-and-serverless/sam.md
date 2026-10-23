# AWS SAM - Serverless Application Model

- A serverless application is not just a Lambda function, it can include many more services such as:
    - Front end code and assets hosted by S3 and CloudFront
    - API endpoint - API Gateway
    - Compute - Lambda
    - Database - DynamoDB
    - Event sources, permissions and more...
- The SAM group of products and features within AWS has 2 main parts:
    - AWS SAM template specification, which is an extension of CloudFormation. Adds components to CloudFormation templates designed specifically for serverless applications: Transforms, Globals & Resources (can include normal CFN resources as well as SAM specific ones)
    - AWS SAM CLI allows local testing, local invocation and deployments into AWS

## SAM Template

- `Transform: AWS::Serverless-2016-10-31` => tells CloudFormation to expand SAM resources into regular CFN resources (a CFN macro)
- `Globals` section: common properties for all functions/APIs (runtime, timeout, memory, env vars, tracing)
- SAM resource types:
    - `AWS::Serverless::Function` (Lambda + role + event sources in `Events`)
    - `AWS::Serverless::Api` (REST API), `AWS::Serverless::HttpApi` (HTTP API)
    - `AWS::Serverless::SimpleTable` (DynamoDB with simple primary key)
    - `AWS::Serverless::StateMachine` (Step Functions)
    - `AWS::Serverless::LayerVersion`, `AWS::Serverless::Application` (nested app from the **Serverless Application Repository** / SAR)
    - `AWS::Serverless::GraphQLApi` (AppSync), `AWS::Serverless::Connector` (generates least-privilege IAM between resources)
- **Policy templates** (e.g. `DynamoDBCrudPolicy`, `S3ReadPolicy`, `SQSPollerPolicy`) for least privilege without writing IAM JSON
- `AutoPublishAlias` + **`DeploymentPreference`**: gradual Lambda deployments through **CodeDeploy** (`Canary10Percent5Minutes`, `Linear10PercentEvery1Minute`, `AllAtOnce`) with `PreTraffic`/`PostTraffic` hook functions and CloudWatch alarms for automatic rollback

## SAM CLI

- `sam init` (templates), `sam build`, `sam deploy --guided` (packages artifacts to S3/ECR and deploys the CFN stack via change sets)
- `sam local invoke` / `sam local start-api` / `sam local start-lambda`: run functions and APIs locally in Docker containers (uses the Lambda Runtime Interface Emulator)
- `sam sync --watch`: fast iterative deploys to a dev stack in the cloud (bypasses full CFN deploy for code changes) - **dev only**
- `sam pipeline init`: generates CI/CD pipelines (CodePipeline, GitHub Actions, GitLab, Jenkins) with multi-account stages
- `sam logs`, `sam traces`, `sam remote invoke`; SAM CLI also supports Terraform-defined Lambda projects for local testing

## SAM vs CDK vs CloudFormation (Exam)

- **SAM**: YAML/JSON, CloudFormation superset focused on serverless, shorthand syntax, local testing
- **CDK**: imperative languages (TypeScript, Python, Java...) synthesized to CloudFormation; better for large/complex infra and reusable constructs (CDK can also be tested locally with SAM CLI)
- **CloudFormation**: underlying engine for both; StackSets for multi-account/region deployments
- Exam: "serverless app, simplified IaC, gradual Lambda deployment with automatic rollback" => SAM + CodeDeploy (`DeploymentPreference`)
