# EMR - Elastic Map Reduce

## MapReduce 101

- Is a framework designed to allow processing huge amount of data in a parallel, distributed way
- Data Analysis Architecture: huge scale, parallel processing
- MapReduce has two main phases: map and reduce
- It also has to optional phases: combine and partition
- At high level the process of map reduce is the following:
  - Data is separated into splits
  - Each split can be assigned to a mapper
  - The mapper perform the operation at scale
  - The data is recombined after the operation is completed
- **Apache Hadoop**: framework for storing and processing massive datasets across clusters of compute nodes
- **HDFS (Hadoop File System)**:
  - Traditionally stored across multiple data nodes
  - Highly fault-tolerant - data is replicated between nodes
  - Named Nodes (NameNode): provide the namespace for the file system and controls access to HDFS
  - Block: a segment of data in HDFS, 128 MB by default in Hadoop 2+/EMR (64 MB in old Hadoop 1)
  - YARN: resource manager/scheduler of the Hadoop cluster (runs MapReduce, Spark, Tez jobs)

![MapReduce1](images/MapReduce1.png)

![MapReduce1-2](images/MapReduce1-2.png)

![MapReduce2](images/MapReduce2.png)

![MapReduce3](images/MapReduce3.png)

## Amazon EMR Architecture

- Is a managed implementation of Apache Hadoop, which is a framework for handling big data workloads
- EMR includes other elements such as Spark, HBase, Presto, Flink, Hive, Pig (also Trino, Hudi, Iceberg, Delta Lake, Tez, JupyterHub)
- EMR can be operated long term, or we can provision ad-hoc (transient) clusters for short term workloads
- EMR runs in one AZ only within a VPC using EC2 for compute => NOT HA (but resilient to hardware failure)
  - With **instance fleets** we can list multiple subnets/AZs and EMR picks the best AZ at launch (cluster still lives in one AZ)
  - For DR keep data in S3 (EMRFS) and re-create clusters in another AZ/region
- It can use spot instances, instance fleets, reserved and on-demand instances as well
- EMR is used for **big data processing, manipulation, analytics, indexing, transformation, and more**
- EMR architecture:
  ![EMR architecture](images/EMRArchitecture.png)
  - Each cluster requires at least one **master node** (now called **primary node**). This manages the cluster and distributes workloads and acts as the NAME node within MapReduce (we SSH into this if necessary)
  - Historically we could have only one master node, nowadays we can have 3 master nodes (EMR 5.23+, HA for YARN ResourceManager/HDFS NameNode, automatic failover)
  - **Core nodes**: cluster can have many core nodes. They run the jobs. They are also used for tracking task, we don't want to destroy these nodes (no spot instances)
    - Spot is technically allowed for core nodes but losing them loses HDFS data => use On-Demand/Reserved for primary and core
  - Core nodes also manage the HDFS storage for the cluster. The lifetime of HDFS is linked to the lifetime of the core nodes/cluster
  - **Task nodes**: used to only run tasks. If they are terminated, the HDFS storage is not affected. Ideally we use spot instances for task nodes
  - Storage sources:
    - HDFS
    - EMRFS: file system backed by S3, can persist beyond the lifetime of the cluster. Offers lower performance than HDFS, which is based on local volumes
      - S3 is strongly consistent (since Dec 2020) => EMRFS consistent view is no longer needed
      - Decouples storage from compute => enables transient clusters and Spot task nodes
    - JDBC sources
    - DynamoDB

## EMR Operations, Cost and Security

- Pricing: EC2 (or EKS/Serverless) cost + per-second EMR uplift. Use Spot for task nodes, Graviton instances, Reserved/Savings Plans for long-running clusters
- **Transient clusters** (run steps, then auto-terminate) + data in S3 => cheapest for periodic batch jobs. **Auto-termination** after idle time
- **Managed Scaling**: EMR resizes core/task nodes automatically based on YARN metrics (min/max limits, on-demand vs spot limits)
- **Instance groups** (one instance type per group) vs **instance fleets** (up to 5 instance types per fleet (30 with allocation strategy), mixed On-Demand/Spot targets, capacity-optimized Spot)
- Steps: units of work (Spark job, Hive script) submitted via API/console, or orchestrated with Step Functions / MWAA
- Bootstrap actions: scripts run on nodes at launch (install software). Configurations customize applications
- Security:
  - **Security configurations**: encryption at rest (EBS/local disks, EMRFS with SSE-S3/SSE-KMS/CSE), in transit (TLS)
  - Kerberos authentication, **Lake Formation** fine-grained access (table/column/row), **runtime roles** per step (IAM role per job)
  - Private subnets require S3 gateway endpoint/NAT; **Block Public Access** setting enabled by default for the account
- Glue Data Catalog can be used as the Hive metastore (shared with Athena, Redshift Spectrum)
- **EMR on Outposts**: run clusters on-prem

## Amazon EMR on EKS

- Run Spark jobs on an existing **Amazon EKS** cluster (namespace = virtual cluster)
- Shares the Kubernetes cluster with other apps => better utilization, one operational model for containers
- Faster startup than EC2-based EMR, multiple EMR versions on the same cluster
- Use when the company already standardizes on Kubernetes

## EMR Deployment Options (exam)

- **EMR on EC2**: most control, all frameworks (HBase, Presto/Trino, Flink...), HDFS, long-running or transient
- **EMR on EKS**: Spark on shared Kubernetes infrastructure
- **EMR Serverless**: Spark/Hive without clusters, pay per vCPU/memory used
- **AWS Glue**: serverless Spark ETL with Data Catalog, crawlers, less tuning; best for ETL pipelines
- **Athena**: ad-hoc SQL only, no cluster
- Migrating on-prem Hadoop => EMR + S3 (EMRFS) instead of HDFS, transient clusters per workload

## Amazon EMR Serverless

- It is a deployment option for Amazon EMR that provides a serverless runtime environment
- With EMR Serverless we don't have to configure, optimize, secure or operate clusters to run applications with frameworks such as Spark and Hive
- Job run:
  - Is a request submitted to an EMR Serverless application that the application asynchronously executes and tracks it until completion
  - When a job is submitted we must specify an IAM role that will provide required access for the job to other services such as S3
  - We can submit different jobs at the same time with different runtime roles
- Workers:
  - EMR Serverless internally uses workers to execute our workloads
  - The default size of this workloads depends on the application type and EMR version
  - When we submit a job, EMR Serverless computes the resources that the application needs for the job and schedules workers
  - EMR Serverless automatically scales workers up or down based on the workload and parallelism required at every stage of the job
- Pre-initialized capacity:
  - Used to keep workers initialized and ready to respond in seconds
  - Effectively creates a warm pool of workers for an application
- Application: created for a specific EMR release and engine (**Spark** or **Hive**). Auto-starts on job submission and auto-stops when idle
- **Maximum capacity** limits per application (vCPU, memory, disk) => cost control
- Billing per vCPU-second, GB-memory-second and storage used by workers (no charge when idle unless pre-initialized capacity)
- Can run in a VPC to access private data sources (RDS, Redshift); Graviton (arm64) supported
- Use when: intermittent/variable Spark jobs, no cluster management, no HBase/Presto requirements
- EMR Studio:
  - It is the user console where we manage our EMR Serverless applications
  - Web-based IDE with managed Jupyter notebooks, Git integration, SSO via IAM Identity Center, works with EMR on EC2, EKS and Serverless
