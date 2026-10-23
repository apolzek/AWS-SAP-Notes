# Amazon MQ

- SNS and SQS are AWS services using AWS APIs
- SNS provides topics which are one to many communication channels, SQS provides queues which are one to one communication channels
- Both SNS and SQS are public services, HA and AWS integrated
- Larger organization might already use on-premise messaging systems, which are not entirely compatible with SNS and SQS
- Amazon MQ:
  - Open-source message broker: managed implementation of **Apache ActiveMQ** (Classic) and **RabbitMQ**
  - ActiveMQ supports the JMS API and protocols such as AMQP 1.0, MQTT, OpenWire, STOMP and WebSocket
  - RabbitMQ supports AMQP 0-9-1 (and AMQP 1.0 natively on RabbitMQ 4.x)
  - It provides **both queues and topics**
  - It uses message broker services which can be single instance (test, dev) and HA pair (Active/StandBy for production)
    - ActiveMQ **active/standby**: 2 brokers in 2 AZs sharing durable storage on **EFS**, automatic failover (clients use the failover transport URL). Single-instance brokers can use EBS (higher throughput, single AZ)
    - ActiveMQ **network of brokers**: multiple brokers connected for scale and availability
    - RabbitMQ **cluster deployment**: 3 nodes across 3 AZs behind an NLB, quorum queues replicated across nodes
  - Unlike SQS and SNS, Amazon MQ is not a public service, it runs in a VPC (brokers can optionally be made **publicly accessible**, still not an AWS public "API" service)
  - It does not have native integration with other AWS services in the same way as SNS/SQS
  - One-to-One or One-to-Many
  - Not a public service => deployed in a VPC
- Versions and features:
  - RabbitMQ 4.2 (on Graviton `mq.m7g` instances): **quorum queues** are the default (replicated, durable), Khepri metadata store, native AMQP 1.0
  - ActiveMQ 5.18+ (Classic)
  - **Cross-Region Data Replication (CRDR)** for ActiveMQ active/standby brokers: async replication to a replica broker in another region, API-driven failover/switchover => DR
  - Authentication: ActiveMQ native users or **LDAP**; RabbitMQ internal users (plus OAuth 2.0/LDAP on newer versions)
  - Encryption at rest with KMS, in transit TLS; logs/metrics to CloudWatch
  - Pricing: broker instance hours + storage (no per-message charge, unlike SQS/SNS)
- Integrations:
  - **Lambda event source mapping** for ActiveMQ and RabbitMQ (Lambda polls the broker)
  - **EventBridge Pipes** source (Amazon MQ => filter/enrich => target)
  - No native SNS/S3/EventBridge style integrations from the broker itself
- Amazon MQ considerations:

  - By default we should chose SNS/SQS for newer implementations
  - We should use Amazon MQ if we migrate from an existing system with little to no application change
  - We should use Amazon MQ if we need APIs such as JMS or protocols such as AMQP, MQTT, OpenWire, STOMP
  - Amazon MQ requires the appropriate private networking

- Example of co-existing on-prem and AWS MQ implementation (during migration) => private connectivity to on-prem is required:

![Amazon MQ](images/AmazonMQ.png)

## Messaging Service Selection (Exam)

- New cloud-native app, decoupling => **SQS** / **SNS** / **EventBridge**
- Lift-and-shift of on-prem ActiveMQ/RabbitMQ/IBM MQ/TIBCO apps using JMS/AMQP/MQTT/STOMP with no code change => **Amazon MQ** (IBM MQ/TIBCO apps usually move via JMS => ActiveMQ)
- Kafka clients / event streaming with replay and long retention => **Amazon MSK**
- AWS-managed streaming without Kafka API => **Kinesis Data Streams**
- IoT devices with MQTT at massive scale => **AWS IoT Core** (not Amazon MQ)
