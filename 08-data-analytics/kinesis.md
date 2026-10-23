# Kinesis

- Is a scalable streaming service, designed to ingest lots of data
- Producers send data into a Kinesis stream (stream = basic entity of Kinesis)
- Streams can scale from low to near infinite data rates
- It is a public service and it is HA in a region by design (data replicated across 3 AZs)
- Persistence: streams store by default a 24H moving window of data
- Kinesis include storage to be able to ingest and retain it for 24H by default (can be increased to 365 days **at additional cost**)
- Multiple producers/consumers can publish/consume the data (and anytime from the moving window)
- Family today: **Kinesis Data Streams**, **Amazon Data Firehose** (formerly Kinesis Data Firehose), **Amazon Managed Service for Apache Flink** (formerly Kinesis Data Analytics), **Kinesis Video Streams**

## Kinesis Data Streams

- Kineses Data Streams are using shards architecture for scaling, initially there is one shard, additional shards can be added over time to increase performance
- Streams are realtime (~200ms)
  - ~70 ms with enhanced fan-out
- Each shard provides its own capacity, each shard has 1MB/s ingestion capacity, 2MB/s consumption capacity
  - Write: **1 MB/s or 1,000 records/s** per shard
  - Read (shared/standard): **2 MB/s per shard shared by all consumers**, max 5 `GetRecords` calls/s per shard
  - Read (**enhanced fan-out**, `SubscribeToShard` push over HTTP/2): **2 MB/s per consumer per shard**, up to **20 consumers** per stream (**50** with On-demand Advantage)
- Shards directly affect the price of the Kinesis stream, we have to pay for each shard
- Pricing is also affected by the length of the storage window. By default, it is 24H => can be increased up to 365 days
- Data is stored in Kinesis Data Records (1MB), these records are distributed across shards
  - Max record size is **1 MiB by default, configurable up to 10 MiB** per stream (Oct 2025, `UpdateMaxRecordSize`). Per-shard throughput is unchanged => large records only for occasional bursts
  - `PutRecords` batch: up to 500 records / 10 MiB per request
- Record = partition key + sequence number + data blob. **Partition key** (hashed, MD5) decides the shard => **ordering is guaranteed per shard (per partition key)**
- Capacity: on-demand VS provisioned (defined number of shards)
  - **Provisioned**: we pick the shard count, scale by **resharding** (shard split = more capacity, shard merge = less) via `UpdateShardCount`. Pay per shard-hour
  - **On-demand Standard**: no capacity planning, default write capacity 4 MB/s, automatically scales up to 2x the previous peak (default max 200 MB/s write per stream, can be raised). Pay per GB + per stream-hour
  - **On-demand Advantage** (Nov 2025, account-level setting per region): no per-stream fee, ~60% lower per-GB pricing, **warm throughput** up to 10 GB/s, 50 EFO consumers; minimum billing of 25 MB/s ingest + retrieval => for high-volume / many-stream / high fan-out accounts
  - Can switch between on-demand and provisioned twice per 24h
- Hot shards / `ProvisionedThroughputExceededException` => better partition key (high cardinality), more shards, retries with backoff, KPL aggregation
- Producers: SDK (`PutRecord`/`PutRecords`), **KPL** (Kinesis Producer Library: batching, aggregation, retries), Kinesis Agent, CloudWatch Logs subscriptions, IoT Core, DynamoDB (Kinesis Data Streams for DynamoDB), DMS
- Consumers: **KCL** (Kinesis Client Library: uses a DynamoDB table for checkpoints/leases, 1 shard per worker at a time), Lambda (event source mapping, batch, parallelization factor up to 10 per shard), Firehose, Managed Service for Apache Flink, Glue streaming, EMR/Spark
- Security: IAM, KMS encryption at rest, TLS, **VPC interface endpoint** (PrivateLink), resource-based policies for **cross-account** access (2023)
- Data is not deleted when consumed (unlike SQS) => replay possible within the retention window

![Kinesis Architecture](images/KinesisArchitecture.png)

## SQS vs Kinesis Data Streams

- Is it about ingestion of data (Kinesis) or about decoupling, worker pools (SQS)
- SQS usually has 1 production group, 1 consumption group
- SQS is designed for decoupling and asynchronous communication
- SQS does not have the concept of persistence, no time window for persistence
  - Messages are deleted once processed (max retention 14 days)
- Kinesis is designed for huge scale ingestion, having multiple consumers with different rates of consumption
- Kinesis is recommended for ingestion, analytics, monitoring, click streams
- Ordering: Kinesis per shard; SQS only with FIFO queues (per message group)
- Scaling: SQS scales transparently; Kinesis by shards (provisioned) or on-demand

## Kinesis Data Streams vs Amazon MSK

- **MSK**: managed Apache Kafka (open source API, existing Kafka apps/ecosystem, Kafka Connect), message size configurable, retention unlimited (tiered storage)
- **Kinesis**: AWS-native, simpler, deeper AWS integrations, pay per shard/GB
- Exam: "migrate existing Kafka workloads with minimal code changes" => MSK

## Kinesis Data Firehose

> **Status (2026):** renamed to **Amazon Data Firehose** (Feb 2024). Same service and APIs (`firehose`)

- Used to provide data ingestion for other AWS services such as S3
- Fully managed service used to load data for data lakes, data stores and analytics services
- Data Firehose scales automatically, it is serverless and resilient
- **Near real time** product (~60 seconds for delivery => **NOT** real time)
  - Buffer interval can now be set from **0 to 900 seconds** (zero buffering for low latency to some destinations), so "near real-time" is still the exam answer vs. KDS real-time
- Supports transformation of data on the fly using Lambda. This transformation can add latency
- Firehose is a pay as you go service, we pay per volume of data that goes through Firehose
- Firehose supported destinations:
  - HTTP endpoints (third party providers: Datadog, New Relic, MongoDB, Dynatrace, etc.)
  - Splunk (has direct support for it)
  - RedShift
  - OpenSearch (ElasticSearch) and OpenSearch Serverless
  - S3
  - **Apache Iceberg tables** in S3 / **S3 Tables** (with routing, updates/deletes)
  - **Snowflake**
- Sources: Direct PUT (SDK, Kinesis Agent), Kinesis Data Streams, **Amazon MSK**, CloudWatch Logs, EventBridge, IoT Core, AWS WAF logs, VPC Flow Logs, database CDC
- Firehose can accept data directly from producers (if no need for Data Streams features => bypass) or from Kinesis Data Streams
- Firehose receives the data in real-time, but the ingestion is buffered
- Firehose's buffer by default waits for 1MB of data in 60 seconds before delivering to consumer. For higher load, it will deliver every time there is an 1MB chunk of data
  - Correction: buffer size is configurable **1-128 MiB** and interval **0-900 s**; delivery happens when **either** threshold is reached first. S3 default is **5 MiB / 300 s**
- Data is sent directly form Firehose to destination, exception being Redshift, where data is stored in an intermediary S3 bucket
  - Firehose writes to S3, then issues a Redshift `COPY` command
- Features:
  - **Format conversion** JSON => **Parquet/ORC** (schema from Glue Data Catalog) => cheaper Athena queries
  - **Dynamic partitioning** of S3 output by keys in the records (e.g. `customer_id/date`)
  - Compression (GZIP, Snappy, ZIP), KMS encryption
  - Failed records (or all source records) backed up to S3
  - No data storage/replay (unlike KDS), no consumers to manage
- Firehose use cases:
  - Persistence for data coming into Kinesis Data Streams
  - Storing data in a different format (ETL)
  - Log delivery to S3/OpenSearch/Splunk, streaming into a data lake (Iceberg)

![Kinesis Firehose](images/KinesisFirehose.png)

## Kinesis Data Analytics

Deprecated for **Amazon Managed Service for Apache Flink**

> **Status (2026):** Kinesis Data Analytics for **Java/Flink** apps was renamed **Amazon Managed Service for Apache Flink** (Aug 2023). Kinesis Data Analytics **for SQL applications is discontinued**: no new apps since Oct 15, 2025, applications deleted from **January 27, 2026**. Migrate to Managed Service for Apache Flink (Studio notebooks support SQL)

- It is a real-time data processing product using SQL
- The product ingests data from Kinesis Data Streams or Firehose
- After the data is processed, it can be sent directly to destinations such as:
  - Firehose (data becoming near-real time)
  - Kinesis Data Streams
  - AWS Lambda
- Kinesis Data Analytics architecture:

![Kinesis Data Analytics](images/KinesisDataAnalytics.png)

- Kinesis Data Analytics use cases:
  - Anything using stream data which needs real-time SQL processing
  - Time-series analytics: election data, e-sports
  - Real-time dashboards: leader boards for games
  - Real-time metrics

![Kinesis Deprecation](images/KinesisDeprecation.png)

## Amazon Managed Service for Apache Flink

- Fully managed **Apache Flink** (Java, Scala, Python, SQL via Table API / **Studio notebooks**)
- Sources: Kinesis Data Streams, **MSK**, S3, others via Flink connectors. Sinks: KDS, Firehose, MSK, S3, OpenSearch, DynamoDB, etc.
- Stateful processing: windowing, joins, aggregations, exactly-once semantics, **checkpoints and snapshots** for recovery
- Scales automatically, billed per **KPU** (1 vCPU + 4 GB memory) per hour
- Runs inside a VPC optionally (access private resources)
- Use cases: real-time anomaly detection, streaming ETL, real-time dashboards, complex event processing
- Exam: "real-time analytics on streaming data with SQL/Java, serverless" => Managed Service for Apache Flink. Simple per-record transformation => Lambda

## Kinesis Video Streams

- Ingest, store (configurable retention) and process video/audio/time-encoded data from devices (cameras, drones)
- Consumers: Rekognition Video, SageMaker, custom apps; playback via HLS/DASH; WebRTC for real-time two-way media
- One producer per stream (e.g. one camera = one stream)

## Choosing the Streaming Service (exam)

- Real-time (~70-200 ms), multiple consumers, replay => Kinesis Data Streams
- Load streaming data into S3/Redshift/OpenSearch/Splunk with no code, near real-time => Data Firehose
- Real-time SQL/stateful analytics => Managed Service for Apache Flink
- Kafka compatibility => MSK
- Decoupling/work queues => SQS
