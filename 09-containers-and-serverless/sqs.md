# SQS - Simple Queue Service

- SQS provides managed message queues
- It is a public AWS service
- It is fully managed with highly available and highly performant queues by design
- Queues come in two types: Standard (ordering is best effort) and FIFO (guarantee an order)
- Messages added to the queue are <= **1 MiB** (raised from 256 KB in Aug 2025, Standard and FIFO; if more => store on S3 and add link in SQS message)
- `VisibilityTimeout`: when a client receives a message, it is not deleted directly => the message is hidden for a period of time. The `VisibilityTimeout` is the period which is given to the client in order to process the message
- If the client processes the message, it has to delete the message from the queue. If the message is not deleted, after the `VisibilityTimeout` the message will reappear in the queue (retry)
- `VisibilityTimeout`: default 30s, 0s - **12 hours**; can be extended per message with `ChangeMessageVisibility` (heartbeat for long jobs)
- Message retention: 1 min - **14 days**, default **4 days**
- In-flight messages (received, not deleted): **120,000** per queue (Standard and FIFO)
- Unlimited number of messages stored in a queue (backlog)
- **Dead-Letter queues**: all unprocessed messages can be sent here after a given number of retries. Allows different types of processing on the messages
- Queues can be used for scaling by ASGs, Lambdas can be invoked based on messages in the queue
  - ASG/ECS scaling: target tracking on a custom metric **backlog per instance** (`ApproximateNumberOfMessagesVisible` / running instances), not on raw queue length

![SQS](images/SQS.png)

- Fan-out Architecture using SNS

![Fanout](images/Fanout.png)

## SQS Standard vs FIFO Queues

- Standard:
  - Messages are delivered **at-least-once** !!! To favour HA and low latency => SQS standard is eventually consistent and it can happen that the same message is sent >1x
  - The order of the delivery is not guaranteed (best-effort ordering)
  - They don't have performance limitations
- FIFO:
  - Messages are delivered **exactly-once** (exactly-once *processing*: duplicates are removed within a **5 minute deduplication interval** using `MessageDeduplicationId` or content-based deduplication (SHA-256 of the body))
  - Ordering is per **message group** (`MessageGroupId`): messages of one group are processed one at a time in order, different groups in parallel => more groups = more parallelism
  - The order of the delivery is guaranteed to be the same as the messages were sent
  - Performance is limited: 3000 messages per second with batching or up to 300 messages per second without batching (since up to 10x messages can be returned)
  - There is also available a high throughput mode for FIFO queues: deduplication scope = message group, throughput limit = per message group ID => up to **70,000 TPS without batching (700,000 msg/s with batching)** in us-east-1/us-west-2/eu-west-1 (lower in other regions, e.g. 2,400 TPS default for smaller regions)
  - FIFO queues have to have `.fifo` suffix in order to be valid FIFO queues

![SQSQueues](images/SQSQueues.png)

## SQS Billing

- With SQS => billed per request
- 1 request = getting 1-10 messages back; every **64 KB chunk** of a payload is billed as 1 request (a 1 MiB message = 16 requests)
- FIFO requests cost more than Standard requests
- Use batching (`SendMessageBatch`, `DeleteMessageBatch`, `ReceiveMessage` with `MaxNumberOfMessages=10`) and long polling to cut cost
- SQS is less cost effective if we poll more frequently

## SQS Polling

- Polling SQS queues:
  - **Short polling** (immediate): can return 0 or more messages. It will return immediately if there are no messages on the queue => more expensive
  - **Long polling**: we can specify `waitTimeSecond` value which can be up to 20 seconds to wait for messages when polling. It uses fewer requests than short polling
  - Set per request (`WaitTimeSeconds`) or per queue (`ReceiveMessageWaitTimeSeconds`); exam: "reduce empty receives / cost" => long polling

## SQS Encryption and Security

- SQS supports encryption at rest (KMS) and encryption in transit
  - **SSE-SQS** (SQS-owned keys) is enabled by default on new queues; **SSE-KMS** for audit/key control (needs KMS permissions for producers, consumers and services like SNS/S3/EventBridge that send to the queue)
  - Enforce TLS with `aws:SecureTransport` in the queue policy
  - Private access from VPC via SQS interface VPC endpoint (no NAT needed)
- Access to a queue is based on identity policies and queue policies
- Queue policies can allow cross-account access

## SQS Extended Client Library

- SQS has a message size limit of 1 MiB (was 256 KB)
- Extended Client Library can be used when we want to send messages > 1 MiB (or to keep queue costs down for large payloads)
- It can process large payloads and have the bulk of the payloads stored in S3
- When we send a message using `SendMessage` API, the library uploads the content to S3 and stores a link to this content in the message
- When receiving a message, the library loads the payload from S3 and replaces the link with the payload in the message
- When deleting a message from a queue, the large S3 payload will also be deleted
- The Extended Client Library can handle messages <= 2GB
- It has an implementation in Java. Equivalent libraries are available for other languages

## SQS Delay Queues

- Delay queues allow us to postpone the delivery of messages in SQS queues
- For a delay queue we configure a `DelaySeconds` value. Messages added to the queue will be invisible for the amount of `DelaySeconds` (= not returned when ReceiveMessages is called)
- The default value of `DelaySeconds` is 0, the max value is 15 min. In order for a queue to be delay queue, `DelaySeconds` should be > 0
- Message timers allows a per-message basis invisibility to be set (`DelaySeconds` on `SendMessage`), overriding the queue setting. Min/max values are the same
- Per message setting is **NOT** supported for FIFO queues
- `VisbilityTimeout` vs `DelaySeconds`:

![SQSDelay](images/SQSDelay.png)

## SQS Dead Letter Queues (DLQ)

- Dead Letter Queues are designed to help handle reoccurring failures while processing messages from a queue
- Every time a message is received, the `ReceiveCount` is incremented. In order to not process the same message over and over again, we define a `maxReceiveCount` and a DLQ for a queue. Once `ReceiveCount` > `maxReceiveCount` => the message is pushed into the DLQ
- Having a DLQ, we can define alarms to get notified in case of a failure
- Messages in DLQ can be further analyzed
- Every queue (including DLQ) have a retention period for messages. Every message has an enqueue timestamp which represents when the message was added to the queue. In case a message is moved into a DLQ from a **Standard** queue, this timestamp is not altered (for **FIFO** queues the enqueue timestamp is reset when moved to the DLQ)
- Generally the retention period for a DLQ should be longer compared to normal queues
- A single DLQ can be used for multiple sources
- The type of the DLQ (standard/FIFO) should match the source queue
- The DLQ must be in the same account and region as the source queue
- **Redrive allow policy** on the DLQ: control which source queues can use it
- **DLQ redrive** (`StartMessageMoveTask`): move messages from the DLQ back to the source queue (or another queue) after fixing the bug, with optional velocity limit - supported for Standard and FIFO
- Don't use a DLQ for FIFO when strict ordering must never be broken (a message moved away changes the order)

![DLQ](images/DLQ.png)

## SQS Fair Queues

- Feature of **Standard** queues (July 2025) for multi-tenant workloads: if producers set `MessageGroupId` (= tenant ID), SQS reduces the "noisy neighbour" effect, keeping dwell time low for other tenants when one tenant floods the queue
- No consumer changes needed, keeps Standard queue throughput (unlike FIFO, which serializes per group)

## SQS and Lambda

- Event source mapping polls the queue (long polling), batch size up to 10,000 (Standard) / 10 (FIFO), batch window up to 5 min
- Scales up to 1,250 concurrent invocations for Standard queues (provisioned mode for more/faster), FIFO scales by number of active message groups
- Use **maximum concurrency** on the ESM (not reserved concurrency) to limit consumers without throttling-induced retries
- Queue visibility timeout >= **6x** function timeout; failed messages go to the queue's DLQ after `maxReceiveCount`
- Return partial batch failures (`ReportBatchItemFailures`) to avoid reprocessing successful messages

## Other Features

- **Temporary queues** (Temporary Queue Client): virtual queues for request-response patterns without creating real queues
- Message attributes (up to 10), message system attributes (X-Ray trace header)
- **Amazon SQS vs Kinesis Data Streams**: SQS = per-message processing, no replay, auto scaling, message deleted after consumption; Kinesis = ordered per shard, multiple consumers read the same data, **replay** within retention (up to 365 days), real-time analytics
- Cross-account: queue policy granting `sqs:SendMessage` / `sqs:ReceiveMessage` to another account or an SNS topic/S3 bucket (`aws:SourceArn`)

## Exam Tips

- "Decouple and absorb spikes" => SQS between tiers; "order + no duplicates" => FIFO (watch the throughput numbers)
- "Process each message by multiple systems" => SNS/EventBridge fan-out to multiple queues (one queue = one logical consumer)
- "Duplicate processing in Standard queue" => make consumers idempotent (e.g. DynamoDB conditional writes) or use FIFO
- "Message processing takes longer than visibility timeout" => increase timeout or heartbeat with `ChangeMessageVisibility`
- "Prioritize messages" => separate queues per priority (SQS has no native priority)
