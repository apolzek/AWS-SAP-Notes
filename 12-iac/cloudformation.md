# CloudFormation

## Physical and Logical Resources

- CloudFormation begins with a template defined in YAML or JSON file
- The template contains logical resources (what we want to create)
- Templates can be used to create CloudFormation Stacks (one ore many stacks)
- The initial job of a stack is to create physical resources based on the logical resources defined in the template
- If a stack's template are changed, physical resources are changed as well
- If a stack is deleted, normally the physical resources are deleted
- Template sections: `AWSTemplateFormatVersion`, `Description`, `Metadata`, `Parameters`, `Rules`, `Mappings`, `Conditions`, `Transform`, `Resources` (only mandatory section), `Outputs`

## Key Quotas

- Resources per stack: 500
- Parameters, outputs, mappings per template: 200 each
- Template body: 51,200 bytes inline (API/CLI), 1 MB when uploaded via S3
- Stacks per account per region: 2,000 (adjustable)
- Exports must be unique per account per region
- Exceeding limits => split into nested stacks / multiple stacks, or use stack refactoring

![CFN Resources](images/CFNResources.png)

## Stacks

- A stack is a collection of AWS resources that you can manage as a single unit
- All the resources in a stack are defined by the stack's CloudFormation template
- Stack options:
  - Tags: key/value pairs attached to the stack. Can be used to identify the stack for cost allocation purposes
  - Permissions: IAM service role that can be assumed by CloudFormation
  - Stack failure options: specifies what to do if something fails while the stack is provisioned
    - Options:
      - Rollback all stack resources
      - Preserve successfully provisioned resources
  - Stack policy: defines the resources that we want to protect from unintentional updates during a stack update
  - Rollback configuration: we can monitor the stack while it is being created/updated and we can roll it back in case a threshold is breached (example if any alarm goes to ALARM state)
  - Notification options: we can specify an SNS topic where notifications should go
  - Stack creation options: following options are included for stack creation, but aren't available as part of stack updates:
    - Timeout: Specifies the amount of time, in minutes, that CloudFormation should allot before timing out stack creation operations
    - Termination protection: prevents the stack from being deleted accidentally
- Stack policy: JSON document with `Allow`/`Deny` on `Update:Modify`, `Update:Replace`, `Update:Delete` per resource. Once set, all resources are protected by default unless explicitly allowed. Can be overridden temporarily during an update
- Rollback triggers: up to 5 CloudWatch alarms monitored during create/update + monitoring period (0-180 min) after the operation
- `--disable-rollback` / "preserve successfully provisioned resources" + `ContinueUpdateRollback` for stacks stuck in `UPDATE_ROLLBACK_FAILED`

## Template Parameters and Pseudo Parameters

- Template parameters allow input via the console, CLI or API when the stack is created or updated
- Parameters are defined in the `Parameters` section of the template and they can be referenced from within the logical resources (`Ref`)
- Parameters can have default values, allowed values, min/max length, allowed patterns, no echo (useful for passwords, the value is not displayed when typed) and types

  ![Parameters](images/Parameters.png)

- Pseudo Parameters:
  - AWS makes available parameters which can be referenced by the CF template
  - Example:
    - `AWS::Region`
    - `AWS::StackId`
    - `AWS::StackName`
    - `AWS::AccountId`
  - Pseudo parameters are parameters which can not be populated by us, they are populated by AWS and provided for us to reference them
- Parameters provide portability for the template

  ![PseudoParameters](images/PseudoParameters.png)

- Other pseudo parameters: `AWS::Partition`, `AWS::URLSuffix`, `AWS::NotificationARNs`, `AWS::NoValue` (removes a property when used with `Fn::If`)
- AWS-specific parameter types: `AWS::EC2::KeyPair::KeyName`, `AWS::EC2::VPC::Id`, `List<AWS::EC2::Subnet::Id>`, and SSM parameter types `AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>` (e.g. latest AMI from public SSM parameters)
- **Dynamic references**: `{{resolve:ssm:name}}`, `{{resolve:ssm-secure:name}}`, `{{resolve:secretsmanager:secret-id:SecretString:json-key}}` => retrieve values/secrets at deploy time without putting them in the template
- `ManageMasterUserPassword` (RDS) or `AWS::SecretsManager::Secret` with `GenerateSecretString` => avoid passwords as parameters
- Best practice:
  - Minimize number of parameters and provide defaults where applicable
  - Use pseudo parameters where possible

## Intrinsic Functions

- Intrinsic functions can be used in templates to assign values to properties that are not available until runtime
- Examples of functions:
  - `Ref` and `Fn::GetAtt`: reference a value from one logical resource
  - `Fn::Join` and `Fn::Split`: join/split strings to create new ones
  - `Fn::GetAZs` and `Fn::Select`: get availability zones in a regions and select one
  - Conditions: `Fn::IF`, `And`, `Equals`, `Not`, `Or`
  - `Fn::Base64` and `Fn::Sub`: encode strings to base64, substitute replacement on variables in the text
  - `Fn::Cidr`: build CIDR blocks
  - `Fn::ImportValue`: import an exported output of another stack
  - `Fn::FindInMap`: lookup in Mappings
  - `Fn::ToJsonString`, `Fn::Length`, `Fn::ForEach` (loops): available with the `AWS::LanguageExtensions` transform
- `Fn::GetAZs` - returns the available AZs in region. If the region has a default VPC configured, it return the AZs which are available in the default VPC
  ![FnGetAtt](images/FnGetAtt.png)
  ![FnSelect](images/FnSelect.png)
  ![FnJoinSplit](images/FnJoinSplit.png)
  ![FnBase64](images/FnBase64.png)
  ![FnCidr](images/FnCidr.png)

## Mappings

- Templates can contain a `Mappings` objects which can contain many mappings
- A mapping maps keys to values, allowed lookup
- Mappings can have one key or top & second level keys
- Mappings use another intrinsic function `Fn::FindInMap`
- Mappings => improve template portability
- Example:

  ```
  Mappings:
      RegionMap:
          us-east-1:
              HVM64: 'ami-xxx'
              HVMG2: 'ami-yyy'
          us-east-2:
              HVM64: 'ami-zzz'
              HVMG2: 'ami-vvv'
      ...
  ```

  ![Mappings](images/Mappings.png)

## Outputs

- The `Outputs` section in a template is optional
- We can declare values in this section which will be visible as output in the CLI/Console
- Output will be **accessible from a parent stack** when using nesting
- Outputs can be exported allowing cross-stack references
- Example:
  ```
  Outputs:
      WordPressUrl:
          Description: 'description text'
          Value: !Join ['', ['https://', !GetAtt Instance.PublicDnsName]]
          Export:
              Name: !Sub '${AWS::StackName}-WordPressUrl'
      ...
  ```

## Conditions

- Allows stack to react to certain conditions and change infrastructure based on those
- They are declared in an optional section named `Conditions`
- We can declare many conditions, each of them being evaluated to `TRUE` or `FALSE`
- Conditions are evaluated before any resource is created
- Conditions use other intrinsic functions: `AND`, `EQUALS`, `IF`, `NOT`, `OR`
- Any resource can have an associated condition which will define if the resource will be created or not
- Examples: we can have conditions which evaluate based on the environment (dev, test, prod) in which the template is executed
- Condition example:
  ```
  Conditions:
      IsProd: !Equals
          - !Ref EnvType
          - 'prod'
  ```
- Conditions can be nested

![CFNConditions](images/CFNConditions.png)

## DependsOn

- Allows us to establish dependencies between resources
- CFN tries to be efficient by creating/updating/deleting resources in parallel
- Also, it tries to determine a dependency order (example: VPC => SUBNET => EC2) by using references or functions
- Dependency can be defined using the `DependsOn` property to specify which other resources a resource depends on
- `DependsOn` can accept a single resource or a list of resources
- Classic use case: EC2 instance with public IP / EIP must depend on the `AWS::EC2::VPCGatewayAttachment` (no implicit reference exists)

![CFNDependsOn](images/CFNDependsOn.png)

## Creation Policies, Wait Conditions and cfn-signal

- Creation Policies, Wait Conditions and cfn-signals provide a few ways to notify CFN with details signals on completion or not of creation of resources
- We can configure CFN to wait for a certain number of success signals
- We also configure a timeout within which the signals are received (max 12H)
- The the number of success signals are received within the timeout, CFN stacks moves to `CREATE_COMPLETE`
- `cfn-signal` is an utility running on the EC2 instances sending success/failure signals to CFN
- If the timeout is reached and the number of success signals are not met, the stack will fail creation
- For provisioning EC2 and ASG, we should us a `CreationPolicy`
  - `ResourceSignal`: `Count` + `Timeout` (ISO 8601, e.g. `PT15M`)
  - ASG: `AutoScalingCreationPolicy.MinSuccessfulInstancesPercent`
  - Typical user data: `cfn-init ... && cfn-signal -e $? --stack ${AWS::StackName} --resource MyASG --region ${AWS::Region}`
- `CreationPolicy` is supported on `AWS::EC2::Instance`, `AWS::AutoScaling::AutoScalingGroup`, `AWS::CloudFormation::WaitCondition` and `AWS::AppStream::Fleet` (`StartFleet`)

  ![CreationPolicy](images/CreationPolicy.png)

- For other requirements we might chose to use a `WaitCondition`
- A `WaitCondition` is defined as a logical resource, meaning it can have `DependsOn` property. It can be used as a general progress gait in the template
- A `WaitCondition` relies on a `WaitHandle`, which is another logical resource. Its job is to generate a presigned url which can be used to send signals to `WaitCondition`
- With `WaitHandle` we can pass back data to the template. This data can be retrieved using the `!GetAtt WaitCondition.Data` function

  ![WaitCondition](images/WaitCondition.png)

## Nested Stacks

- Most simple projects will generally utilize a CFN stack
- Stacks can have limits:
  - Resource limit: 500 resources per stack
  - We can't easily reuse resources, example reference a VPC
- There are 2 ways to architect multi-stack projects:
  - Nested Stacks
  - Cross-Stack References
- Nested Stacks:
  - Root Stack: the stack which is created first, created manually or using some automation
  - A Parent Stack is the parent of any stack which it immediately creates
  - A root stack can create nested stacks having several parent stacks
  - A root stack can have parameters and outputs (just like a normals stack)
- A stack can have another CFN stack as a resource using `AWS::CloudFormation::Stack` type which needs an url to the template
- We can provide input values to the nested stacks. We need to supply values to any parameters from a nested stack if the parameter does not have a default value defined
- Any outputs of a nested stack are returned to the root stack which can be referenced as `NESTEDStack.Outputs.XXX`
- Benefits of a nested stack is to reuse the same template, not the actual stack created
- We should use nested stacks when we want to link the life cycles of different stacks
- Updates are always initiated from the root stack; nested stacks count as resources (each nested stack has its own 500 resource limit)
- Change sets on the root stack can include nested stacks

  ![Nested Stacks](images/NestedStacks.png)

## Cross-Stack References

- CFN stacks are designed to be isolated and self-contained
- With nested stacks we can reuse code only, with cross-stack references we can reference resources created by other stacks
- Outputs are normally not visible from other stacks, exception being nested stacks which case the parent stack can reference the nested stack output
- Outputs of a template can be exported making them visible from other stacks
- Exports must have unique names in the region
- To use the exported resources we can use `Fn::ImportValue` intrinsic function
- Cross-region or cross-account is not supported for cross-stack references
- An export can't be deleted or modified while another stack imports it => the importing stacks must be updated first (tight coupling)
- For cross-account/cross-region sharing use SSM Parameter Store (with dynamic references, RAM sharing for advanced parameters), custom resources, or CDK `crossRegionReferences`
- Nested vs cross-stack (exam):
  - Nested: reuse templates/components, single lifecycle, deployed together (e.g. same ALB module in many apps)
  - Cross-stack: share long-lived resources owned by another team/stack with independent lifecycles (e.g. network stack exporting VPC/subnet IDs for app stacks)

  ![CrossStackRef](images/CrossStackRef.png)

## StackSets

- Allows to create/update/delete stacks across many regions or many accounts
- StackSets are containers in an admin account (account where the StackSet is applied: any account for self-managed permissions; the Organizations management account or a **delegated administrator** account for service-managed permissions)
- StackSets contain stack instances (= container for an individual stack) which reference stacks
- Stack instances and stacks are created in target accounts
- Each stack created by a StackSet is a stack created in one region in one account
- Security: we can use self-managed roles or service-managed roles within an Org (everything handled by the product). CFN will assume a role to interact with the target accounts
  - Self-managed: create `AWSCloudFormationStackSetAdministrationRole` in admin account and `AWSCloudFormationStackSetExecutionRole` in each target account
  - Service-managed: requires trusted access with AWS Organizations; target **OUs**; roles are created automatically
- **Automatic deployment** (service-managed only): new accounts added to a target OU automatically get the stack instances; removed accounts can have their stacks deleted or retained
- **Deployment ordering** (Nov 2025): dependencies between stack instances (up to 10 per instance) in auto-deployment so foundational stacks (network/security) deploy first
- Deployment options: region order, maximum concurrent accounts (number or %), failure tolerance, region concurrency (sequential/parallel), concurrency mode (strict vs soft failure tolerance)
- Parameter overrides per account/region
- Drift detection supported on StackSets
- A template for a StackSet is a normal CloudFormation template
- Terminology:
  - **Concurrent Accounts**: a value that you can set that specifies in how many accounts it can be deployed at the same time
  - **Failure Tolerance**: amount of individual deployments which can fail before declaring the StackSet itself as failed
  - **_Retain Stacks_**: remove stack instances from a StackSet to retain the infrastructure. When a StackSet is deleted, it deletes all the stacks by default.
- StackSet use cases:
  - Crate AWS Config Rules (MFA, Elastic IPs, EBS Encryption)
  - Create IAM Roles for cross-account access at scale
  - Baseline guardrails/logging (CloudTrail, GuardDuty, Config recorders) in every new account
  - Control Tower uses StackSets under the hood for account customizations (CfCT)

  ![StackSets](images/StackSets.png)

## DeletionPolicy

- If we delete a logical resource from a template, by default the physical resource will be deleted by CFN
- With certain type of resources this can cause data loss
- With deletion policy, we can define on each resource:
  - **Delete**: default
  - **Retain**: physical resources remain untouched
  - **RetainExceptOnCreate** (2023): retain on stack delete, but delete if the resource was created during a create operation that rolled back (avoids orphaned resources after failed creates)
  - **Snapshot**: only supported on EBS volumes (`AWS::EC2::Volume`), ElastiCache (CacheCluster, ReplicationGroup), Neptune, RDS (DBInstance, DBCluster), Redshift, DocumentDB. A snapshot is taken before the physical resource is deleted. Snapshots then persist past the Stack lifetime => you have to clean up ($)
- Deletion policies only apply to delete operation, NOT replace operation!
- **UpdateReplacePolicy**: same options (`Delete`, `Retain`, `Snapshot`) but applies when an update causes **replacement** of the resource
- Exam trap: S3 bucket deletion fails if the bucket is not empty (use a custom resource to empty it or `Retain`)

  ![DeletionPolicy](images/DeletionPolicy.png)

## Stack Roles

- By default CFN uses the permissions of the logged in identity who initiates the stack creation
- CFN stack roles is a feature where CFN can assume a role to gain permissions to create resources from a stack without the need for the initiator to have the necessary permissions
- Allows role separation
- The identity creating the stack does not need resource permissions, only `iam:PassRole` (to pass the role to CloudFormation)
- Combine with Service Catalog: end users launch approved products (CFN templates) using a launch constraint role, without direct permissions
- Condition key `cloudformation:RoleArn` can force users to always use a specific service role

  ![Stack Roles](images/StackRoles.png)

## `AWS::CloudFormation::Init` and `cfn-init`

- CloudFormationInit is a simple configuration management system
- Configuration directive are stored in the template
- `AWS::CloudFormation::Init` is part of EC2 instance logical resource. With this we can specify configurations which will be applied to the created EC2 instance
- UserData is procedural (HOW should things to be done)

VS

- `cfn-init` is a desired state (WHAT we want to occur)
- `cfn-init` is idempotent and can be cross-platform
- `cfn-init` is a helper script which is installed on the EC2 OS (`/opt/aws/bin/cfn-init`)
- Config keys (executed in this order): `packages`, `groups`, `users`, `sources`, `files`, `commands`, `services`; `configSets` group configs and define order
- Logs: `/var/log/cfn-init.log` and `/var/log/cfn-init-cmd.log` (troubleshooting: disable rollback to inspect the instance)
- Helper scripts: `cfn-init`, `cfn-signal`, `cfn-hup`, `cfn-get-metadata`

  ![cfn-init](images/cfn-init.png)

## `cfn-hup`

- `cfn-init` is a helper tool running only once as part of bootstrapping (user data)
- If the `AWS::CloudFormation::Init` is updated, `cfn-init` is not rerun
- `cfn-hup` is a helper tool which **can be** installed on EC2 instances (our responsibility)
- It will detect changes in the resource metadata
- When change is detected, it can run configurable actions. It might rerun `cfn-init` if necessary (to re-apply the desired state)

  ![cfn-hup](images/cfn-hup.png)

## Change Sets

- Change Sets let us preview the changes that will happen when the stack will be updated (= when the Change Set is applied)
- A stack can have multiple Change Sets in here:

  ![ChangeSets Menu](images/ChangeSetsMenu.png)

- We can have multiple change sets and preview each of them separately to decide which one is the right one
- We can chose which change set we want to apply by executing it
- Change set shows: action (Add/Modify/Remove/Import), logical/physical ID, **Replacement** (True/False/Conditional) - key for avoiding data loss
- Change sets don't tell if the update will succeed (validation errors can still happen)
- **Drift-aware change sets** (2025): compare new template vs last deployed template vs actual live configuration => see and avoid overwriting manual changes (drift)
- **Early validation** (2025): during change set creation, checks invalid properties, resource name conflicts, non-empty S3 bucket deletes
- Change set types: `UPDATE`, `CREATE`, `IMPORT` (import existing resources into a stack)

## Update Behaviors

- **No interruption**: update in place, no downtime (e.g. changing tags)
- **Some interruption**: update in place with brief interruption (e.g. EC2 instance type on EBS-backed instance => reboot)
- **Replacement**: new physical resource created, references updated, old one deleted (e.g. RDS `DBInstanceIdentifier`, EC2 AMI change) => new physical ID, potential data loss
- `UpdatePolicy` attribute:
  - ASG: `AutoScalingRollingUpdate` (batches, `MinInstancesInService`, `MaxBatchSize`, `PauseTime`, `WaitOnResourceSignals`), `AutoScalingReplacingUpdate` (new ASG, `WillReplace`), `AutoScalingScheduledAction`
  - `AWS::Lambda::Alias`: `CodeDeployLambdaAliasUpdate` (traffic shifting)
  - `AWS::ElastiCache::ReplicationGroup`: `UseOnlineResharding`; `AWS::OpenSearchService::Domain`: `EnableVersionUpgrade`
- Blue/green ECS deployments through `AWS::CodeDeployBlueGreen` transform/hook

## Drift Detection

- Detects differences between the expected configuration (template + parameters) and the actual configuration of resources changed outside CloudFormation
- Statuses: `IN_SYNC`, `MODIFIED`, `DELETED`, `NOT_CHECKED`
- Works at stack, resource and StackSet level; not all resource types support it
- Can be automated with AWS Config rule `cloudformation-stack-drift-detection-check` + EventBridge/SNS
- Resolve: revert manually, update the template to match, or use drift-aware change sets / resource import

  ![ChangeSets](images/ChangeSets.png)

## Custom Resources

- CloudFormation doesn't support everything in AWS
- Custom Resources let CFN integrate with anything it does not yet support or won't support at all
- With Custom Resources we can extend CFN to do things which it does not natively support
- Examples:
  - Populate S3 Bucket with objects
  - Integrate with 3rd party to fetch a config, etc
- Architecture of custom resources:
  - CFN sends data to an endpoint defined in the custom resource
  - This endpoint might be a Lambda function or an SNS topic
  - When a custom resources is create/update/delete, CFN sends events to this endpoint containing the operation and any additional property information
  - The compute (Lambda function or SNS topic) can respond to this custom data, letting CloudFormation know about the success/failure of the execution (so it can move into CREATE_COMPLETE or not)

  ![CustomResource](images/CustomResource.png)

- Custom resource details:
  - Type `Custom::MyName` or `AWS::CloudFormation::CustomResource` with property `ServiceToken` (Lambda ARN or SNS topic ARN)
  - Request types: `Create`, `Update`, `Delete`; the function must send the response (SUCCESS/FAILED + `PhysicalResourceId` + `Data`) to the **pre-signed S3 URL** (`ResponseURL`)
  - If no response is sent, the stack waits until timeout (configurable with `ServiceTimeout`, default/max 1 hour) => always handle errors and `Delete` events
  - Returned `Data` can be read with `!GetAtt MyCustom.Attribute`
  - Typical exam use: empty an S3 bucket before deletion, look up AMI IDs, call 3rd party APIs, cross-account/cross-region lookups
- `cfn-response` module helps for inline Lambda (ZipFile) code

## Resource Import and Stack Refactoring

- **Resource import**: bring existing resources into a stack (IMPORT change set); resources need `DeletionPolicy` in the template; useful to adopt manually created resources or move resources between stacks (retain + import)
- **Stack refactoring** (2025): move resources between stacks, split monolithic stacks and rename logical IDs without re-creating resources (CLI/SDK, console and `cdk refactor`)
- **IaC generator** (2024): scan existing resources in the account and generate a CloudFormation template (and CDK app) to manage them; partial scans by resource type (2025)

## Extensions: Registry, Modules, Macros, Hooks

- **CloudFormation Registry**: public (AWS and third-party, e.g. Datadog, MongoDB Atlas) and private extensions; resource types written with the CFN CLI (replace custom resources for reusable cases)
- **Modules**: reusable template fragments registered as `MyOrg::Service::Thing::MODULE`, versioned, used like a resource
- **Macros / Transforms**: Lambda-backed template processing before deployment
  - `AWS::Serverless-2016-10-31` (SAM): `AWS::Serverless::Function`, `Api`, `SimpleTable`
  - `AWS::Include`: include template snippets from S3
  - `AWS::LanguageExtensions`: `Fn::ForEach`, `Fn::Length`, `Fn::ToJsonString`, intrinsic functions in `DeletionPolicy`
  - Templates with transforms must be deployed via change sets (`CAPABILITY_AUTO_EXPAND`)
- **Hooks**: proactive controls invoked before create/update/delete of resources, stacks or change sets => fail or warn on non-compliant resources (e.g. unencrypted S3 bucket)
  - Lambda hooks, **Guard hooks** (CloudFormation Guard rules from S3) (2024)
  - **Managed proactive controls** (2025): use Control Tower control catalog in hooks without code, with warn mode
  - Proactive vs detective: Hooks/Guard block before deployment; AWS Config detects after deployment
- **cfn-lint** (syntax/best practices) and **cfn-guard** (policy as code) in CI/CD pipelines
- Capabilities acknowledgements: `CAPABILITY_IAM`, `CAPABILITY_NAMED_IAM` (custom IAM names), `CAPABILITY_AUTO_EXPAND` (macros/nested with transforms)

## Deployment Options and CI/CD

- **Git sync** (2023): stack automatically updated from a template + deployment file in a Git repository (GitHub, GitLab, Bitbucket) via CodeConnections
- CodePipeline CloudFormation actions: create/update stack, create/execute change set (manual approval between them), StackSets deploy
- `aws cloudformation deploy` (creates and executes a change set), `package` (uploads local artifacts to S3)
- Service Catalog products (governed self-service); Infrastructure Composer (formerly Application Composer) for visual authoring
- AWS IaC MCP server and CloudFormation Language Server (2025) for IDE/AI-assisted authoring

## AWS CDK (Cloud Development Kit)

- Define infrastructure in TypeScript, Python, Java, C#/.NET, Go => `cdk synth` produces CloudFormation templates => deployed as CFN stacks
- Constructs:
  - L1 (`Cfn*`): 1:1 with CloudFormation resources
  - L2: higher-level with sensible defaults and helper methods (`bucket.grantRead(role)`)
  - L3 (patterns): multi-resource architectures (e.g. `ApplicationLoadBalancedFargateService`)
- `cdk bootstrap`: creates the CDKToolkit stack (S3 bucket, ECR repo, IAM roles) per account/region - required before deploying (incl. cross-account trust for pipelines)
- `cdk diff` (like change sets), `cdk deploy`, `cdk destroy`, `cdk watch`/hotswap for dev, `cdk refactor` (2025, preview)
- CDK Pipelines: self-mutating CodePipeline for multi-account/multi-region deployments
- CDK Nag / Aspects for compliance checks
- Exam: CDK vs CFN - CDK when developers prefer programming languages, loops/abstractions, reusable constructs; result is still CloudFormation (same limits, same drift/rollback behavior)
- SAM: CFN transform + CLI focused on serverless (local testing `sam local`, `sam deploy`, gradual Lambda deployments)

## Exam Tips

- Deploy the same baseline to all accounts in an Org including future accounts => StackSets with service-managed permissions + automatic deployment targeting OUs
- Prevent accidental replacement/deletion of a production DB => stack policy + `DeletionPolicy: Snapshot/Retain` + `UpdateReplacePolicy` + termination protection
- Wait for app bootstrap before marking complete => `CreationPolicy` + `cfn-signal`; external process => `WaitCondition`
- Apply config changes to running instances without replacement => `cfn-hup` + `cfn-init`
- Preview impact before update => change sets (look at Replacement column)
- Resources changed manually => drift detection / drift-aware change sets
- Unsupported resource or custom logic => custom resource (Lambda) or registry resource type
- Enforce compliance before deployment => CloudFormation Hooks / Guard; after deployment => AWS Config
- Share VPC IDs across stacks in same region => exports + `Fn::ImportValue`; across accounts/regions => SSM parameters / StackSets
- Secrets in templates => dynamic references to Secrets Manager / SSM SecureString, never plaintext parameters (`NoEcho` only hides the value in console/API output)
- Legacy manually-created infra to IaC => IaC generator + resource import
