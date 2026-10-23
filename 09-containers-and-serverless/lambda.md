# AWS Lambda

- Lambda is a Function-as-a-Service (FaaS) product. We provide specialized short running focused code for Lambda and it will take care running it and billing us for only what we consume
- Every Lambda function uses a supported runtime, example: Python 3.13, Java 21, Node.js 22 (old ones like Python 3.8 / Java 8 / Go 1.x are deprecated - runtimes follow a deprecation schedule, after which we can't create/update functions on them)
- Every Lambda function is loaded into an executed in a runtime environment
- When we create a function we define the resources the function will use:
  - We define the memory directly
  - And CPU usage allocation indirectly (based on the amount of memory)
- We are only billed for the duration the function is running based on the number of invocations and the resources specified
- Lambda is a key part of serverless architectures in AWS
- Lambda has support for the following runtimes:
  - Node.js
  - Python
  - Ruby
  - Java
  - C# (.NET)
  - OS-only runtimes `provided.al2023` / `provided.al2` for compiled languages (Go, Rust, C++) and **custom runtimes** (Runtime API, bundled in the package or a layer)
- Architectures: `x86_64` or `arm64` (Graviton2: up to ~34% better price/performance)
- Lambda deployment package size:
  - 50 MB zipped
  - 250 MB unzipped
  - Up to 10 GB as a Docker image
- Lambda functions are stateless, meaning there is no data left over after an invocation
- When creating a Lambda function we define the memory. The memory can be between 128 MB - 10240 MB in 1 MB steps
- We do not directly define the vCPU allocated to each function, it will automatically scale with the memory: 1769 MB of memory gives 1 vCPU
- The runtime env has a 512 MB (by default) storage available as `/tmp`. We can scale this storage up to 10240 MB. We can use this storage for whatever we need as long as we assume that it is empty at each execution of the function
- Lambda function can run for **up to 15 minutes**, after this a timeout will occur
- The security for a Lambda function is controlled by the execution role. This is an IAM role attached to the function. This can have permissions for integration with other AWS services
- Common uses:
  - Serverless applications (S3, API Gateway, Lambda)
  - File processing (S3, S3 Events, Lambda)
  - Database triggers (DynamoDB, Streams, Lambda)
  - Serverless CRON (EventBridge, CW Events, Lambda)
  - Realtime Stream Data Processing (Kinesis, Lambda)

![Lambda-1](images/Lambda-1.png)

## Lambda Networking

- Lambda functions can have 2 types of networking modes:
  - Public (default):
    - Lambda can access public AWS services such as SQS, DynamoDB, etc. and also internet based services
    - Lambda has network connectivity to public services running on the internet
    - Offers the best performance for Lambda, no customer specific networking is required
    - With public networking mode Lambda function wont be able to access resources in a VPC unless the resources do have public IPs and security controls allow external access
  - VPC Networking:
    - Lambda functions will run inside a VPC, so they will access everything in a VPC, assuming NACLs and SGs allow access
    - They wont be able to access services outside of the VPC, unless networking configuration exists in the VPC to allow external access
    - The Lambda execution role needs EC2 network permissions (`ec2:CreateNetworkInterface`, `DescribeNetworkInterfaces`, `DeleteNetworkInterface` - managed policy `AWSLambdaVPCAccessExecutionRole`) in order ot be able to create ENIs in the VPC
- VPC Lambda has **no internet access and no public IP** even in a public subnet => use NAT Gateway in private subnets for internet, **VPC endpoints** for AWS services (S3/DynamoDB gateway endpoints, interface endpoints for others)
- Use multiple subnets in different AZs for HA; IPv6 dual-stack supported
    - VPC based Lambda functions do not directly in the VPC, they will use a shared **Hyperplane ENI** to access resources in the VPC: one ENI per unique **subnet + security group** combination, shared by all functions using that combination. In case new Security Groups are attached to a certain Lambda, new ENIs are placed inside the VPC
    - At the creation of the function, a certain ENI might be created for accessing the VPC. The initial setup would take up to 90 seconds. This setup will take place only once, not at every invocation

![LambdaNetworking](images/LambdaNetworking.png)

## Lambda Security

- There are 2 key parts of the security model
  - Lambda Functions will assume an execution role in order to access other AWS resources
  - Resource policies: similar to resource policies for S3. Allows external accounts to invoke a Lambda functions, or certain services to use Lambda functions. Resources polices can be modified using the CLI/API or the Console
- Exam rule: **push** sources (S3, SNS, API Gateway, EventBridge) need the Lambda **resource policy**; **poll** sources (SQS, Kinesis, DynamoDB Streams via event source mapping) need permissions in the **execution role**
- Env vars / secrets: prefer Secrets Manager or Parameter Store (with the Parameters and Secrets Lambda Extension for caching) over plain env vars
- **Code signing** (AWS Signer): only allow trusted, signed packages to be deployed
- Function URLs auth: `AWS_IAM` or `NONE`; CloudFront + OAC can front a function URL

![LambdaSecurity](images/LambdaSecurity.png)

## Lambda Logging

- Lambda uses CloudWatch Logs and X-Ray
- Logs from Lambda executions are stored in CloudWatch Logs
- Details about Lambda metrics are stored in CloudWatch Metrics
- Lambda can be integrated with X-Ray for distributed tracing
- Advanced logging controls: JSON structured logs, log level filtering, custom log group per function; CloudWatch Lambda Insights for system metrics
- Telemetry API / extensions for third-party observability (Datadog, ADOT)
- For Lambda to be able to log we need to give permissions via the execution role

## Lambda Invocations

- There are 3 ways Lambda functions can be invoked:
  - **Synchronous invocation**:
    - Command line or API directly invoking the function
    - The CLI or API will wait until the function returns
    - API Gateway will also invoke Lambdas synchronously, use case for many serverless applications
    - Any errors or retries have to be handled on the client side
      ![SynchronousLambda](images/SynchronousLambda.png)
  - **Asynchronous invocation**:
    - Used typically when AWS services invoke the function (example: S3 events)
    - The service will not wait for the response (fire and forget)
    - Lambda is responsible for any failure. Reprocessing will happen between 0 and 2 times (default 2 retries, configurable 0-2), events are kept in the internal queue for up to **6 hours** (`MaximumEventAgeInSeconds` configurable 60s - 6h)
    - Async payload limit: **1 MB** (raised from 256 KB in 2025)
    - The function should be idempotent in order to be rerun
    - Lambda can be configured to send events to a DLQ in case of the processing did not succeed after the number of retries (DLQ: SQS or SNS, only the event payload)
    - Destination: events processed by Lambdas can be delivered to destinations like SQS, SNS, other Lambda, EventBride (and S3 for failures). Success and failure events can be sent to different destinations
    - Destinations are preferred over DLQs: they include invocation record + error/response details and support on-success
      ![AsynchronousLambda](images/AsynchronousLambda.png)
  - **Event Source mapping**:
    - Typically used on streams or queues which don't generate events (Kinesis, DynamoDB streams, SQS, Amazon MSK / self-managed Kafka, Amazon MQ, DocumentDB change streams)
    - Event Source mappers polls these streams and retrieves batches. These batches can be broken in pieces and sent to multiple Lambda invocations for processing
    - By default a batch is all-or-nothing: if the function fails, the whole batch is retried. Today we can return **partial batch responses** (`ReportBatchItemFailures`) for SQS, Kinesis and DynamoDB Streams so only the failed records are retried
    - Streams (Kinesis/DynamoDB): records processed in order per shard; on error use `BisectBatchOnFunctionError`, `MaximumRetryAttempts`, `MaximumRecordAgeInSeconds` and an **on-failure destination** (SQS/SNS/S3), otherwise a poison record blocks the shard until it expires. `ParallelizationFactor` up to 10 concurrent batches per shard
    - SQS: batch size up to 10,000 (standard) / 10 (FIFO), batching window up to 5 min, **maximum concurrency** setting to protect downstream; set queue visibility timeout >= 6x function timeout; failures go to the **queue's** redrive policy DLQ (not Lambda's)
    - Event filtering on the ESM: discard records before invoking (pay less)
    - **Provisioned mode** for Kafka and SQS ESMs: pre-provisioned pollers for predictable high throughput / low latency
      ![EventLambda](images/EventLambda.png)
- In case of event processing in async invocation, in order to process the event we don't explicitly need rights to read from the sender
- In case of event source mapping the event source mapper is reading from the source. The event source mapping uses permissions from the Lambda execution role to access the source service
- Even if the function does not read data directly from the stream, the execution role needs read rights in order to handle the event batch
- Any batch that consistently fails to be processed, it can be sent to an SQS queue or SNS topic for further processing

## Lambda Versions

- We can define different versions for given functions
- A version of a function is the code + configuration of the function
- When we publish a version, it becomes immutable, it no longer can be changed. It event gets its own ARN (Amazon Resource Name)
- `$Latest` points to the latest version of Lambda version (it is not immutable)
- We can also define aliases (DEV, STAGE, PROD) which point to a version of the function. Aliases can be changed to point to other versions

## Lambda Start-up Times

- Lambda code runs inside of a runtime environment (execution context)
- At first invocation this execution context needs to be created and this will take time
- This process is known as cold start and it can take 100ms or more
- If the function is invoked again without too much of a gap, it might use the same execution context. This is called warm start
- One function invocation runs in an execution environment at a time. If multiple parallel instances are needed, the contexts will require cold starts
- **Provisioned concurrency**: we can provision one or more execution contexts in advance for Lambda invocations (configured on an alias/version, not `$LATEST`; can be auto scaled with Application Auto Scaling; costs money even when idle)
- **SnapStart**: snapshot of the initialized execution environment (Firecracker microVM) taken at version publish, restored on cold start => up to 10x faster starts. Supported for **Java 11+, Python 3.12+, .NET 8+**; no extra cost for Java (Python/.NET pay caching/restore). Not compatible with provisioned concurrency, EFS or ephemeral storage > 512 MB; beware uniqueness (random seeds, connections) after restore - use runtime hooks
- The improve performance we can use the `/tmp` folder to pre-download data to it. If another invocations uses the same execution context, it will be able to access the previously downloaded data
- We can create database connections outside of the Lambda handler. These will also be available for other invocations afterwards

## Lambda Function Handler

- Lambda function executions have life cycles
- The function code runs inside an execution environment
- Lifecycle phases:
  - `INIT`: creates or unfreezes the execution environment
    - It has the following sub-components:
      - `EXTENSION INIT`
      - `RUNTIME INIT`
      - `FUNCTION INIT`
    - Init phase runs only at cold-starts
  - `INVOKE`: runs the function handler (cold start)
  - `NEXT INVOKE`(s): warm start using the same environment
  - `SHUTDOWN`: execution environment is terminated after a period of inactivity
    - It has the following sub-components:
      - `RUNTIME SHUTDOWN`
      - `EXTENSION SHUTDOWN`
    - We can use Provisioned Concurrency to avoid having a cold-start in case the Lambda should be terminated

![Lambda Handler](images/LambdaHandler.png)

## Lambda Versions and Aliases

- Unpublished functions can be changed and deployed
- `$LATEST` version of the Lambda code can be edited and deployed
- We can take the current state of the function and publish it which will create an immutable version
- If the function is published, the code, dependencies, runtime settings and env. variables in the version created can not be edited
- Each version gets an uniq ARN (Qualified ARN)
- Unqualified ARN points at the function without a specific version (`$LATEST`)
- An alias is a pointer to a function version
- Example: PROD => function:1, BETA => function:2
- Each alias has an unique ARN
- Aliases can be updated, changing which version they reference
- Useful for PROD/DEV, BLUE/GREEN deployments, A/B testing
- We can also use alias routing: sending a certain percentage of request to v1 and other percentage to v2. Both versions need the same role, same DLQ (or no DLQ) will be used and both versions need to be published

![Lambda Versions](images/LambdaVersions.png)

![Lambda Aliases](images/LambdaAliases.png)

## Lambda Environment Variables

- Key and value pairs for associated with Lambda functions
- By default they are associated with `$LATEST` - can be edited
- If they are publishes, they can not be edited
- They can be accessed within the execution environment
- The environment variables can be encrypted with KMS
- They allow code execution to be adjusted based on variables

## Lambda Layers

- Used to split off libraries and dependencies from Lambda functions
- Reduces the size of the deployment package
- Layers can be reused by multiple Lambda functions
- Libraries in layers are extracted in the `/opt` folder
- Layers allow new runtimes which are not explicitly supported by AWS

![Lambda Layers](images/LambdaLayers.png)

## Lambda Container Images

- Until recently Lambda was considered to be a Function as a Service (FaaS) product, which means creation of a function, uploading code and executing it
- Many organizations use containers and CI/CD processes built for containers
- Lambda is now capable to use containers images
- It is an alternative way of packaging the function code and using it with the Lambda product
- Lambda Runtime API - has to be included with the container images, it is a package which allows interaction between a container and the Lambda
- AWS Lambda Runtime Interface Emulator (RIE): used for local Lambda testing

![Lambda Container](images/LambdaContainer.png)

## Lambda and ALB

- Lambda functions can be registered to ALB target groups
- Communication between the user and the ALB is via HTTP/HTTPS, there is no difference than connecting to a classic server (EC2) from the user's perspective
- When the ALB receives a request from the client it synchronously invokes the Lambda function
- The LB passes in a JSON structure to the Lambda function, inside the `Event` structure. This has to be interpreted by the Lambda. What actually happens is that the LB translates the HTTP(S) request to a Lambda compatible event, to which the Lambda answers with a JSON object that gets translated back to HTTP/HTTPS response

![Lambda ALB](images/LambdaALB.png)

- Multi-Value headers:
  - For an example lets us this URL for the Lambda: http://catagram.io?&search=roffle&search=winkie
  - Without multi-value headers the Lambda receives the following:
    ```
    "queryStringParameters": {
        "search": "winkie"
    }
    ```
  - If the multi-value headers are enabled, we get this delivered to Lambda:

    ```
    "multiValueQueryStringParameters": {
        "search": ["roffle", "winkie"]
    }
    ```

    ![Lambda Multi-value](images/LambdaMultiValue.png)

## Lambda Concurrency and Scaling

- Default **1,000 concurrent executions per region per account** (soft limit, can go to tens of thousands); new accounts may start lower
- Concurrency = requests per second x average duration
- Scaling rate: each function can add **1,000 execution environments every 10 seconds**
- **Reserved concurrency**: guarantees AND caps concurrency for a function (setting 0 = throttle/disable the function). Free
- **Provisioned concurrency**: pre-initialized environments, no cold starts, paid
- Throttling: sync => `429 TooManyRequestsException` to the caller; async => Lambda retries for up to 6h; ESM => pollers back off
- Exam: protect a downstream RDS from a burst => reserved concurrency (cap) + RDS Proxy; latency-sensitive API with Java => SnapStart or provisioned concurrency

## Lambda Limits (cheat sheet)

- Memory 128 MB - 10,240 MB (up to 6 vCPU), timeout **15 min**, `/tmp` 512 MB - 10,240 MB
- Payload: **6 MB** sync request/response, **200 MB** streamed response, **1 MB** async
- Package: 50 MB zipped upload (larger via S3), 250 MB unzipped incl. layers, **10 GB** container image
- 5 layers per function, 4 KB env vars total, 1,024 file descriptors/threads
- Code storage 300 GB per region (or self-managed S3 code storage)
- No persistent local storage => EFS mount (VPC required) for shared/large files

## Function URLs and Response Streaming

- **Function URL**: dedicated HTTPS endpoint `https://<id>.lambda-url.<region>.on.aws`, auth `AWS_IAM` or `NONE`, CORS; no throttling/WAF/caching (put CloudFront in front)
- **Response streaming**: send the response progressively (TTFB improvement, payloads up to 200 MB); via Function URLs (`RESPONSE_STREAM`), `InvokeWithResponseStream` API and API Gateway REST response streaming

## Lambda@Edge vs CloudFront Functions

- **CloudFront Functions**: JavaScript, sub-millisecond, viewer request/response only, no network/body access, tiny code (10 KB), millions of RPS, very cheap => header manipulation, URL rewrites/redirects, simple token validation, cache key normalization (+ **CloudFront KeyValueStore** for config data)
- **Lambda@Edge**: Node.js/Python, viewer (5s, 128 MB) and origin request/response (30s, up to 10 GB), network access, body access => auth with external calls, dynamic origin selection, image transformation. Authored in **us-east-1** and replicated; no VPC, no env vars, no layers, no container images
- Exam: "lowest latency/cost simple header rewrite" => CloudFront Functions; "call external service / modify origin" => Lambda@Edge

## Lambda Recursive Loop Detection

- Lambda detects recursive loops between Lambda and **SQS, SNS and S3** (e.g. function writes to the same bucket that triggers it) and stops them after ~16 invocations, sending events to DLQ/on-failure destination
- Enabled by default, can be turned off per function (`RecursiveLoop = Allow`) when recursion is intended

## Lambda Managed Instances

- Launched re:Invent 2025: Lambda functions running on **EC2 instances in our account/VPC** (choose instance families incl. Graviton4, latest gen), AWS manages provisioning, OS patching, scaling and routing
- Each execution environment handles **multiple concurrent requests** (no 1-request-per-environment model) => fewer cold starts, better utilization
- Pricing = EC2 instance pricing (Savings Plans / RIs apply) + management fee + request charge; no per-ms duration billing
- Async and most ESM invocations can run up to **90 minutes**
- Use case: steady, high-volume workloads where classic Lambda is expensive, or that need specific hardware, while keeping the Lambda programming model

## Lambda Durable Functions

- Launched re:Invent 2025: write **multi-step, long-running workflows in code** (Python, Node.js; more SDKs added later) with automatic **checkpointing** and replay
- `step` (checkpointed unit), `wait` (pause **up to 1 year** without paying compute), callbacks (wait for external event / human approval)
- Each invocation still max 15 min; the overall durable execution can last up to 1 year
- Lambda durable functions vs Step Functions: durable = workflow in application code (developer-centric); Step Functions = visual/ASL orchestration, 200+ service integrations, better for cross-service orchestration and operations visibility

## Other Lambda Form Factors (2026)

- **Lambda MicroVMs** (June 2026): isolated Firecracker VMs with their own HTTPS URL, suspend/resume, run up to **8 hours**, up to 16 vCPU/32 GB (ARM64) => sandboxes for user- or AI-generated code, per-tenant/session environments
- **Lambda Web Functions** (Oct 2026): run standard HTTP servers (Express, Fastify, Next.js) without event handler adapters, up to 128 concurrent requests per environment, **active CPU billing**, optional multi-region deployment behind one global URL
- Classic Lambda is now called **Event Functions** in this taxonomy
- These are very new; the SAP-C02 exam is unlikely to test them, but they matter for real architecture choices

## Multi-Tenancy

- **Tenant isolation mode** (Nov 2025): pass a tenant ID per invocation, Lambda guarantees execution environments are never shared between tenants (SaaS isolation without one function per tenant)

## Deployments

- Traffic shifting with **CodeDeploy** on aliases: `Canary10Percent5Minutes`, `Linear10PercentEvery1Minute`, `AllAtOnce`, with pre/post traffic hooks and CloudWatch alarm rollback (SAM `DeploymentPreference`)
- Lambda Powertools: idempotency, structured logging, metrics, tracing, batch processing utilities

## Exam Tips

- Job > 15 min => Step Functions (orchestrate) / ECS Fargate / AWS Batch (or Lambda Managed Instances up to 90 min async, durable functions for waiting)
- Many DB connections from Lambda => **RDS Proxy**
- Large shared files / ML models => EFS mount or container image (10 GB)
- Cold starts on Java => SnapStart; strict latency => provisioned concurrency
- Need static outbound IP => VPC Lambda in private subnet + NAT Gateway (Elastic IP)
- Process S3 events reliably at scale => S3 => SQS => Lambda (buffer, retries, DLQ) or S3 => EventBridge
- Lambda can't be reached by inbound connections; invoke via API, URL, ALB, API Gateway or events
