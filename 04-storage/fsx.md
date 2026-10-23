# FSx

## FSx For Windows File Servers

- FSx for Windows are fully managed native Windows file servers/file shares
- Designed for integration with Windows environments
- Integrates with AWS managed Directory Service or Self-Managed AD
- Resilient and highly available service. Can be deployed in single or multi-AZ within a VPC. Even in a single-AZ deployment the backend of the service uses replications to ensure that is resilient to hardware failure
  - Multi-AZ: active/standby file servers in 2 AZs, synchronous replication, automatic failover (DNS name stays the same)
- It can perform a full range of different kind of backups, including client-side and AWS-side features. On the AWS-side, it can perform automatic and on-demand backups (stored in S3, also via AWS Backup, cross-region copies)
- FSx can be accessed over VPC peering, VPN and DX (and Transit Gateway)
- FSx supports de-duplication, scaling through Distributed File System (DFS), KMS at rest encryption and enforced encryption in transit
- Allows for volumes shadow copies, we can initiate previous versions for a file
- It is highly performant: 32 MB/s up to 12 GB/s throughput (12 GB/s and 400k IOPS in major regions, 2 GB/s / 80k IOPS elsewhere), 100k's IOPS, <1ms latency
- Storage types: SSD (latency sensitive: databases, home dirs) or HDD (cheaper: general file shares); storage and throughput can be scaled up after creation
- Features:
    - VSS - user-driven restores (view previous versions)
    - Native file system accessible over SMB (2.0 - 3.1.1), SMB Multichannel
    - Uses Windows permission model (NTFS ACLs)
    - Supports DFS - scale-out file share structure, group file shares together in one enterprise-wise structure
    - Managed - no file server admin
    - Integrates with Amazon managed DS or our own directory
    - Continuously available (CA) shares for SQL Server and SMB transparent failover
    - User storage quotas, file access auditing (to CloudWatch Logs / Kinesis Data Firehose)
    - Can be mounted from Linux clients (SMB/CIFS) too
- Use cases: Windows home directories, line-of-business apps, SQL Server on EC2 (shared storage for FCI), Amazon WorkSpaces / AppStream user profiles
- Exam: "Windows SMB file share + AD integration + ACLs" => FSx for Windows (not EFS, EFS is Linux NFS only)

## FSx for Lustre

- File system designed for high performance workloads
- Is a managed implementation of the Lustre file system, designed for HPC - Linux clients (POSIX file system)
- Lustre is designed for machine learning, big data, financial modelling, video rendering, EDA, genomics
- Can scale to 100's GB/s (up to TB/s) throughput, millions of IOPS and offers sub millisecond latency
- Supports **EFA** and **GPUDirect Storage** for GPU-based ML training (high throughput per client)
- Can be provisioned using 2 different deployment types:
    - Persistent: provides HA in one AZ only, provides self-healing, recommended for long term data storage
    - Scratch: highly optimized for short term solutions, no replication is provided
- Only single-AZ deployments (no Multi-AZ) => for DR rely on backups / S3 data repository
- FSx is available over VPN or DX for on-premises
- S3 repository: files are stored in S3 and they are lazily loaded into FSx for Lustre file system at first usage
    - Data Repository Associations (DRA): link file system directories to S3 prefixes; **automatic import** (S3 changes => file system) and **automatic export** (file system changes => S3) for Persistent 2
    - Data repository tasks for bulk import/export
- Sync changes between the file system and S3: `hsm_archive` command (or export data repository tasks). Without automatic export configured, the file system and the S3 bucket are not automatically in sync
- Lustre file system:
    - MDT - Metadata stored on Metadata Targets
    - OST - Objects are stored on object storage targets, each can be up to 1.17 TiB
- Baseline performance of the file system is based on the size:
    - Size: min 1.2 TiB  and then we can use increments of 2.4 TiB
    - Scratch: base 200 MB/s per TiB of storage
    - Persistent 1 (SSD) offers: 50 MB/s, 100 MB/s and 200 MB/s per TiB storage
    - Persistent 2 (SSD) offers: 125 MB/s, 250 MB/s, 500 MB/s and 1000 MB/s per TiB storage
    - Persistent HDD: 12 or 40 MB/s per TiB (optional SSD read cache)
    - For both types we can burst up to 1300 MB/s per TiB using credits
- **Intelligent-Tiering storage class** (May 2025): fully elastic Lustre storage (no provisioning of capacity), automatic tiering between Frequent Access, Infrequent Access and Archive tiers, optional SSD read cache. Lowest cost Lustre option for mixed hot/cold data
- FSx for Luster deployment types:
    - Scratch:
        - Is designed for pure performance, for short term and temporary workloads
        - Does not provide any type of HA or replication, in case of a HW failure all the data stored on that hardware is lost
        - Larger file systems mean more servers, more disks => more chance of failure
    - Persistent:
        - Has replication within one AZ only
        - Auto-heals when hardware failure occurs
    - Backups to S3 (manual or automatic with a retention of 0-35 days) are supported on **Persistent** file systems only, not on Scratch (for Scratch, export data to S3)
- Exam: "HPC / ML training reading datasets from S3 with very high throughput" => FSx for Lustre linked to S3

## FSx for NetApp ONTAP

- Fully managed storage built on NetAPP ONTAP
- Provides reach set of features available with NetApp's data management software:
    - Storage efficiencies: compression, deduplication, compaction, thin provisioning
    - Low-cost, fully elastic capacity pool tiering
    - Data protection: snapshots, SnapVault and native Amazon FSx backups
    - Disaster recovery using SnapMirror and Amazon FSx Backups
    - Caching: FlexCache, Global File Cache
    - Multiprotocol access from Linux, Windows, macOS: **NFS (v3, v4.x), SMB and iSCSI / NVMe over TCP** (block storage)
    - Instant writable clones (FlexClone) - dev/test copies without extra storage
    - Other features: antivirus scanning, SnapLock (WORM), Autonomous Ransomware Protection
- Deployment: Single-AZ or Multi-AZ (HA pair across 2 AZs, automatic failover); scale-up (1 HA pair) or scale-out (up to 12 HA pairs, up to 72 GB/s read throughput)
- Intelligent policy-based data movement between tier:
    - Amazon FSx for NetApp ONTAP file system has two storage tiers: primary storage and capacity pool storage
        - Primary storage is provisioned, scalable, high-performance SSD storage (up to 192 TiB on first-gen; second-gen up to 512 TiB per HA pair, up to 1 PiB) that's purpose-built for the active portion of our data set
        - Capacity pool storage is a fully elastic storage tier that can scale to petabytes in size and is cost-optimized for infrequently accessed data
    - Enabling tiering allows intelligent data movement between tiers based on the access (tiering policies: Snapshot-only, Auto, All, None)
- Managed via AWS console/API or NetApp tools (ONTAP CLI/REST API, BlueXP)
- S3 access points can be attached to ONTAP volumes => access file data via S3 APIs (e.g. for Bedrock, Athena, analytics) without copying
- Use case for NetApp ONTAP:
    - Backup and archive using SnapVaults
    - Cross region DR copy of FSx file data using SnapMirror
    - FlexCache for caching data and bringing data closer between regions and on-prem access
    - Can be used with Amazon WorkSpaces to provide shared network-attached storage (NAS) or to store roaming profiles for Amazon WorkSpaces accounts
    - Lift-and-shift of on-prem NetApp NAS (SnapMirror from on-prem ONTAP), VMware Cloud on AWS / Amazon EVS datastores
- Exam: "multi-protocol (NFS + SMB) or iSCSI, existing NetApp, SnapMirror" => FSx for NetApp ONTAP

## FSx for OpenZFS

- Fully managed file storage service built on the open-source OpenZFS file system
- Can be accessed over the industry-standard Network File System (NFS) protocol (NFS v3, v4.0, v4.1, v4.2)
- Powered by AWS Graviton processors, along with the latest AWS disk and networking technologies
- Performance: up to ~10 GB/s throughput, up to 1 million IOPS, sub-ms latency (higher with in-memory/NVMe cache)
- Deployment: Single-AZ (non-HA), Single-AZ (HA) and Multi-AZ (HA)
- Features: point-in-time snapshots, instant clones, compression (LZ4/ZSTD), user/group quotas, cross-region volume backups
- **Intelligent-Tiering storage class**: fully elastic storage with automatic tiering (Frequent/Infrequent/Archive), optional SSD read cache (up to 512 TiB per file system)
- S3 access points can be attached to OpenZFS volumes (HA file systems) => S3 API access to file data
- Use cases:
    - Migration of on-premises data stored in ZFS or other Linux-based file servers to AWS
    - Wide range of Linux, Windows, and macOS workloads, including big data and analytics, code and artifact repositories, DevOps solutions, web content management, front-end electronic design automation (EDA), genomics research, and media processing
- Data security:
    - Encryption of data at rest is automatically enabled when we create an Amazon FSx for OpenZFS file system through the AWS
    - Amazon FSx for OpenZFS uses industry-standard AES-256 encryption algorithm to encrypt file system data at rest
    - Amazon FSx for OpenZFS file systems automatically encrypt data in transit when they are accessed from Amazon EC2 instances that support encryption in transit
- Exam: "lift-and-shift ZFS/Linux NFS with snapshots and clones, low latency" => FSx for OpenZFS

## Amazon File Cache

- Managed high-speed cache (Lustre-based) in front of on-premises NFS file systems, FSx file systems or S3 buckets
- Lazy loads data on first access, presents multiple data repositories under a unified namespace
- Use cases: burst HPC/rendering to AWS while data stays on-premises
- Exam alternative for the same pattern: FSx for ONTAP FlexCache, Storage Gateway, FSx for Lustre + S3

## Choosing an FSx / File Service

| Requirement                                   | Service                     |
| --------------------------------------------- | --------------------------- |
| Linux shared NFS, serverless, elastic         | EFS                         |
| Windows SMB, AD, NTFS ACLs, DFS               | FSx for Windows             |
| HPC/ML, hundreds of GB/s, S3 integration      | FSx for Lustre              |
| Multi-protocol NFS/SMB/iSCSI, NetApp features | FSx for NetApp ONTAP        |
| ZFS migration, NFS with snapshots/clones      | FSx for OpenZFS             |
| Multi-AZ                                      | EFS, FSx Windows, ONTAP, OpenZFS (not Lustre) |

- All FSx: deployed in a VPC (ENIs), accessed over DX/VPN, encryption at rest via KMS, backups to S3 (AWS Backup integration)
- Migration into FSx: AWS DataSync (all FSx types), Robocopy (Windows), SnapMirror (ONTAP)
