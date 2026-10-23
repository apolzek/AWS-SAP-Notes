# CloudWatch Events and EventBridge

- Deliver a near real-time stream of system events
- These events describe changes in AWS services, example: EC2 instance is started
- EventBridge is a newer system replacing CloudWatch Events. It can perform the same functionality, in addition it can handle events from third-parties and custom applications
- CloudWatch Events is now just the legacy name: same API, rules created in CloudWatch Events show up on the EventBridge default bus. Use EventBridge for everything new
- Both of the services operate using an event bus. Both have a default event bus
- In CloudWatch Events there is only the default event bus, which is explicit and it is not exposed to the UI
- In EventBridge we can have additional event buses
- In both systems we create rules matching incoming events, or we have scheduled based rules
- Events themselves are JSON objects, including for example which EC2 instance changed state, in what state changed into as well as other things such as date and time

![EventBridge](images/EventBridge.png)

## Event Buses and Rules

- Bus types: **default** (AWS service events), **custom** (our applications), **partner** (SaaS: Datadog, Zendesk, Shopify, etc.)
- Rule = event pattern (or schedule, legacy) + up to **5 targets** per rule
- Event patterns: match on any field, prefix/suffix, `anything-but`, numeric ranges, `exists`, IP CIDR, wildcards, `$or`
- **Input transformer**: reshape the event before sending to the target
- Max event size: **1 MB** (raised from 256 KB in 2025)
- Delivery is **at-least-once**; ordering not guaranteed => idempotent consumers
- Retry policy per target: up to **24h** and **185 attempts** (exponential backoff), then optional **SQS DLQ** per target
- Targets: Lambda, SQS, SNS, Step Functions, Kinesis, Firehose, ECS task, Batch, CodePipeline, SSM, API Gateway, **API destinations**, another event bus, etc. (20+)
- AWS API calls via **CloudTrail** events (`AWS API Call via CloudTrail`) => react to almost any API call (e.g. auto-remediate security group change)
- Resource-based policy on the bus => allow other accounts / whole AWS Organization (`aws:PrincipalOrgID`) to put events

## Cross-Account and Cross-Region

- Bus-to-bus: rule target = event bus in another account/region => central "event hub" / aggregation account pattern
- **Direct cross-account targets** (Jan 2025): SQS, Lambda, SNS, Kinesis, API Gateway in another account as direct targets (receiver only grants access via resource policy, no intermediate bus)
- Cross-region event routing supported to buses in other regions

## Archive and Replay

- **Archive**: store all/filtered events from a bus, retention indefinite or N days, encrypted
- **Replay**: re-send archived events from a time window to the bus => recover from bugs, test new consumers, DR

## Schema Registry

- Schemas discovered automatically (schema discovery) or defined (OpenAPI / JSONSchema)
- Code bindings for Java, Python, TypeScript, Go

## API Destinations

- HTTP endpoint (any SaaS/on-prem public API) as a target, with **connections** (Basic, API key, OAuth client credentials) stored in Secrets Manager
- Rate limit per destination (invocations/second) to protect the receiver
- Private API destinations via VPC Lattice resource configurations (reach private endpoints in VPC / on-prem)

## EventBridge Pipes

- **Point-to-point** integration: source => (filter) => (enrichment) => target, no glue code
- Sources: SQS, Kinesis, DynamoDB Streams, MSK / self-managed Kafka, Amazon MQ
- Enrichment: Lambda, Step Functions Express, API Gateway, API destination
- Targets: same breadth as rules (Step Functions, event bus, SQS, Kinesis, ECS, etc.)
- Only filtered events are billed => cheaper than a Lambda poller just to forward/filter
- Pipes = 1 source => 1 target; Event bus = many sources => many targets (routing)

## EventBridge Scheduler

- Serverless scheduler, replacement for scheduled rules: **one-time** (`at()`), `rate()` and `cron()` schedules
- Time zones and DST aware, **flexible time windows** (spread load), start/end dates
- Scale: millions of schedules per account (scheduled rules are limited to 300 rules per bus)
- Universal targets: 270+ services / 6,000+ API actions via SDK integration, plus templated targets (Lambda, SQS, SNS, Step Functions, ECS `RunTask`, etc.)
- Retry policy + DLQ, schedule groups, IAM execution role per schedule
- Exam: "run a task at a specific time per user/order (e.g. send reminder in 48h)" => Scheduler one-time schedule (not DynamoDB TTL / cron Lambda)

## Global Endpoints

- Multi-region resiliency for event ingestion: publishers send to a **global endpoint**, Route 53 health check decides primary/secondary region
- Optional event **replication** to the secondary region; failover automatically when the health check fails (based on a CloudWatch alarm)
- Recovery target RPO/RTO in the order of minutes; consumers must exist in both regions

## EventBridge vs SNS vs SQS (Exam)

- **EventBridge**: content-based routing with rich patterns, many AWS/SaaS sources, schema registry, archive/replay, cross-account buses; higher latency (~ hundreds of ms) and lower default throughput (PutEvents soft limit, e.g. 10,000/s in big regions)
- **SNS**: high-throughput, low-latency fan-out, mobile push/SMS/email, FIFO topics for ordering
- **SQS**: buffering / decoupling, consumer pulls at its own pace
- Common: EventBridge rule => SQS queue => Lambda (buffering + retries + DLQ)
