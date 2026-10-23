# Amazon Aurora

## Aurora Architecture

- Aurora architecture is very different from RDS
- It uses the base entity of a cluster, which something that other instances of RDS database do not have
- Aurora does not use local storage for the compute instances, instead an Aurora cluster has a shared custom volume
- A cluster is made up from a number of important things:
  - A single primary instance + 0-15x replicas
  - The replicas can be used for read during normal operations
- Aurora uses a _Cluster_:
  - Made up of a single primary instance and 0 or more replicas
  - The replicas can be used for reads (not like the standby replica in RDS)
  - Storage: the cluster uses a shared cluster volume (SSD based by default). Provides faster provisioning, improved availability and better performance. Size can go up to **256 TiB** on recent engine versions (Aurora MySQL 3.10+/PostgreSQL 17.5+, 2025), 128 TiB on older versions
  - The storage has 6 replicas across 3 AZs (2 per AZ). The data is replicated synchronously. Replication happens at the storage level, no extra resources are consumed for replication
  - Quorum: writes need 4/6 copies, reads need 3/6 => tolerates losing an entire AZ (+1 copy) for reads, an AZ for writes
  - By default only the primary instance is able to write to the storage, replicas and the primary can perform read operation
  - Replica lag is typically < 100 ms (replicas read the same shared volume)
  - Self-healing: Aurora can repair its data if a replica or part of the replica if there is a disk failure
  - Aurora uses the cluster to repair the data with no corruption. As a result, Aurora avoids data loss and reduces needs for point-in-time restores or snapshot restores
  - With Aurora, you can have up to 15 replicas, any of the replicas can be failed over to
  - Failover: promotes a replica based on **failover priority tiers** (0-15), typically completes in < 30-60 seconds. Without replicas, Aurora recreates the primary (slower, minutes)

![AuroraArchitecture](images/AuroraArchitecture.png)

- Billing for Aurora storage:
  - Storage is billed to what we consume up to 256 TiB limit (128 TiB on older versions)
  - High water mark: we get billed for the most used data at any past point in time, in case of free-up we will be billed for the max usage consumed
  - In case we have to reduce data significantly, we will need to migrate the database to another cluster to avoid paying for the storage
  - Recently Aurora introduced dynamic resizing, where we have to pay for only what we use. It is recommended to upgrade our database to an Aurora version which supports dynamic resizing
- Aurora clusters use endpoints, providing multiple endpoints:
  - **Cluster endpoint**: always point to the primary instance, can be used for reads and writes
  - **Reader endpoint**: load balances (DNS round robin, connection level) across the read replicas. Points to the primary only if there are no replicas
  - **Custom endpoint**: can be created by us, points to a chosen subset of instances (e.g. bigger instances for analytics/reporting). Max 5 per cluster
  - **Instance endpoint**: each instance has its own endpoint
- **Aurora Auto Scaling**: adds/removes Aurora Replicas based on CPU or connections (target tracking). Reader endpoint picks them up automatically

## Aurora Costs

- No free-tier option, Aurora does not support micro instances (smallest are burstable `db.t3/t4g.medium`)
- Beyond RDS singleAZ (micro) Aurora offers better value compared to other RDS options
- Compute: hourly charge, per second, 10 minute minimum
- Storage: GB/month consumed, IO cost per request made to the cluster's shared storage
- Backups: 100% DB size in backups are included
- Storage configurations (can switch: to I/O-Optimized once every 30 days, back to Standard anytime):
  - **Aurora Standard**: pay per I/O request, cheaper for low/moderate I/O
  - **Aurora I/O-Optimized**: no I/O charges, higher instance and storage price. Use when **I/O spend > ~25% of total Aurora spend** => predictable pricing, up to 40% savings for I/O-heavy apps
- Reserved Instances or **Database Savings Plans** (2025, 1-year, flexible across engines/instance families/Regions and serverless) for steady workloads

## Aurora Restore, Clone and Backtrack

- Backups in Aurora work the same way as other RDS
- Automated backups retention 1-35 days (cannot be disabled), PITR to any second in the window
- Restores create a new cluster
- Backtrack can be enabled per cluster. They allow in-place rewinds to a previous point-in-time
  - **Aurora MySQL only**, window up to **72 hours**, no new cluster needed => fast recovery from logical errors (e.g. accidental `DELETE`)
  - Rewinds the entire cluster, brief downtime. Not supported with Global Database / binlog replication in some cases
- Fast clone: makes a new database much faster than copying all the data. Aurora references the original storage, only stores any differences between the old data and the new one
  - Uses **copy-on-write** => cheap and fast test/dev/analytics copies of production
  - Can clone cross-account (shared via AWS RAM), up to 15 clones per copy-on-write chain
  - Clone can be provisioned from serverless and vice-versa
- Snapshots can be shared cross-account and copied cross-region (encrypted snapshots need the KMS key shared)

## Aurora Serverless

> **Status (2026):** Aurora Serverless **v1 reached end of life on March 31, 2025**. All serverless clusters are now **Serverless v2**. The shared warm pool / proxy fleet architecture below describes v1; keep it for concept understanding

- Aurora provisioned vs Aurora Serverless: defined by instance class, can be a specific size (e.g. 2xlarge) for provisioned VS scale by itself
- It provides a version of Aurora without the need to statically provision the database instance
- Removes admin overhead for managing db instances
- Aurora Serverless uses the concept of **ACU - Aurora Capacity Units**: represent a certain amount of compute and a corresponding amount of memory
- We can set min and max ACU values per cluster, can go down to 0
- Consumption billing is on a per-second basis
- Aurora Serverless provides the same resilience as Aurora provisioned (6 copies across AZs)
- Aurora cluster architecture still exists, but in a for of serverless cluster. Instead of using provisioned instances we have capacity units
- Capacity units are allocated from a warm pool (shared) of Aurora instances managed by AWS
- ACUs are stateless, shared across multiple AWS customers
- If the load increases beyond the ACU limit and the pool allows it, than more ACU will be allocated to the instance
- In Aurora Serverless we have shared Proxy Fleet (EC2 instances managed by AWS) for connection management:
  - It is used to distribute connections from us, Aurora users, to Aurora capacity units => We never directly connect to Aurora, this makes Aurora scaling seamless

![AuroraServerless](images/AuroraServerless.png)

- Aurora serverless cluster use cases:
  - Infrequently used applications
  - New applications where we are unsure about the levels of load that will be places on the application
  - Variable and/or unpredictable workloads
  - Development and test databases: Aurora can be configured to stop itself
  - Multi-tenant applications where the scaling is aligned with infrastructure size and revenue

### Aurora Serverless v2

- Serverless v2 is an **instance class** (`db.serverless`) inside a normal Aurora cluster => can **mix provisioned and serverless instances** in the same cluster (e.g. provisioned writer + serverless readers)
- 1 ACU ~= 2 GiB memory + corresponding CPU/network. Scales in **0.5 ACU** increments, in place, in seconds (no connection drops, no scaling-point wait like v1)
- Range: **0 to 256 ACU** on recent versions (0.5-128 on older versions)
- **Scale to 0 / auto-pause** (Nov 2024): min ACU = 0, pauses after an idle period (`SecondsUntilAutoPause`, 5 min to 24 h). Resume takes up to ~15 seconds => dev/test, rarely used apps. Not for apps needing constant low latency
- Supports everything provisioned does: Multi-AZ, read replicas, **Global Database**, RDS Proxy, IAM auth, Performance Insights, Data API
- Readers in failover tier 0-1 scale with the writer (ready for failover)
- Exam: "unpredictable/spiky workload, minimize admin, relational" => Aurora Serverless v2

## Aurora Global Database

- One primary Region (read/write) + up to **10 secondary Regions** (read-only, raised from 5 in May 2025)
- Replication at the **storage layer**, asynchronous, typically **< 1 second** lag, no performance impact on the primary
- Up to 16 replicas per secondary cluster (primary replica count is reduced when using many secondaries)
- **RPO ~1 second, RTO < 1 minute** for cross-region DR
- Recovery options:
  - **Switchover** (formerly "managed planned failover"): planned, **RPO = 0**, for region rotation/DR drills. Cluster topology preserved
  - **Failover** (managed unplanned failover, `FailoverGlobalCluster` with `--allow-data-loss`): during a regional outage, promotes a secondary, RPO = replication lag
  - Detach and promote: manual, secondary becomes standalone cluster
- **Global writer endpoint**: single endpoint that follows the writer across Regions after switchover/failover
- **Write forwarding**: apps in secondary Regions can send writes, forwarded to the primary => simpler multi-region apps (still one writer, extra latency)
- `rpo_lag` (PostgreSQL `aurora_global_db_rpo`) can enforce max lag
- Global Database vs cross-region read replicas: Global DB is faster (storage-level), lower lag and better RTO
- Exam: "cross-region DR with RPO of seconds and RTO of a minute for relational" => Aurora Global Database

## Aurora Limitless Database

- Aurora PostgreSQL only (GA Oct 2024), **horizontal scaling of writes** via automatic sharding
- **DB shard group**: routers (distributed transaction coordinators) + shards, capacity in ACUs (serverless), I/O-Optimized
- Sharded tables (shard key), reference tables (replicated to all shards), standard tables
- Millions of write transactions per second, petabytes of data, single endpoint, ACID across shards
- Use when one writer instance is not enough (beyond the largest instance) and we do not want to shard at the app level

## Aurora DSQL

- Serverless **distributed SQL** database, PostgreSQL-compatible (GA May 2025)
- **Active-active**: single-Region (multi-AZ, 99.99%) or **multi-Region** (99.999%) with **strong consistency** across Regions: 2 peer Regions + 1 **witness Region**, synchronous replication, RPO 0
- No instances, no ACU settings, scales reads/writes/compute/storage independently, scales to zero, pay per DPU + storage
- Optimistic concurrency control (OCC) => apps must retry on conflicts. IAM-based auth tokens
- Subset of PostgreSQL: no foreign key constraints, no triggers, limited extensions, per-transaction row/size limits => not a drop-in for existing PostgreSQL apps
- Multi-Region clusters limited to Regions in the same geographic group
- DSQL vs Global Database vs DynamoDB global tables:
  - Need SQL + multi-region active-active writes + strong consistency => **DSQL**
  - Full PostgreSQL/MySQL compatibility, one writer region, fast DR => **Aurora Global Database**
  - Key-value, massive scale, multi-region writes => **DynamoDB global tables** (MRSC for strong consistency)

## Aurora Integrations and Other Features

- **Zero-ETL integration to Amazon Redshift** (Aurora MySQL/PostgreSQL): near real-time replication to Redshift without pipelines (no DMS/Glue). Also to SageMaker Lakehouse
- **Aurora Parallel Query** (MySQL): pushes query processing to the storage layer for analytics on OLTP data
- **Database Activity Streams**: near real-time stream of DB activity to Kinesis Data Streams, for compliance/auditing (separation of duties from DBAs)
- **Babelfish** for Aurora PostgreSQL: accepts SQL Server T-SQL / TDS protocol => migrate from SQL Server with fewer code changes
- **RDS Data API**: HTTPS endpoint for SQL, no drivers/connections (Lambda friendly), for provisioned and Serverless v2
- **Blue/Green Deployments**: supported for Aurora MySQL and PostgreSQL (major upgrades, schema changes, switchover typically < 1 min)
- **Aurora Optimized Reads**: NVMe-based tiered cache + temp objects on `r6gd/r6id` instances (up to 8x faster queries for large datasets)
- Integration with ML: call SageMaker/Comprehend/Bedrock from SQL; `pgvector` for vector search (RAG with Bedrock Knowledge Bases)
- Native functions to invoke Lambda and load/export to S3 (`LOAD DATA FROM S3`, `aws_s3.query_export_to_s3`)
- **Aurora PostgreSQL / MySQL Extended Support**: same model as RDS, paid after end of standard support of a major version

## Aurora Multi-Master

> **Status (2026):** Multi-master was only available on Aurora MySQL 1 (5.6 compatible), which is end of life. Not available for new clusters. For multiple writers use **Aurora Limitless** (sharding), **Aurora DSQL** (active-active) or **Global Database write forwarding**

- **Only available with Aurora MySQL version 1** => now deprecated
- Default Aurora mode is single-master: one read/write endpoint and 0 or more read replicas
- In contrast with default mode for Aurora, multi-master => all instances are read/write
- **There is no cluster endpoint to use**, the application is responsible for connection to instances in the cluster
- Benefits:
  - Multiple writer endpoints, if we have an application which can failover between endpoints, the failover time can be significantly reduced
  - Fault tolerance can be implemented at the application level, but the application needs to manually load balance between the instances
