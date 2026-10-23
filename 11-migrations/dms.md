# DMS - Database Migration Service

- Database migrations are complex actions to perform
- DMS is a managed database migration service
- It starts with using a replication instance running on EC2 (or **DMS Serverless**, see below)
- This instance runs one or more replication task
- For these tasks we have to specify the source and destination endpoints at source and target databases.
- **One endpoint must be on AWS!** We can't use the product between on-premises migrations
- Source databases supported are: MySQL, Aurora, Microsoft SQL, MariaDB, MongoDB, DocumentDB, PostgreSQL, Oracle, SAP ASE, Db2, Azure SQL, S3, etc.
- Targets also include non-relational/streaming services: S3, DynamoDB, Kinesis Data Streams, MSK/Kafka, OpenSearch, Neptune, Redshift, DocumentDB, Babelfish for Aurora PostgreSQL
- Source DB stays fully operational during the migration => minimal downtime
- DMS uses jobs to handle migrations. Jobs can be one of 3 types:
  - Full load migrations: used to migrate existing data, simply migrates the data from source to target. Great if we can afford an outage for the source DB
  - Full load + CDC (Change data capture): migrates the existing data and replicates any ongoing changes
  - CDC only: designed to replicate only data changes. In some situations might be more efficient to use other tools for full migration and use CDC only for ongoing changes afterwards
- DMS itself only creates basic target tables (no secondary indexes, FKs, stored procedures, etc.). For heterogeneous schema/code conversion we should use **DMS Schema Conversion** (managed, in the DMS console) or the **Schema Conversion Tool (SCT)** (desktop app)

![DMS](images/DMS.png)

## Replication Instance Details

- Multi-AZ option: synchronous standby replication instance in another AZ for HA of the migration itself
- Instance class sizing matters: CDC and LOB-heavy tables need memory; tasks can be split per table set
- Can be placed in a VPC; source/target reached via VPN, Direct Connect or public endpoints
- Supports SSL to endpoints, KMS encryption of storage, Secrets Manager for endpoint credentials
- Table mappings: selection rules and transformation rules (rename schema/table/column, add columns, filter rows)
- LOB modes: limited LOB (faster, truncates over max size), full LOB, inline LOB
- **Data validation**: compares source and target rows after full load and during CDC
- **Premigration assessments**: check for unsupported data types, missing PKs, CDC prerequisites before running a task

## CDC Requirements (exam favourites)

- MySQL/MariaDB: binary logging enabled with `binlog_format=ROW`, adequate binlog retention
- Oracle: supplemental logging + ARCHIVELOG mode (LogMiner or Binary Reader)
- SQL Server: MS-CDC or MS-Replication enabled, full recovery model
- PostgreSQL: logical replication (`wal_level=logical`), replication slots
- Tables should have primary keys for efficient CDC

## DMS Serverless

- No replication instance sizing: capacity provisioned and scaled automatically in **DCUs** (DMS Capacity Units, 1 DCU = 2 GB RAM), min/max DCU configured
- Multi-AZ supported, pay per DCU-hour
- Good for unpredictable CDC volume or many migrations where sizing is hard

## Homogeneous Data Migrations

- Feature of DMS for same-engine migrations (MySQL => RDS/Aurora MySQL, PostgreSQL => RDS/Aurora PostgreSQL, MongoDB => DocumentDB)
- Uses **native database tools** (e.g. `pg_dump`/`pg_restore`, native replication) under the hood, serverless, full load + CDC
- Created in a DMS **migration project** with instance profile + data providers
- Alternatives for homogeneous: native replication (MySQL binlog replication, Aurora read replica of RDS MySQL), RDS snapshot restore, Percona XtraBackup to S3 => Aurora MySQL, Oracle Data Pump / RMAN

## SCT - Schema Conversion Tool

- SCT is a standalone app used for converting one database engine to another including conversion of schema from a DB to S3
- SCT is not used when migrating between DBs of the same type (for ex when migrating on-prem MySQL => RDS MySQL)
- SCT works with OLTP DBS (MySQL, Oracle, Aurora, etc.) and OLAP databases (Teradata, Oracle, Vertica, Greenplum, Netezza, etc.)
- Example when SCT should be used: on-premises MSSQL -> RDS MySQL (the engine changes from MSSQL to MySQL) or from Oracle -> Aurora
- Generates an **assessment report**: % of automatic conversion, action items that need manual work
- **SCT data extraction agents**: extract data from data warehouses (Teradata, Netezza, Oracle DW, etc.) to S3/Snowball => load into Redshift
- Also converts application SQL embedded in code (Java, C#, C++) and ETL scripts

## DMS Schema Conversion (managed SCT)

- SCT capabilities built into the DMS console (no desktop install), using migration projects, data providers and instance profiles
- **Generative AI** conversion (Amazon Bedrock) for code objects that the rules-based engine can't convert (stored procedures, functions, triggers); expanded to more regions in 2026
- AI agent automation via the AWS MCP Server (Jul 2026): coding agents can drive conversion workflows
- SCT desktop still exists and covers more source engines and DW extraction agents

## Discovery / Assessment

- **DMS Fleet Advisor** (database inventory and target recommendations):
  > **Status (2026):** closed to new customers May 20, 2025 and reached end of support on May 20, 2026. Use **Migration Evaluator** / AWS Transform assessment instead

## DMS and Snowball

> **Status (2026):** Snowball Edge is no longer available to new customers (Nov 7, 2025). For new projects prefer DMS over Direct Connect/VPN, or SCT extraction agents + DataSync/Data Transfer Terminal to S3

- Larger migrations might imply moving databases with sizes of multi-TB
- Moving data over networks takes time and consumes capacity
- DMS is able to utilize Snowball products to migrate databases
- Migration steps:
  1. Use SCT to extract data locally and move the data to a Snowball
  2. Ship the device back to AWS. They will load the data into an S3 bucket
  3. DMS migrates from S3 into a target source
  4. Change Data Capture (CDC) can capture changes and via S3 intermediary they are also written to the target database

## Exam Tips

- Heterogeneous migration (Oracle => Aurora PostgreSQL, SQL Server => Aurora MySQL) => DMS Schema Conversion/SCT first, then DMS full load + CDC
- Homogeneous migration => DMS (or homogeneous data migrations) without SCT, or native tools
- "Minimal downtime" => Full load + CDC, then cutover when lag ~0
- DMS to Kinesis/MSK => stream DB changes into analytics pipelines (CDC as event source)
- DMS to S3 (Parquet/CSV) => build a data lake from operational DBs
- Continuous replication between on-prem and RDS for DR/hybrid => DMS CDC (or native replication)
- Licensing cost reduction (Oracle/SQL Server => open source) => Replatform with DMS + Schema Conversion, possibly Babelfish for SQL Server apps
