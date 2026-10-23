# Amazon Redshift

- It is petabyte scale data warehouse
- It is designed for reporting and analytics
- It is an OLAP (column based) database, not OLTP (row/transaction, like RDS)
  - **OLTP (Online Transaction Processing)**: capture, stores, processes data from transactions in real-time
  - **OLAP (Online Analytical Processing)**: designed for complex queries to analyze aggregated historical data from other OLTP systems.
- OLAP: Data is stored in columns => better to query a specific attributes without having to get all rows and then select a column.
- Pay as you use (similar to RDS)
- Advanced features of Redshift:
  - RedShift Spectrum: allows querying data from S3 without loading it into Redshift platform
  - Federated Query: directly query data stored in remote data sources
- Redshift integrates with Quicksight for visualization
- It provides a SQL-like interface with JDBC/ODBC connections (JDBC: Java DB Connectivity API library, ODBC: Open DB Connectivity)
- Redshift is a provisioned product, it is not serverless (AWS offers Redshift Serverless option as well). It does come with provisioning time
  - Provisioned clusters can be **paused/resumed** (pay only storage while paused)
- It uses a cluster architecture. A cluster is a private network, and it can not be accessed directly
- **Redshift runs in one AZ, not HA by design**
  - Update: **Multi-AZ deployments** for RA3 clusters (GA Nov 2023): compute in 2 AZs behind one endpoint, **99.99% SLA**, automatic failover, no extra storage cost (managed storage is shared). Requires encryption
  - **Cluster relocation** (RA3): moves the cluster to another AZ on AZ issues, keeps the same endpoint (free, no extra compute)
- Cluster:
  - **Leader node** with which we can interact in order to do querying, planning and aggregation
  - **Compute nodes**: perform queries on data. A compute node is partitioned into slices. Each slice is allocation a portion of memory and disk space, where it processes a portion of workload. Slices work in parallel, a node can have 2, 4, 16 or 32 slices, depending on the resource capacity
- Redshift is a VPC service, it uses VPC security: IAM permissions, KMS encryption at rest, CloudWatch monitoring
- Redshift Enhanced VPC Routing: setting that can be enabled
  - By default Redshift uses public routes for traffic when communicating with external services or any public AWS service (such as S3)
  - When enabled, traffic is routed based on the VPC networking configurations (SG, ACLs, etc.)
  - Traffic is routed based on the VPC networking configuration
  - Traffic can be controlled by security groups, it can use network DNS, it can use VPC gateways

![Redshift architecture](images/RedshiftArchitecture.png)

## Redshift Components

- **Cluster**: a set of nodes, which consists of a leader node and >=1 compute nodes
  - Redshift creates one database when we provision a cluster. This is the database we use to load data and run queries on your data
  - We can scale the cluster in or out by adding or removing nodes. Additionally, we can scale the cluster up or down by specifying a different node type
  - Redshift assigns a 30-minute maintenance window at random from an 8-hour block of time per region, occurring on a random day of the week. During these maintenance windows, the cluster is not available for normal operations
  - Redshift launches clusters in a VPC only (EC2-Classic was retired in 2023). We create a cluster subnet group if you are provisioning our cluster in our VPC, which allows us to specify a set of subnets in our VPC
  - Resize: **elastic resize** (minutes, add/remove nodes or change type within limits) vs **classic resize** (hours, new cluster, read-only during resize)
- **Redshift Nodes**:
  - The leader node receives queries from client applications, parses the queries, and develops query execution plans. It then coordinates the parallel execution of these plans with the compute nodes and aggregates the intermediate results from these nodes. Finally, it returns the results back to the client applications
  - Compute nodes execute the query execution plans and transmit data among themselves to serve these queries. The intermediate results are sent to the leader node for aggregation before being sent back to the client applications
  - Node Type:
    - Dense storage (DS) node type – for large data workloads and use hard disk drive (HDD) storage => **DS2 retired**, migrate to RA3
    - Dense compute (DC) node types – optimized for performance-intensive workloads. Uses SSD storage => DC2 is previous generation
    - **RA3** (recommended): **Redshift Managed Storage (RMS)** => compute and storage scale independently, hot data cached on local SSD, rest in S3. Required for data sharing, Multi-AZ, relocation, cross-AZ features. Sizes: `ra3.large`, `ra3.xlplus`, `ra3.4xlarge`, `ra3.16xlarge`
    - AQUA (hardware-accelerated cache) is now built into RA3 automatically
- **Parameter Groups**: a group of parameters that apply to all of the databases that we create in the cluster. The default parameter group has preset values for each of its parameters, and it cannot be modified

## Redshift Resilience and Recovery

- Redshift can use S3 for backups in the form a snapshots
- There are 2 types of backups:
  - **Automated backups**: occur every 8 hours or after every 5 GB per node of data changes, by default having 1 day retention (configurable, up to 35 days). Snapshots are incremental
  - **Manual snapshots**: performed after manual triggering, no retention period (forever)
- Restoring from snapshots creates a brand new cluster, we can chose a working AZ to be provisioned into
- We can copy snapshots to another region where a new cluster can be provisioned
  - **Cross-region snapshot copy** is configured on the cluster (automatic). For KMS-encrypted clusters we need a **snapshot copy grant** for a KMS key in the destination region
- Snapshots can be shared with other accounts; restore a single table from a snapshot is possible
- Serverless: recovery points every 30 min (kept 24 h) + snapshots
- Copied snapshots also can have retention periods

![Redshift Resilience and Recovery](images/RedshiftDR.png)

## Amazon Redshift Workload Management (WLM)

- Enables users to flexibly manage priorities within workloads so that short, fast-running queries won’t get stuck in queues behind long-running queries
- Amazon Redshift WLM creates query queues at runtime according to service classes, which define the configuration parameters for various types of queues, including internal system queues and user-accessible queues
- From a user perspective, a user-accessible service class and a queue are functionally equivalent
- **Automatic WLM** (recommended): Redshift manages memory/concurrency, with **query priorities** (highest...lowest). Manual WLM: up to 8 queues, fixed slots/memory
- **Short Query Acceleration (SQA)**: runs short queries in a dedicated space so they don't wait behind long ones
- Query monitoring rules (QMR): log/hop/abort queries exceeding thresholds

## Redshift Serverless

- No clusters: **namespace** (databases, users, storage) + **workgroup** (compute, network, RPUs)
- Compute in **RPUs** (Redshift Processing Units): base capacity **4 to 512 RPUs** (1024 in some regions, default 128), billed per RPU-second only while queries run (60 s minimum)
- **AI-driven scaling and optimization**: set a price-performance target instead of base RPUs
- Max RPU-hours usage limits for cost control
- Use for unpredictable/intermittent analytics, dev/test, new workloads; provisioned RA3 + Reserved Instances for steady 24/7 workloads

## Redshift Scaling and Performance Features

- **Concurrency Scaling**: adds transient clusters automatically for bursts of concurrent read (and some write) queries; **1 hour of free credits per day** per cluster accrued, then per-second billing
- **Distribution styles**: `AUTO`, `EVEN`, `KEY` (co-locate join keys on the same slice), `ALL` (copy small dimension tables to every node)
- **Sort keys** (compound/interleaved/AUTO) => zone maps skip blocks; columnar compression (encodings)
- **Materialized views** (with auto refresh, incremental refresh), result caching
- Automatic table optimization, auto `VACUUM`/`ANALYZE`
- Loading: `COPY` from S3 (parallel, split into multiple files = multiple of slices, compressed), DynamoDB, EMR, SSH. Avoid single-row `INSERT`s. `UNLOAD` to S3 (Parquet)
- **Redshift ML**: `CREATE MODEL` with SQL (SageMaker Autopilot behind the scenes), also LLMs via Bedrock

## Redshift Data Integration Features

- **Redshift Spectrum**: query S3 data (Parquet/ORC/CSV, Iceberg/Hudi/Delta) via external tables in the Glue Data Catalog, without loading; scales with Spectrum fleet, billed per TB scanned. Cluster and S3 bucket must be in the same region
- **Federated Query**: query live data in RDS/Aurora PostgreSQL and MySQL without ETL
- **Streaming ingestion**: directly from **Kinesis Data Streams** and **MSK** into materialized views (low latency, no Firehose/S3 staging)
- **Zero-ETL integrations**: Aurora MySQL/PostgreSQL, RDS MySQL/PostgreSQL/Oracle, DynamoDB, SaaS apps => near real-time replication into Redshift without pipelines
- **Data sharing** (RA3/Serverless only): share live, transactionally consistent data across clusters/workgroups, **accounts** and **regions** without copying. Consumers pay their own compute. Supports writes via data shares (multi-warehouse writes). Can be governed with Lake Formation and offered via AWS Data Exchange
- `auto-copy` from S3: continuously ingest new files from an S3 prefix
- **Redshift Data API**: run SQL over HTTPS without drivers/persistent connections (Lambda, Step Functions)
- Security: IAM, IAM Identity Center, database roles, **row-level security**, column-level grants, dynamic data masking, audit logging to S3/CloudWatch

## Redshift vs Others (exam)

- Structured data warehouse, complex joins/BI at scale, high concurrency => Redshift
- Ad-hoc, infrequent queries directly on S3, no infra => Athena
- Join warehouse tables with S3 data lake => Redshift Spectrum
- Share data between business units in different accounts without copying => Redshift data sharing
- OLTP => RDS/Aurora/DynamoDB (Redshift is OLAP, not for transactional workloads)
