# SNS - Simple Notification Service

- It is a highly available, durable, secure, pub-sub messaging service
- It is a public AWS service: requires network connectivity with a public endpoint
- It coordinates the sending and delivery of messages
- Messages are <= 256 KB payloads by default; since Sept 2026 a topic can accept up to **1 MiB** by setting the `MaximumMessageSize` topic attribute (large-message topics are limited to SQS, Lambda and Firehose subscriptions). Bigger payloads (up to 2 GB) => **SNS Extended Client Library** (payload in S3)
- Billing: each 64 KB chunk of a publish counts as one request
- SNS Topics are the base entity of SNS. On this topics are permissions controlled and configurations defined
- A publisher sends messages to a topic
- Topics can have subscribers which will receive messages from a topic
- Supported subscribers are:
  - HTTP(s)
  - Email (JSON)
  - SQS queues
  - Mobile Push Notifications
  - SMS messages
  - Lambda Functions
  - Amazon Data Firehose (archive to S3/Redshift/OpenSearch)
  - Email-JSON
- Subscribers can be in other accounts and other regions (SQS/Lambda cross-region delivery supported)
- SNS is used across AWS for notifications, example CloudWatch uses it extensively
- It is possible to apply filters to a subscriber
  - **Subscription filter policies** on message attributes (default) or on the **message body** (`FilterPolicyScope = MessageBody`, payload-based filtering)
  - Operators: exact, prefix/suffix, `anything-but`, numeric ranges, `exists`, IP address, `$or`
  - Without filters every subscriber receives every message; filtering avoids "filter in the consumer" costs
- Fan-out architecture: single topic with multiple SQS queue subscribers
- SNS offers delivery status for supported subscribers: HTTP, Lambda and SQS
- SNS supports delivery retries
  - AWS-managed endpoints (SQS, Lambda, Firehose): retried aggressively for days (up to ~100,000 attempts over 23 days)
  - HTTP(S): customizable **delivery policy** (retries, backoff, throttling), up to 1h
  - Per-subscription **DLQ** (redrive policy => SQS queue) for messages that could not be delivered after retries (e.g. endpoint deleted, KMS issue)
- **Raw message delivery** for SQS/HTTP/Firehose: deliver the payload without the SNS JSON envelope
- SNS is a HA and scalable service within a region
- SNS supports Server Side Encryption (SSE) with KMS. If an AWS service publishes (CloudWatch alarms, S3 events, EventBridge) to an encrypted topic, the **KMS key policy must allow that service** - with a customer managed key (the AWS managed key `aws/sns` can't be used for that) - common exam trap
- Encrypted topic => subscribed encrypted SQS queues also need KMS permissions for `sns.amazonaws.com`
- Private access from a VPC via an SNS interface VPC endpoint
- We can use cross-account access to a topic via topic policies

![SNS](images/SNS.png)

## SNS FIFO Topics

- Strict ordering per **message group** (`MessageGroupId`) + deduplication (`MessageDeduplicationId` or content-based, 5 min window)
- Name must end in `.fifo`
- Subscribers: SQS FIFO **and SQS standard** queues (standard queues lose ordering/dedup guarantees), up to 100 subscriptions per FIFO topic
- Throughput: 300 msg/s per message group; per topic 3,000 msg/s (or 20 MB/s) with `FifoThroughputScope = Topic`, higher when set to `MessageGroup` (bounded by the account publish quota, e.g. 30,000 msg/s in us-east-1)
- **Message archiving and replay** (FIFO only): keep messages on the topic (up to 365 days) and replay them to a subscription (e.g. recover a failed consumer, seed a new one) - no need for a separate archive queue
- Pattern: SNS FIFO => SQS FIFO queues = ordered fan-out (e.g. price updates to several systems)

## Message Data Protection

- **Data protection policies** on topics: detect PII/PHI (managed data identifiers such as credit card numbers, addresses, names) or custom identifiers in messages in transit
- Actions: audit (report findings to CloudWatch/S3/Firehose), **de-identify** (mask/redact), or **deny** publish/delivery
- Compliance use case (HIPAA, PCI, GDPR) without changing publishers

## Limits

- Standard topics: 100,000 per account; 12,500,000 subscriptions per topic
- FIFO topics: 1,000 per account; 100 subscriptions per topic
- Publish throughput: up to 30,000 msg/s in us-east-1 (soft, varies per region); `PublishBatch` up to 10 messages
- Email delivery: 10 msg/s per subscription (hard)
- SMS: spend threshold 1 USD by default; production SMS uses **AWS End User Messaging SMS** (formerly Pinpoint SMS) for origination identities

## SNS vs EventBridge vs SQS (Exam)

- **SNS**: push-based fan-out, very high throughput, low latency, mobile push/SMS/email, filter policies on attributes/body
- **EventBridge**: more targets and sources (AWS services, SaaS), richer content-based routing, schema registry, archive/replay for standard events, cross-account buses; lower throughput
- **SQS**: durable buffer, consumers poll
- **Fan-out**: SNS => multiple SQS queues (each consumer gets its own copy, independent retries and scaling). SQS queue policy must allow the topic to `sqs:SendMessage`
