# Other Data Analytics Related Products

## AWS Data Exchange

- AWS Data Exchange is a service that helps AWS easily share and manage data entitlements from other organizations at scale
- As a data receiver, we can track and manage all of our data grants and AWS Marketplace data subscriptions in one place
- For data senders, AWS Data Exchange eliminates the need to build and maintain any data delivery and entitlement infrastructure
- Data delivery types: S3 files, **S3 data access** (in-place, no copy), **Amazon Redshift data shares**, **Lake Formation** tables, APIs (API Gateway)
- Third-party data subscriptions (weather, financial, healthcare) via AWS Marketplace, auto-export new revisions to S3
- Exam: "consume third-party datasets with minimal operational overhead" => AWS Data Exchange

## AWS Data Pipeline

> **Status (2026):** AWS Data Pipeline is **closed to new customers since July 25, 2024** (maintenance mode, no new features). Existing customers can keep using it. For new designs use **AWS Glue** (ETL + workflows), **AWS Step Functions**, or **Amazon MWAA** (managed Apache Airflow); EventBridge Scheduler for simple schedules

- AWS Data Pipeline is a web service that we can use to automate the movement and transformation of data
- We you can define data-driven workflows, so that tasks can be dependent on the successful completion of previous tasks
- Components of a data pipeline:
    - Pipeline definition:  specifies the business logic of our data management
    - Pipeline: schedules and runs tasks by creating Amazon EC2 instances to perform the defined work activities
    - Task Runners: polls for tasks and then performs those tasks. For example, Task Runner could copy log files to Amazon S3 and launch Amazon EMR clusters. Task Runner is installed and runs automatically on resources created by your pipeline definitions
- Use case examples:
    - We can use AWS Data Pipeline to archive your web server's logs to Amazon Simple Storage Service (Amazon S3) each day and then run a weekly Amazon EMR (Amazon EMR) cluster over those logs to generate traffic reports

## AWS Lake Formation

- AWS Lake Formation is a service that makes it easy to set up a secure data lake in days
- Creating a data lake with Lake Formation is as simple as defining data sources and what data access and security policies we want to apply
- Helps us collect and catalog data from databases and object storage, move the data into new Amazon S3 data lake, clean and classify data using machine learning algorithms, and secure access to sensitive data
- Built on top of the **Glue Data Catalog** (same databases/tables), adds a permission layer on top of IAM + S3 bucket policies
- Fine-grained permissions with `GRANT`/`REVOKE` style: database, table, **column**, **row** and **cell-level** (data filters)
- **LF-Tags** (tag-based access control, TBAC): assign tags to databases/tables/columns, grant on tags => scales to thousands of tables
- **Cross-account sharing** of tables/databases (via AWS RAM), central governance account pattern for **data mesh** (producer accounts register data, consumer accounts query)
- Enforced by integrated engines: **Athena, Redshift Spectrum, EMR, Glue, Quick Sight**, SageMaker; engines get temporary credentials (credential vending)
- Blueprints: ingest from JDBC databases/logs into S3 (generates Glue workflows)
- Register S3 locations; hybrid access mode lets IAM-based and Lake Formation permissions coexist during migration
- Exam: "centralized fine-grained (column/row) access control on an S3 data lake across accounts" => Lake Formation (not S3 bucket policies)

## AWS Glue

- Serverless **data integration / ETL** service
- **Glue Data Catalog**: central metadata store (databases, tables, schemas, partitions), Hive metastore compatible; used by Athena, Redshift Spectrum, EMR, Lake Formation. One catalog per account per region; can be shared cross-account
- **Crawlers**: scan S3, JDBC, DynamoDB sources, infer schemas and create/update tables and partitions (can run on schedule or on S3 event)
- **ETL jobs**: Apache Spark (PySpark/Scala), Spark streaming (from Kinesis/MSK), Python shell. Billed per **DPU**-hour (per second, 1 min minimum)
  - **Job bookmarks**: process only new data since the last run (incremental)
  - **Flex execution**: cheaper for non-urgent jobs (spare capacity)
  - Auto scaling of workers, worker types G.1X/G.2X/... and G.025X for streaming
- **Glue Studio** (visual ETL), **Glue DataBrew** (no-code data preparation, 250+ transforms), **Glue Data Quality** (rules, DQDL)
- **Glue workflows / triggers** to orchestrate crawlers and jobs (or Step Functions/MWAA)
- **Glue Schema Registry**: schema versioning/validation for streaming (KDS, MSK, Managed Flink), Avro/JSON/Protobuf
- Supports open table formats: Iceberg, Hudi, Delta Lake
- Zero-ETL integrations from SaaS (Salesforce, SAP, ServiceNow, etc.) into S3/Redshift
- Glue vs EMR: Glue = serverless, managed ETL with less control; EMR = full control of big data frameworks, long-running clusters, cost optimization with Spot

## Amazon MSK (Managed Streaming for Apache Kafka)

- Fully managed **Apache Kafka** in our VPC: brokers + controllers (ZooKeeper/KRaft) managed by AWS, deployed across 2-3 AZs
- **MSK Provisioned**:
  - **Standard brokers**: we choose instance type and EBS storage (auto scaling of storage), tiered storage to cheaper long-term storage => effectively unlimited retention
  - **Express brokers** (Nov 2024): up to **3x throughput per broker**, up to **20x faster scaling**, **90% faster recovery**, no storage provisioning (pay per GB), always 3 AZs => for high throughput, elastic Kafka
- **MSK Serverless**: no capacity management, pay per cluster-hour + partition + data; **IAM authentication only**; limits on throughput/partitions per cluster
- **MSK Connect**: managed Kafka Connect connectors (e.g. sink to S3, source from databases with Debezium CDC)
- **MSK Replicator**: managed cross-region / same-region replication between MSK clusters (DR, aggregation)
- Security: IAM access control, SASL/SCRAM (Secrets Manager), mTLS (ACM Private CA); KMS at rest, TLS in transit
- Connectivity: private by default; **multi-VPC private connectivity** (PrivateLink) for cross-account/cross-VPC clients; public access optional
- Integrations: Lambda (event source), Firehose (MSK as source), Managed Service for Apache Flink, Glue streaming, Redshift streaming ingestion, OpenSearch Ingestion
- MSK vs Kinesis: MSK for Kafka compatibility/ecosystem, custom configs, large messages; Kinesis for simplicity and native AWS integrations

## Amazon MWAA (Managed Workflows for Apache Airflow)

- Managed Apache Airflow: DAGs stored in S3, runs in our VPC, auto scaling workers
- Use for complex data pipeline orchestration when teams already use Airflow (Step Functions for AWS-native serverless orchestration)

## AWS Clean Rooms

- Collaborate on combined datasets with partners **without sharing/copying raw data** (analysis rules, differential privacy, cryptographic computing)
- Exam: "analyze joint customer data with a partner while protecting PII" => Clean Rooms
