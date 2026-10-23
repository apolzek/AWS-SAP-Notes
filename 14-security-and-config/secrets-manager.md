# AWS Secrets Manager

- It does share functionality with SSM Parameter Store
- Secrets Manager is designed specifically for secrets, example passwords, API Keys
- It is usable via Console, CLI, API or SDK's
- It supports the automatic rotation of secrets using a Lambda functions
- For certain AWS services, Secrets Manager offers direct integration, such as RDS (automatic synchronization when the secrets are rotated)
- Secrets are encrypted using KMS

## Rotation

- Rotation uses a Lambda function (AWS provides templates for RDS, Aurora, Redshift, DocumentDB, etc.) - Lambda needs network access to both the database and the Secrets Manager endpoint (VPC interface endpoint or NAT) <span style="color: red;">EXAM</span>
- Schedule: every N days/hours or `cron`/`rate` expression, minimum every **4 hours**; optional rotation window
- Rotation steps: `createSecret` -> `setSecret` -> `testSecret` -> `finishSecret`; staging labels `AWSPENDING`, `AWSCURRENT`, `AWSPREVIOUS`
- Strategies: **single user** (update the same user's password - brief window where connections may fail) vs **alternating users** (clone user, switch between two users - no downtime, needs a superuser secret)
- **Managed rotation**: some services own and rotate the secret themselves, no Lambda to manage - e.g. RDS/Aurora `ManageMasterUserPassword` (master password in Secrets Manager, rotated every 7 days by default), Redshift Serverless/provisioned admin password, etc.
- Rotate immediately on demand (`RotateSecret`)

## Other Features

- Secret value up to 64 KB, stored as key/value JSON or binary, versioned
- Encryption: KMS - AWS managed `aws/secretsmanager` key or **customer managed key** (required for cross-account access)
- **Resource-based policies** on secrets: allow other accounts/roles to `GetSecretValue` (+ key policy on the CMK) <span style="color: red;">EXAM</span>
- **Multi-region secrets**: replicate a secret to other regions (replicas kept in sync, including rotation) - for DR and multi-region apps; a replica can be promoted to a standalone secret
- `BatchGetSecretValue` to retrieve many secrets in one call; client-side caching libraries, **Secrets Manager Agent** (local HTTP cache for EC2/containers) and the **Parameters and Secrets Lambda Extension** reduce API calls/cost
- Integrations: CloudFormation dynamic references (`{{resolve:secretsmanager:...}}`) and `AWS::SecretsManager::RotationSchedule`, ECS/EKS (Secrets Store CSI driver), Lambda, RDS Proxy (uses secrets for DB auth), CodeBuild, EventBridge events on rotation
- Monitor/compliance: CloudTrail, Config rules (`secretsmanager-rotation-enabled-check`, `secretsmanager-secret-unused`), Security Hub
- Deletion: scheduled with a 7 - 30 day recovery window (can force delete without recovery via API)
- Pricing: ~$0.40 per secret per month (replicas charged as secrets) + $0.05 per 10,000 API calls

## Exam Scenarios

- "Database credentials must be rotated automatically every 30 days with no app changes" -> Secrets Manager rotation (app retrieves secret at runtime)
- "Credentials needed in DR region" -> replicate the secret to the DR region
- "Cheapest way to store non-rotating config/strings" -> Parameter Store standard tier
- "Hard-coded credentials in code" -> Secrets Manager + IAM role; detect with Amazon Inspector code scanning / CodeGuru (legacy)
