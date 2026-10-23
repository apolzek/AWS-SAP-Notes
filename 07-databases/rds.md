# RDS - Relational Database Service

- RDS is often described as a Database-as-a-service (DBaaS) but this is not accurate. It should be named Database Server as a Service (DBSaaS) product
- RDS provides managed database instances, which can themselves hold one **or more** databases
- Benefits of RDS are that we don't need to manage the physical hardware, the server operating system or the database system itself
- RDS supports MySQL, MariaDB, PostgreSQL, Oracle, Microsoft SQL Server, **IBM Db2** (since 2023). Aurora is part of RDS but has its own architecture
- RDS Subnet Group: list of subnets which an RDS database can use. Generally it is best practice to have one Subnet Group per database deployment
- Each RDS instance (within a multiAZ deployment) has its own dedicated EBS storage (different for Aurora)

![RDSArchitecture](images/RDSArchitecture.png)

## RDS Database Instance

- Runs one of the few types of db engine mentioned above
- Can contain multiple user created databases
- A database instance after creation can be accessed using its hostname (CNAME)
- RDS instances come in various types, share many of features of EC2. Example of instances: db.m5, db.r5, db.t3 (current gen: Graviton `db.m7g`, `db.r7g`, `db.r8g`, `db.t4g`)
- RDS instances can be single AZ or multi AZ (active-passive failover)
- When an instance is provisioned, it will have a dedicated storage allocated as well (EBS)
- Storage allocated can be based on SSD storage (IO1, GP2) or magnetic (mainly for compatibility)
  - Current options: **gp3** (baseline 3,000 IOPS / 125 MiBps, IOPS and throughput configurable independently of size), **io2 Block Express** (up to 256,000 IOPS, sub-ms latency), io1, gp2. Magnetic is legacy
  - Max storage **64 TiB** for most engines
  - **Storage autoscaling**: set a max storage threshold, RDS grows storage automatically when free space is low (no downtime)
- Billing for RDS:
  - We are billed based on instance size on a hourly rate
  - We are billed for additional instances used for Multi AZ deployments
  - We are also billed per storage (GB/month) + extra per iops in case of provisioned iops (IO1)
  - Data transfer is also billed if data is coming from/goes to the internet/other regions
  - Backups and snapshots are also billed (per GB per month)
  - Licensing if applicable is also billed (License Included vs BYOL for Oracle/SQL Server/Db2)
  - **Extended Support** charges (see below) for major versions past end of standard support
  - Savings: Reserved Instances (1/3 years) or **Database Savings Plans** (2025, 1-year, flexible across engines, instance families, Regions)
- An instance can be **stopped for up to 7 days**, then RDS starts it automatically (exam: use snapshot + delete for long idle periods, or Aurora Serverless v2 scale to 0)

## RDS Multi AZ

- They are 2 types of Multi AZ deployments:
  - Multi AZ Instance (historically called Multi AZ)
  - Multi AZ Cluster
- Multi AZ Instance:
  - Used to add resilience to an RDS instance
  - Replication happens at the storage level
  - Enables **synchronous** replication between primary and standby instances
  - Multi AZ is an option which can be enabled on an RDS instance, when enabled secondary hardware is allocated in another AZ (standby replica)
  - RDS is accessed via provided endpoint address (CNAME)
  - With a single instance the endpoint address points the instance itself, with multi AZ, by default the endpoint points to the primary instance
  - We can not directly access the standby instance
  - If an error occurs with the primary instance, RDS automatically changes the endpoint to point to the standby replica. This failover occurs in around 60-120 seconds
  - Multi AZ is not available in the Free-tier (generally costs double as it would the single AZ)
  - Backups are taken from the standby instance (removes performance impact)
  - In case of a failover the DNS name will be updated to point to the standby replica instance. Since this is a DNS change, for the update it generally takes between 60-120 seconds to occur. This can be shortened by removing DNS caching in the application
  - With Multi AZ instance we have ONE standby replica. This replica cannot be used for read and writes. It waits for a failover to happen and then it can be used
  - Backups can be taken from the standby instance to improve performance
  - Converting Single-AZ to Multi-AZ is an online operation (snapshot of primary restored in the other AZ, then sync)
  - Standby is in the **same region** only. For cross-region use read replicas or cross-region automated backups
  - Failovers can happen if:
    - AZ outage
    - Primary instance failure
    - Manual failover (reboot with failover)
    - Instance type change
    - Software patching
  - Failover events can be captured via **RDS Event Notifications** (EventBridge/SNS)
- Multi AZ Cluster:
  - RDS is capable of having one writer replicate into two reader instances (**synchronously**). We can have 2 readers only!
  - Only for **RDS for MySQL and RDS for PostgreSQL**
  - These readers are in different AZs compared to the writer (3 AZs total)
  - Compared to Aurora cluster mode, Multi AZ cluster can have 2 readers only, while Aurora Cluster can have more
  - In case of Multi AZ cluster the instances to which data is replicated are usable, compared to Multi AZ instance mode when they are not
  - In terms of replication the data is viewed as committed when one of the readers confirms that it was written (**semi-synchronous**: commit acknowledged by at least 1 reader)

![RDSCluster](images/RDSCluster.png)

- Other comparisons to Aurora Cluster:
  - In RDS multi AZ cluster each instance has its own storage, in case of Aurora this is not the case
  - Like Aurora, the cluster can be accessed with multiple endpoints:
    - Cluster endpoint: database CNAME, points to the writer, can be used for reads/writes and administration
    - Reader endpoint: points to any available endpoint for reads (it can point to the writer instance in certain cases). Generally it points to the dedicated reader instances
    - Instance endpoints: each instance gets one endpoint, generally not recommended to be used
- Generally Multi AZ Cluster runs on faster hardware: Graviton + local NVME SSD storage. Any writes are written to local super fast storage, after that they are flushed to the EBS
- Replications are done via transaction logs => much more efficient then Multi AZ instance. This also allows faster failover: ~35 seconds + any time required to apply the transaction logs
- Lower write latency (up to 2x faster commits) vs Multi-AZ instance
- Exam: "RDS MySQL/PostgreSQL, failover < 35 s, readable standbys, no Aurora" => Multi-AZ DB cluster

## RDS Backups and Restores

![RDSBackup](images/RDSBackup.png)

- RPO (Recovery Point Objective): time between the last working backup and the failure. Lower the RPO value, usually the more expensive the solution
- RTO (Recovery Time Objective): time between the failure and system being fully recovered. Can be reduced with spare hardware, predefined processes, etc. Lower the RTO value, the system is usually more expensive
- RDS backup types:
  - Manual snapshots:
    - Have to be run manually, or via a script
    - First snapshot is full content of the DB, incremental onward
    - When any snapshot occurs, there is brief interruption in the flowing of data between the compute resource and the storage (no noticeable effect in case of Multi AZ, since the backup is taken from the standby instance)
    - Manual snapshots do not expire
    - When we delete an RDS instance, AWS offers to make one final snapshot
    - Can be shared with other accounts (or made public, if unencrypted). Encrypted snapshots can be shared only with a **customer managed KMS key** (not the AWS managed `aws/rds` key)
    - Can be copied cross-region (re-encrypted with a KMS key in the destination region)
    - Default quota: 100 manual snapshots per region (AWS Backup snapshots don't count)
    - Can be **exported to S3 in Parquet** format for analytics (Athena, EMR)
  - Automatic backups:
    - They occur once per day (backup window is defined on the instance)
    - Snapshots which occur automatically, first being full snapshot, incremental afterwards
    - In addition to the automated snapshots, every 5 minute transaction logs are written to S3
    - Automatic backups are not retained, we can set the retention period between 0 and 35 days (0 = disabled, also disables read replicas)
    - Automatic backups can be retained after a DB is deleted, but they still expire after the retention period
    - We can replicate backups to another region: both snapshots and transaction logs can be replicated. Charges apply to cross-region data copy and any storage used in the destination region
    - Cross-region replication has to be explicitly configured within automated backups. **NOT THE DEFAULT**
    - **Cross-Region automated backups** allow PITR in the DR region (cheap cross-region DR: no running instance, RTO = restore time)
- Backups are stored in AWS manages S3 buckets (backups are not visible to us directly in S3) => any data in S3 is regionally resilient
- RDS backups are taken from the standby instance in case Multi AZ is enabled
- **AWS Backup** can centrally manage RDS/Aurora backups: backup plans, cross-account and cross-region copy, Vault Lock (WORM) for compliance
- **RDS Restores**
  - RDS creates a new RDS instance when we restore an automated backup or a manual snapshot => new address will be created for the DB
  - When we restore a snapshot, we restore our DB to a single point in time, when the creation time of the snapshots happened
  - With automated backups we can chose a point-in-time to where we want to restore (any 5 minute point-in-time => good RPO)
  - Restoring snapshots is not a fast procedure (important for RTO)
  - Restore from S3: MySQL (Percona XtraBackup files), SQL Server (native `.bak` files) => migration path from on-prem

## RDS Read-Replicas

- Provide 2 main benefits: performance and availability
- Read replicas are read-only replicas of an RDS instance
- Multi AZ Cluster mode is similar to how read replicas work, but for read replicas we have to think of read replicas as separate things:
  - They are not part of the main database instance
  - They have their own endpoint address
  - Require application support
  - There is no automatic failover to a read replica
- The primary instance and read replica is kept sync using **asynchronous** replication
- There can be a small amount of lag in case of replication
- Read replicas can be created in a different AZ or different region (CRR - Cross-Region Replication)
- We can create up to **15 direct** read-replicas per DB instance (default quota, historically 5; AWS recommends max 5 for Oracle to limit lag)
- Automated backups must be enabled on the source to create read replicas
- Each read-replica provides an additional instance of read performance
- Read-replicas can also have read-replicas, but lag starts to be a problem in this case (MySQL, MariaDB, some PostgreSQL versions only; not Oracle, SQL Server, Db2)
- A read replica can itself be **Multi-AZ** (good for DR in another region)
- No data transfer charge for replication within the same region; cross-region replication is charged
- Oracle (mounted) and Db2 (standby) replicas can be non-readable, used only for cross-region DR
- RDS does **not** auto scale read replicas (unlike Aurora Auto Scaling)
- Read-replicas can provide global performance improvements
- Snapshots and backups improve RPO but not RTO.
- Read-replicas offer near 0 RPO => **only from failure, corrupted data would be replicated!**
- Read-replicas can be promoted to primary in case of a failure. This offers low RTO as well (lags of minutes)
- Promotion is one way: the replica becomes a standalone instance, replication is broken, application must be pointed to the new endpoint
- Read-replicas can replicate data corruption
- Exam: Multi-AZ = HA (sync, same region), Read Replica = scaling reads + cross-region DR (async)

## RDS Data Security

- With all the RDS engines we can use encryption in transit (SSL/TLS). This can be set to be mandatory on a per user basis (e.g. `rds.force_ssl` for PostgreSQL/SQL Server, `require_secure_transport` for MySQL)
- For encryption at rest RDS supports EBS volume encryption using KMS which is handled by the host EBS and it is invisible for the database engine
- We can use customer-managed or AWS generated KMS data keys for encryption at rest
- Storage, logs and snapshots will be encrypted with the same KMS key (formerly called customer master key)
- Encryption can not be removed after it is activated
- Encryption must be chosen at creation. To encrypt an existing unencrypted DB: snapshot => copy snapshot with encryption enabled => restore => switch app (or use DMS for minimal downtime)
- Read replicas must have the same encryption state as the primary; cross-region replicas use a KMS key of the destination region
- In addition to encryption at rest MSSQL and Oracle support **TDE (Transparent Data Encryption)** - encryption at the database engine level => more secure
- Oracle supports TDE with CloudHSM, offering much stronger encryption

![RDSEncryption](images/RDSEncryption.png)

- IAM authentication with RDS:
  - Normally login is controlled with local database users (username/password)
  - We can configure RDS to allow IAM authentication (only authentication, not authorization, authorization is handled internally!):
  - Supported for MySQL, MariaDB, PostgreSQL (and Aurora). Uses a token generated by `generate-db-auth-token`, valid **15 minutes**
  - Policy action `rds-db:connect` on the DB user resource

![RDSIAM](images/RDSIAMAuthentication.png)

- **Kerberos / AWS Managed Microsoft AD** authentication: SQL Server, Oracle, MySQL, PostgreSQL, Db2 (can use on-prem AD via trust)
- **Secrets Manager integration**: RDS can manage the master user password in Secrets Manager with automatic rotation (`--manage-master-user-password`)
- Network: SGs, private subnets, `PubliclyAccessible` flag. Access from other VPCs via peering/TGW/PrivateLink (RDS Proxy or NLB)
- **Database Activity Streams** (Oracle, SQL Server and Aurora) for auditing to Kinesis

## RDS Proxy

- Opening and closing connections consumes resources and takes time => in case we only want to read/write a tiny amount of data the overhead of establishing a connection creates a significant latency
- Handling failure of databases instances is hard, this adds significant overhead and risks to our application
- With a serverless application based on Lambda => every function would require setting up a new connection => a proxy can save up time and resources
- DB proxies can help, but managing them is not trivial (scaling, resilience)
- In case of an RDS proxy our application connects to the proxy, which handles connection pooling and connectivity to the database
- RDS proxies provide multiplexing: a smaller number of connections can be used to connect to the database while having a larger number of applications using the database through the proxy. This helps to reduce the load on the database
- RDS Proxy can help with database failover events abstracting this from the applications. The proxy can wait until a healthy database instance is in place and can automatically connect to it
- When to use RDS proxy?
  - When having `Too many connections` errors. An RDS proxy can reduce the number of connections to the database while being able to handle many more connections from the applications to itself. Particularly likely with small/burst DB instances using T2/T3.
  - Useful when using AWS Lambda, we won't need to invoke a new connection after each invocation of our function. Saves time by connection reuse and IAM auth
  - Useful for long running applications (SAAS apps) by reducing latency
- RDS Proxy key facts:
  - Fully managed by RDS/Aurora
  - By default provides auto scaling, HA
  - Provides connections pooling, which reduces DB load
  - Only accessible from a VPC, not accessible from the public internet
  - Accessed via Proxy Endpoint
  - Can enforce SSL/TLS connection
  - Can reduce failover time by over 60% in case of Aurora (up to 66% for Aurora and RDS Multi-AZ)
  - Abstracts the failure of a database away for our application
  - Supported engines: RDS MySQL, MariaDB, PostgreSQL, SQL Server and Aurora MySQL/PostgreSQL
  - Credentials stored in **Secrets Manager**, apps can authenticate to the proxy with **IAM** (enforce IAM auth even if the DB uses passwords)
  - Read-only proxy endpoints can target Aurora readers
  - Cross-VPC access via additional proxy endpoints
  - Not useful for: Aurora DSQL, DynamoDB; and adds cost (per vCPU/ACU of the DB)

![RDSProxy](images/RDSProxy.png)

## RDS Blue/Green Deployments

- Creates a **staging (green) environment** that is a copy of production (blue), kept in sync via **logical replication**
- Use for major version upgrades, parameter/schema changes, instance class changes with minimal downtime
- **Switchover** typically **< 1 minute**, guardrails (replication lag, health checks), endpoints are renamed so apps don't need changes, no data loss
- Supported for RDS MySQL, MariaDB, PostgreSQL and Aurora MySQL/PostgreSQL
- Exam: "upgrade major version with minimal downtime and easy testing" => Blue/Green Deployments

## RDS Extended Support

- When a major version reaches end of standard support (e.g. MySQL 5.7, PostgreSQL 11/12), instances are **automatically enrolled** in Extended Support (up to 3 years) **at an additional cost** (per vCPU-hour, price increases in year 3)
- Applies to RDS MySQL/PostgreSQL and Aurora MySQL/PostgreSQL
- Can opt out only at create/restore time (`--engine-lifecycle-support open-source-rds-extended-support-disabled`) => RDS upgrades automatically at end of standard support
- Exam/cost trap: unexpected RDS cost increase on old engine versions => upgrade (use Blue/Green)

## RDS Performance Features

- **Optimized Writes** (MySQL 8.0.30+, MariaDB): atomic 16 KiB writes, no InnoDB doublewrite buffer => up to 2x write throughput
- **Optimized Reads** (MySQL, MariaDB, PostgreSQL): instances with local NVMe SSD (`d` classes, e.g. `r6gd`) hold temporary tables => up to 2x faster complex queries
- **CloudWatch Database Insights** (Standard / Advanced mode): DB load analysis, top SQL, wait events, fleet-level monitoring
  > **Status (2026):** the Performance Insights console reached end of life on **July 31, 2026** and redirects to CloudWatch Database Insights (PI API remains)
- **Enhanced Monitoring**: OS-level metrics (agent on the host), up to 1 s granularity
- **Zero-ETL integration to Redshift**: RDS MySQL, RDS PostgreSQL, RDS Oracle (and Aurora) => near real-time analytics without ETL pipelines
- **RDS on Outposts**: run RDS on-prem (MySQL, PostgreSQL, SQL Server) for low latency/data residency

## RDS Custom

- Fills the gap between the main RDS product and EC2 running a DB engine
- RDS normally is a fully managed database service => OS/Engine access is limited
- In contrast, databases running on EC2 are self-managed, this can have significant management overhead
- **RDS Custom** bridges this gap, we can utilize RDS but still get access to customization we would have when running a DB instance on EC2
- Currently only available for MicrosoftServer SQL and Oracle
- We can connect to the underlying OS using SSH, RDP or Session Manager
- RDS custom will run withing our AWS account. Classic RDS will run in an AWS managed environment
- If we need to perform RDS customization for RDS Custom, we need to look inside the Database Automation settings to make sure we wont have any disruption caused by the Database Automation. We need to pause Database Automation for this period
- Use cases: legacy/packaged apps (e.g. Oracle E-Business Suite, SharePoint) requiring OS access, custom patches, third-party agents
- Exam: "managed RDS but need OS/sysadmin access for Oracle/SQL Server" => RDS Custom. Other engines with OS access => EC2

## RDS vs Aurora vs EC2 (quick decision)

- Need full OS control or unsupported engine/version => EC2 (self-managed)
- Oracle/SQL Server with OS access => RDS Custom
- Standard managed relational, cost-sensitive, small/medium => RDS
- Higher performance, 15 low-lag replicas, fast failover, global DR, serverless => Aurora
