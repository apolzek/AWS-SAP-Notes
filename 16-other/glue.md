# AWS Glue

- Is a serverless ETL (Extract, Transform, Load) product
- Similar to AWS Datapipeline (which can do ETL) but it uses servers (EMR clusters)
    > **Status (2026):** AWS Data Pipeline is no longer available to new customers (since July 2024) - migrate to Glue, Step Functions or Amazon MWAA
- Glue is used to move data and transform data between source and destination
- Glue also crawls data sources and generates the AWS Glue Data catalog
- Supports a range of data collection as data sources: S3, RDS, JDBC compatible data sources and DynamoDB
- Supports data streams as data sources such as Kinesis Data Streams and Apache Kafka
- Data targets supported: S3, RDS, JDBC data sources (also Redshift, DynamoDB, OpenSearch, DocumentDB, Kafka, and 3rd-party SaaS via Glue connectors - Salesforce, SAP, Snowflake, etc.)
- Glue is a family of services: Data Catalog, crawlers, ETL jobs (Spark, Spark Streaming, Python shell, Ray), Glue Studio (visual), DataBrew (no-code prep), Data Quality, Schema Registry, zero-ETL integrations

## Data Catalog

- Collection of persistent metadata about data sources in a region
- Provides one uniq data catalog in every region per account
- It helps avoid data silos: makes data not visible in an organization be visible a able to be browsed
- Data Catalog can be used by Amazon Athena, Redshift Spectrum, EMR and AWS Lake Formation
- Data is discovered by configuring crawlers and givin it credentials to access data sources
- Structure: databases -> tables (schema, partitions, location, format) -> connections (JDBC/network config, credentials in Secrets Manager)
- Hive metastore compatible - EMR, Athena, Redshift Spectrum share the same metadata; supports Apache Iceberg, Hudi and Delta Lake tables (incl. Iceberg statistics/compaction optimizers)
- Catalog federation: S3 Tables (`s3tablescatalog`), Redshift managed catalogs, remote Iceberg REST catalogs - basis of the **SageMaker Lakehouse** / unified Studio catalog
- Fine-grained access (database/table/column/row/cell) via **AWS Lake Formation** permissions on top of the Data Catalog; cross-account sharing with Lake Formation + RAM <span style="color: red;">EXAM</span>
- Encryption of catalog metadata and connection passwords with KMS; resource policies for cross-account catalog access

## Crawlers

- Connect to a data store (S3, JDBC, DynamoDB, DocumentDB/MongoDB, Iceberg/Delta/Hudi), infer schema with classifiers (built-in or custom - grok, XML, JSON, CSV) and create/update tables + partitions
- Run on demand or on a schedule (cron); **incremental crawls** (new folders only) or S3 event-based crawls (SQS) to avoid rescanning large buckets
- Alternative to crawlers for partitions: Athena partition projection or `MSCK REPAIR TABLE`/`ALTER TABLE ADD PARTITION`

## Glue Job

- Extract, Transform an Load jobs
- Jobs can do transformation by using scripts created by us
- Jobs are serverless, AWS maintains a pool of resources
- We are only billed by resources we consume (per **DPU-hour**, billed per second, 1 min minimum for Glue 2.0+)
- Job types:
    - **Spark** (batch, PySpark/Scala) and **Spark Streaming** (Kinesis, Kafka/MSK sources, micro-batches)
    - **Python shell** (small jobs, 1/16 or 1 DPU)
    - **Ray** jobs - **Status (2026):** no longer available to new customers since Apr 30, 2026
- Versions: Glue 4.0 (Spark 3.3), **Glue 5.0** (Dec 2024, Spark 3.5, Python 3.11, Java 17), **Glue 5.1** (Nov 2025, Spark 3.5.6, Iceberg 1.10, Lake Formation write support)
- Worker types: G.025X (streaming, small), G.1X, G.2X (general), G.4X, G.8X (memory/compute heavy), R.*X memory-optimized; auto scaling of workers
- **Flex execution** class: spare capacity, up to ~35% cheaper, for non-urgent jobs (not time-sensitive) <span style="color: red;">EXAM</span>
- **Job bookmarks**: track already-processed data to process only new data on each run (incremental ETL) <span style="color: red;">EXAM</span>
- DynamicFrames (schema flexibility, `ResolveChoice`, `Relationalize` for nested JSON) vs Spark DataFrames
- **Glue Studio**: visual ETL editor generating code; **interactive sessions** / notebooks for development
- Orchestration: **triggers** (scheduled, on-demand, conditional on job/crawler state, EventBridge event) and **workflows** (DAG of crawlers + jobs); or Step Functions / MWAA
- Monitoring: CloudWatch metrics/logs, Spark UI, job run insights; generative AI troubleshooting and Spark upgrade assistant

## Other Glue Features

- **AWS Glue DataBrew**: no-code visual data preparation with 250+ transformations, data profiling, recipes run as jobs - for analysts/data scientists <span style="color: red;">EXAM</span>
- **Glue Data Quality**: rules in DQDL (Data Quality Definition Language) on catalog tables or inside ETL jobs, rule recommendations, anomaly detection
- **Sensitive data detection** (PII) in ETL jobs with redaction/masking
- **Glue Schema Registry**: schemas (Avro, JSON Schema, Protobuf) with compatibility rules for streaming producers/consumers (Kinesis, MSK, Kafka, Lambda) - free
- **Zero-ETL integrations**: replicate data without pipelines from SaaS apps (Salesforce, SAP OData, ServiceNow, Zendesk...) and DynamoDB into Redshift or SageMaker Lakehouse (S3 Iceberg tables). Related: Aurora/RDS/DynamoDB zero-ETL to Redshift

## When to use what

- Serverless Spark ETL, catalog, crawlers -> **Glue**
- Full control of big data frameworks (Spark, Hive, Presto, HBase, Flink), long-running clusters, custom libraries/AMIs -> **EMR** (EMR Serverless for serverless Spark/Hive)
- Ad-hoc SQL on S3 data using the catalog -> **Athena**
- No-code visual data cleaning -> **DataBrew**
- Real-time streaming transformations with low latency -> **Managed Service for Apache Flink** (or Glue streaming for micro-batch)
- Simple record transformations on the way into S3/Redshift -> **Firehose** (Lambda transform, format conversion to Parquet/ORC using a Glue table schema)
- Workflow orchestration of many tools -> **Step Functions** / **MWAA**
