# Amazon Athena

- It is a serverless interactive querying service
- We can take data stored in S3 and perform ad-hoc queries on the data paying only for the data consumed
- Athena uses a process named **Schema-on-read** - table-like translation
- Original data in S3 is never changed, it remains in its original form. It is translated to the predefined schema when it is read for processing
- Supported formats by Athena: XML, JSON, CSV/TSV, AVRO, PARQUET, ORC, Apache, CloudTrail, VPC Flowlogs, etc. Supports standard formats of structured data, semi-structured and unstructured data
- "Tables" are defined in advance in a data catalog and data is projected through when read. It allows SQL-like queries on data without transforming source data
- Athena has no infrastructure. We don't need set up anything in advance
- Athena is ideal for situations where loading/transforming data isn't desired
- It is preferred for querying AWS logs: VPC Flow Logs, CloudTrail, ELB logs, cost reports, etc.
- Can query data form Glue Data Catalog and Web Server Logs
- Athena Federated Query: Athena now supports querying other data sources than S3. Requires a data source connector (AWS Lambda)

![Athena](images/Athena.png)

![AthenaDemo](images/AthenaDemo.png)

## Pricing and Performance

- On-demand pricing: **$5 per TB scanned** (10 MB minimum per query). DDL and failed queries are free
- Reduce cost/scan = improve performance:
  - Columnar formats (**Parquet/ORC**) => read only needed columns (up to 90%+ less scanned)
  - Compression (Snappy, GZIP, ZSTD)
  - **Partitioning** (e.g. `year/month/day`) + **partition projection** (partitions computed from config, no `MSCK REPAIR TABLE`/crawler needed for highly partitioned data)
  - Bigger files (> 128 MB) rather than many small files
- **Query result reuse**: returns cached results of identical queries (configurable max age) => no scan cost
- Results are written to an S3 bucket (or managed query results storage)

## Workgroups and Provisioned Capacity

- **Workgroups**: separate users/teams/apps, enforce result location + encryption, per-query and per-workgroup **data usage limits** (cost control), CloudWatch metrics, separate billing via tags
- **Provisioned capacity** (capacity reservations): dedicated compute measured in **DPUs** (1 DPU = 4 vCPU, 16 GB). Billed per DPU-hour instead of per TB scanned
  - Minimum is now **4 DPU** with 1-minute reservations (2026, previously 24 DPU / 1 hour)
  - Use for predictable cost, guaranteed concurrency, isolating critical workloads
  - Queries are queued if all DPUs are in use

## Federated Query

- Uses **Lambda-based data source connectors** to query DynamoDB, RDS/Aurora, Redshift, DocumentDB, OpenSearch, CloudWatch Logs, on-prem JDBC sources, other clouds, etc.
- Can JOIN S3 data with operational data in one SQL query, results can be written back to S3 (`CTAS`)
- Connectors can also be registered as Glue Data Catalog connections
- Exam: "query data across multiple sources without moving it, serverless" => Athena Federated Query

## Table Formats and Other Features

- **Apache Iceberg** support: ACID transactions, `INSERT`/`UPDATE`/`DELETE`/`MERGE`, time travel, schema evolution, `OPTIMIZE`/`VACUUM` => use for GDPR deletes / upserts on a data lake
- Also queries Apache Hudi and Delta Lake tables, and **S3 Tables** (managed Iceberg tables)
- `CTAS` (Create Table As Select) and `INSERT INTO`: convert CSV/JSON to Parquet, partition data
- **Athena for Apache Spark**: run PySpark interactively in notebooks, serverless, billed per DPU-hour
- Athena ML: invoke SageMaker endpoints from SQL; UDFs via Lambda
- Security: IAM + **Lake Formation** fine-grained (column/row/cell-level) permissions, SSE-S3/SSE-KMS/CSE-KMS for results, cross-account Glue Catalog access
- Engine based on Trino/Presto (engine v3)

## Athena vs Others

- Athena: ad-hoc, serverless SQL on S3, infrequent queries, logs analysis
- Redshift (Spectrum): frequent, complex BI, high concurrency, joins with warehouse data, predictable performance
- EMR: full control, custom big data frameworks, heavy ETL
- S3 Select: deprecated for new customers (2024) => use Athena
- OpenSearch: full-text search, near real-time log analytics with dashboards
