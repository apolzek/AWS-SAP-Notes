# EFS - Elastic File System

- It is a network based file system which can be mounted on Linux based instance
- Can be mounted to multiple instances at once
- EFS is an implementation of the NFSv4 file system format (NFSv4.0 and v4.1), POSIX compliant
- **Linux only** - not supported on Windows instances (Windows => FSx for Windows File Server)
- EFS file systems can be mounted in a folder in Linux based operating systems
- EFS storage exists separately from the lifecycle of an EC2 instance
- It can be shared between many EC2 instances (thousands of concurrent NFS clients)
- Elastic capacity: grows/shrinks automatically up to petabytes, pay for what is stored (no provisioning)
- It is a private service, it can be mounted via mount targets inside a VPC. By default an EFS it is isolated to the VPC in which was provisioned
- Possible to choose Regional (Multi-AZ) vs One Zone (Single-AZ) EFS. Single-AZ still implements data replication but within single-AZ
  - One Zone is ~47% cheaper, but data is lost if the AZ is lost; only 1 mount target; Max I/O not supported
- EFS can be HA with storage replicated to multiple AZs behind the scenes. To have a fully HA implementation, however => needed to have mount targets in multiple AZs or a single mount target in one subnet will be a single point of failure.
- EFS can be accessed outside of the VPC over hybrid networking: VPN or DX. EFS is a great tool for storage handling across multiple units of compute
- Other VPCs/accounts can mount via VPC peering or Transit Gateway (use mount target IP or efs-utils with cross-VPC DNS)
- EFS is accessible for Lambda functions. Lambda has to be configured to use VPC networking in order to use EFS
- Also integrates with ECS/Fargate tasks and EKS (EFS CSI driver) as persistent shared storage
- Mount targets: provide IP addresses in the range of the VPC. For HA we should provision mount targets in every availability zones present in a VPC
  - One mount target per AZ, protected by **security groups** (allow NFS TCP 2049)
- EFS offers 2 performance modes:
  - General Purpose: ideal for latency sensitive use cases (it is the default and **recommended for all file systems**)
  - Max I/O: can be used to scale to higher levels of aggregate throughput. Has higher latencies
    - Previous generation mode; not supported with Elastic throughput or One Zone. General Purpose + Elastic now scales higher (up to 250k read IOPS / 50k write IOPS by default, increase possible)
- EFS offers 3 different throughput modes:
  - Bursting: works similar to EBS GP2 storage, throughput scales with storage size (baseline 50 KiB/s per GiB + burst credits)
  - Enhanced:
    - Elastic (default and recommended): for workloads with unpredictable I/O. Performance automatically scales with activity, pay for throughput you use. Reads up to tens of GiB/s, writes several GiB/s per file system (region dependent)
    - Provisioned: we can specify throughput requirements independent of the size. Good for predictable, steady high throughput with small data size
  - Throughput mode can be changed (Bursting/Provisioned changes limited to once per 24h)
  - Per-client throughput up to 1,500 MiB/s with `amazon-efs-utils` 2.0+ (500 MiB/s otherwise)
- EFS offers 3 storage classes:
  - Standard: for data that is accessed and modified frequently (SSD, sub-ms latency)
  - Infrequent Access (IA): cost-optimized storage class for data that is less frequently accessed (few times a quarter)
  - Archive: for data that is accessed a few times a year (up to ~50% cheaper than IA)
  - IA and Archive: per-GB access (retrieval) charges, 128 KiB minimum billable file size; Archive has a 90-day minimum storage duration
  - Archive requires Elastic throughput
  - One Zone file systems have One Zone Standard and One Zone IA
- We can automatically move data between these classes using lifecycle policies
  - Default policy: Standard => IA after 30 days without access, IA => Archive after 90 days
  - Option to move back to Standard on first access
  - Small files (< 128 KiB) and metadata always stay in Standard
- EFS can have File system policies (JSON) to restrict permissions to interact with the EFS storage.
  - Examples: enforce encryption in transit (`aws:SecureTransport`), deny root access, enforce read-only, require access points

![EFS](images/EFS.png)

## EFS Security

- Access control layers: security groups on mount targets (network), IAM + file system policy (API/NFS client authorization), POSIX permissions (users/groups)
- **EFS Access Points**: application-specific entry points that enforce a POSIX user/group and a root directory => multi-tenant / per-application isolation (common with Lambda, ECS, EKS)
- Encryption at rest with KMS: must be chosen **at creation** (can't be enabled later => create new file system and copy with DataSync)
- Encryption in transit: TLS via the EFS mount helper (`mount -t efs -o tls`)

## EFS Replication and Backup

- **EFS Replication**: asynchronous replication to a destination file system in the same or another region, also **cross-account**
  - RPO of 15 minutes for most file systems (after initial sync)
  - Destination is read-only until replication is deleted (failover); failback by reversing replication direction (incremental)
  - Exam: "cross-region DR for shared Linux file system" => EFS Replication
- **AWS Backup**: automatic daily backups on by default for new file systems (console), incremental, cross-region/cross-account copies, Vault Lock
- DataSync: migration from on-premises NFS to EFS, or between EFS file systems/regions

## EFS vs Others

- EFS vs EBS: EFS is shared, multi-AZ, elastic, NFS; EBS is single-AZ block storage, attached to one instance (except io1/io2 Multi-Attach)
- EFS vs FSx for Lustre: Lustre for HPC/ML with hundreds of GB/s and S3 integration; EFS for general purpose shared Linux storage (home dirs, CMS, web serving, containers)
- EFS vs FSx for OpenZFS / ONTAP: need snapshots/clones, NFSv3, ZFS features or multi-protocol (SMB + NFS) => FSx
- EFS vs S3: EFS for POSIX file semantics; S3 for object storage at lower cost
