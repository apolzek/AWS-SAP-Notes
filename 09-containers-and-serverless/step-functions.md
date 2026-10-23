# AWS Step Functions

- Step Functions address limitations of Lambda product
- AWS Step Functions = name of the service **VS** State machine = execution of the workflow itself created in the Step Functions service
- A Lambda function has an execution time of maximum 15 minutes
- Lambda functions can be chained together, but it is considered to be an anti-pattern and it can get messy. Lambda runtime environments are stateless
- States can do things, can decide things, can take in data, modify data and output data
- Maximum duration for state machine execution is **1 year**
- Step Functions can represent 2 different type of workflow:
  - Standard workflow: **default**, and it has a 1 year execution limit
  - Express workflow: designed for high volume event processing (IoT, streaming data processing and transformations), which can run **up to 5 min**
- State machines can be initiated with API Gateway, IoT Rules, EventBridge, Lambda or even manually
- State machines for Step Functions can be created with Amazon State Language (ASL) JSON templates (or visually with **Workflow Studio**, also in VS Code / Infrastructure Composer)
- IAM Roles used for permissions

## States

- Type of states available for a state machine:
  - `SUCCEED` and `FAIL`
  - `PASS`: passes input to output, optionally injecting/transforming data (useful for mocking)
  - `WAIT`: will wait for a certain period of time or until a date
  - `CHOICE`: allows a state machine to take a different path depending on an input
  - `PARALLEL`: allows to create parallel branches in a state machine
  - `MAP`: expects a list of things, for each it will perform a specific task
  - `TASK`: represents a single unit of work performed by a state machine. It can be integrated with: Lambda, Batch, DynamoDB, ECS, SNS, SQS, Glue, SageMaker, EMR, other Step Functions, etc.
    - Optimized integrations for common services + **AWS SDK integrations** for 200+ services / 10,000+ API actions (`arn:aws:states:::aws-sdk:service:apiAction`)
    - **Activities**: worker code anywhere (EC2, on-prem) polls `GetActivityTask` and reports back
    - **HTTP Task**: call any public HTTPS API (auth via EventBridge connections, max 60s), or private APIs via VPC Lattice

## Standard vs Express Workflows

- **Standard**:
  - Up to **1 year**, **exactly-once** execution of each step
  - Priced **per state transition**
  - Execution history kept 90 days in the service (25,000 events max per execution => use child executions for long loops)
  - Supports `.sync` and `.waitForTaskToken` patterns, redrive
  - Use for: long-running, auditable, non-idempotent business processes (orders, approvals, ETL orchestration)
- **Express**:
  - Up to **5 minutes**, priced per request + duration/memory (much cheaper at high volume), very high rates (6,000 `StartExecution`/s default vs 300/s for Standard in big regions, unlimited state transitions)
  - **Asynchronous** Express: at-least-once; **Synchronous** Express (`StartSyncExecution`, e.g. from API Gateway): at-most-once, returns result
  - History only via CloudWatch Logs; no `.sync` / `.waitForTaskToken`
  - Use for: high-volume event processing, IoT ingestion, streaming transforms, microservice orchestration behind APIs
- Can combine: Standard parent with Express child workflows for high-volume inner steps

## Service Integration Patterns

- **Request Response** (default): call the service and move on immediately
- **Run a Job** (`.sync`): wait for the job to complete (Batch, ECS `RunTask`, Glue, EMR, CodeBuild, nested Step Functions, etc.)
- **Wait for Callback** (`.waitForTaskToken`): pass a task token to SQS/SNS/Lambda/ECS..., pause until `SendTaskSuccess`/`SendTaskFailure` (human approval, external systems, third-party). Heartbeats with `SendTaskHeartbeat`

## Error Handling

- `Retry` (per error type, `IntervalSeconds`, `BackoffRate`, `MaxAttempts`, `MaxDelaySeconds`, jitter) and `Catch` (fallback state) on Task, Parallel and Map states
- Built-in errors: `States.ALL`, `States.Timeout`, `States.TaskFailed`, `States.Permissions`, etc.; `TimeoutSeconds` / `HeartbeatSeconds` on tasks
- **Redrive** failed Standard executions from the failed step (within 14 days) without rerunning successful steps

## Map State: Inline vs Distributed

- **Inline Map**: up to 40 concurrent iterations, items from the state input (payload limit applies)
- **Distributed Map**: up to **10,000 parallel child executions**, reads items directly from **S3** (objects listing, CSV, JSON, JSONL, Parquet, manifests) - millions of items; `ItemBatcher`, tolerated failure threshold, `ResultWriter` to S3
- Exam: "process millions of S3 objects in parallel serverless" => Distributed Map (+ Lambda)

## Data Handling

- Max payload **256 KiB** per state input/output => store large data in S3/DynamoDB and pass references
- Classic JSONPath fields: `InputPath`, `Parameters`, `ResultSelector`, `ResultPath`, `OutputPath`; intrinsic functions (`States.Format`, `States.UUID`, `States.Array`...)
- **JSONata** query language (Nov 2024) as an alternative: `Arguments` and `Output` fields with `{% %}` expressions, much simpler transformations
- **Variables** (`Assign`): store data in one state and reuse it in later states without threading it through the payload

## Operations and Security

- IAM execution role per state machine (least privilege per integration)
- Versions and aliases for safe deployments (weighted traffic between versions)
- `TestState` API to unit test a single state; X-Ray tracing; CloudWatch metrics/logs
- Triggers: API Gateway, EventBridge rules/Scheduler/Pipes, Lambda, SDK, other state machines
- KMS customer managed keys for data at rest

## Step Functions vs Others (Exam)

- vs **Lambda chaining**: anti-pattern; Step Functions adds retries, state, visibility, timeouts
- vs **Lambda durable functions** (re:Invent 2025): workflow written in function code with checkpoints and waits up to 1 year; Step Functions = visual, low-code orchestration across 200+ services
- vs **SWF**: Step Functions is the default; SWF only for legacy or very specific decider logic
- vs **EventBridge Pipes**: Pipes = simple point-to-point source => target; Step Functions = multi-step logic
- vs **MWAA (Airflow)**: Airflow for data pipelines needing open-source DAGs/operators; Step Functions for serverless AWS-native orchestration
