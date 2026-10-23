# AWS Systems Manager Parameter Store

- Used to store system configurations (documents, configurations, secrets, strings) in a resilient, secure and scalable way
- Stores data in a key-value format
- Many AWS services have native integration with Parameter Store
- Parameter store offers the availability to store 3 different type of values: Strings, StringLists and SecureStrings
- We can store license codes, database strings, full configs and passwords in Parameter Store
- Values can be stored in a hierarchical way. Different versions of the values are also stored
- Parameter Store can store plaintext and ciphertext which can be decrypted using the KMS integration
- Public Parameters: parameters maintained and provided by AWS, example: latest AMIs per region
- Parameter Store is public service
- Parameter Store is tightly integrated with IAM

## Tiers

| | Standard | Advanced |
|---|---|---|
| Max parameters per account/region | 10,000 | 100,000 |
| Max value size | 4 KB | 8 KB |
| Parameter policies (expiration, notifications) | No | Yes |
| Share with other accounts (AWS RAM) | No | Yes |
| Cost | Free (storage) | $0.05 per parameter/month |

- Standard -> Advanced upgrade is possible; Advanced -> Standard is not (delete and recreate)
- **Intelligent-Tiering** option: automatically uses Advanced only when needed
- Throughput: default (standard) throughput ~40 TPS for `GetParameter*`; enable **higher throughput** (up to 10,000 TPS, charged per API interaction) - fix for `ThrottlingException` <span style="color: red;">EXAM</span>
- Parameter policies (Advanced only): `Expiration` (delete at a date), `ExpirationNotification` and `NoChangeNotification` (EventBridge events)

## Other Details

- `SecureString` uses KMS (AWS managed `aws/ssm` key or a customer managed key - use a CMK for cross-account access); decrypt with `WithDecryption=true` - requires `kms:Decrypt`
- Hierarchies (`/app/prod/db/url`) + `GetParametersByPath` (recursive) + IAM policies scoped to paths
- Versions and labels (e.g. label `prod` points to a specific version); history kept (last 100 versions)
- **No native automatic rotation** - unlike Secrets Manager (could be built with EventBridge + Lambda)
- Can reference Secrets Manager secrets through Parameter Store (`/aws/reference/secretsmanager/<secret>`)
- Public parameters: `/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64`, `/aws/service/global-infrastructure/regions` - commonly used in CloudFormation `AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>` for latest AMI <span style="color: red;">EXAM</span>
- Native integrations: CloudFormation dynamic references (`{{resolve:ssm:...}}`, `{{resolve:ssm-secure:...}}`), ECS/EKS task secrets, Lambda (Parameters and Secrets Lambda Extension caches values), CodeBuild, EC2 user data via CLI
- Changes emit EventBridge events (e.g. trigger redeploy on config change)
- Cross-region: no replication - copy values with automation if needed

## Parameter Store vs Secrets Manager vs AppConfig

- **Parameter Store**: cheap/free config + simple secrets, hierarchies, no rotation, no cross-region replication
- **Secrets Manager**: secrets needing **automatic rotation**, cross-region replication, RDS/Redshift/DocumentDB integration, resource policies for cross-account (~$0.40/secret/month)
- **AppConfig**: feature flags/dynamic config with validation, gradual rollout and automatic rollback
