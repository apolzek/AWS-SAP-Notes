# EBS and Instance Store

## EBS Volume Types

**General Purpose SSD (GP2/GP3)**

GP2:

- GP2 is the previous default storage type for EC2, GP3 is the newer version (now the console default and recommended choice; migrate gp2 => gp3 live with Elastic Volumes)
- Volumes can range from 1GB - 16TB
- IO Credit:
  - An IO is one input/output operation, one 16 KB chunk of data
  - 1 IOPS is 1 IO in 1 second
  - 1 IO credit = 1 IOPS
- If we have no credits for the volume, we can not perform any IO
- The IO credit bucket has 5.4 million of credits, it refills based at rate based on the baseline performance of the storage
- The baseline performance for GP2 is based on the volume size,
- Bucket filling rate = maximum of (100 IO credit per sec) OR (3 IO credits per second per GB of volume size). Max is 16'000 IO credit per second.
- By default GP2 can burst up to 3000 IOPS (only volumes < 1 TiB burst; at ≥ 1 TiB baseline is already ≥ 3000)
- Max throughput 250 MiB/s
- All volumes get an initial 5.4mio IO credits = 30min at 3000 IOPS

![GP2](images/GP2.png)

- If we consume more credits than the bucket is refilling => bucket is getting emptier
- We have to ensure the buckets are replenishing and not depleting down to 0, otherwise the storage will be unusable
- Max IO 16000 IO credits per second, any volume larger than 5.33 TB will achieve this maximum rate constantly
- GP2 can be used for boot volumes

GP3:

- GP3 is similar to GP2, but it removes the credit system for a simpler way of working:
  - Standard: 3000 IOPS and 125 MiB/s regardless of volume size
  - Extra cost for up to **80'000 IOPS** and **2000 MiB/s** throughput (8x the max throughput of GP2) - limits raised in Sept 2025 (previously 16'000 IOPS / 1000 MiB/s / 16 TiB)
  - Ratios: max 500 IOPS per GiB, max 0.25 MiB/s per provisioned IOPS => full 80k IOPS needs ≥ 160 GiB
  - IOPS and throughput are configured independently of size (unlike gp2)
- Volumes can range from 1GB - 64TB
- Base price for GP3 is 20% cheaper than GP2

![GP3](images/GP3.png)

**Provisioned IOPS SSD (IO1/2)**:

- There are 2 types of provisioned IOPS storage options: IO1 and its successor IO2 Block Express (all io2 volumes are now Block Express, GA - no longer preview)
- For this storage category the IOPS value can be configured independently of the storage size
- Provisioned IOPS storages are recommended for usage where consistent low latency and high throughput is required
- Max IOPS per volume is 64_000 IOPS per volume and 1000 MB/s throughput (io1), while with BlockExpress we can achieve 256_000 IOPS per volume and 4000 MB/s throughput
- Volume size ranges from 4 GB up to 16 TB for IO1 and up to 64 TB for io2 BlockExpress
- Durability: io2 99.999% (vs 99.8-99.9% for io1, gp2, gp3, st1, sc1), sub-millisecond average latency
- Only io1/io2 support **Multi-Attach**
- We can allocate IOPS performance values independently of the size of the volume, there is a maximum IOPS value per size:
  - IO1 50 IOPS / GB MAX
  - IO2 500 IOPS / GB MAX
  - BlockExpress 1000 IOPS / GB MAX
- Per instance performance: maximum performance between EBS service and EC2. Usually this implies more than one volume in order to be saturated. Historic max values (newest EBS-optimized instance families, e.g. the `*b` variants, go higher - always check the instance type specs):
  - IO1 260_000 IOPS and 7500 MB/s (4 volumes to saturate)
  - IO2 160_000 IOPS and 4750 MB/s (less than IO1)
  - BlockExpress 260_000 IOPS and 7500 MB/s
- Use cases: smaller volumes and super high performance

**GP3 vs IO1/2**

Both can be used to specify IOPS **BUT**:

- io1/2: physical slice of an SSD cluster capable of the requested IOPS
  - ultra-low latency, guaranteed, and not variable.
  - more expensive
  - higher limits than gp3
- gp3: limits are enforced in software over a shared SSD pool
  - IOPS still guaranteed, but if other users on the shared pool have a heavy use => latency can increase

![IO1/2](images/IO12.png)

**HDD-based volume types**

- There are 2 types of HDD based storages: ST1 Throughput Optimized, SC1 Cold HDD
- **ST1**:
  - Cheaper than SSD based volumes, ideal for larger volumes of data
  - Recommended for sequential data, applications when throughput is more important than IOPS
  - Volume size can be between 125 GB and 16 TB
  - Max 500 IOPS **BUT** data IO is measured in blocks of 1 MB => max throughput of 500 MB/s
  - Works similar as GP2 with a credit system
  - Offer a base performance of 40 MB/s per TB of volume size with bursting to 250 MB/s per TB
  - Designed for frequently accessed sequential data at lower cost
- **SC1**:
  - SC1 is cheaper than ST1, has significant trade-offs
  - Geared towards maximum economy when we want to share a lot of data without caring about performance
  - Offers a maximum of 250 IOPS, 250 MB/S throughput
  - Offer a base performance of 12 MB/s per TB of volume size with bursting to 80 MB/s per TB
  - Volume size can be between 125 GB and 16 TB
  - It is the cheapest EBS storage available

## Instance Store Volumes

- Provides block storage devices, raw volumes which can be mounted to a system
- They are similar to EBS, but they are local drives instead of being presented over the network
- These volumes are physically connected to the EC2 host, instances on the host can access these volumes
- Provides the **highest storage performance** in AWS
- Instance stores are included in the price of EC2 instances with which they come with
- **Instance stores have to be attached at launch time, they can not be added afterwards!**
- If an EC2 instance moves between hosts the instance store volume loses all its data
- Instances can move between hosts for many reasons: instance are stopped and restarted, maintenance reasons, hardware failure, etc.
- Data **survives a reboot**, but is lost on stop/hibernate/terminate, host failure or resize
- Instance store can't be snapshotted with EBS snapshots => protect via app replication or copy to EBS/S3
- **Instance store volumes are ephemeral volumes!**
- One of the primary benefit of instance stores is performance.
- Varies depending on instance type:
  - D3: 4.6 GB/s of throughput
  - I3: 16 GB/s of throughput with NVMe SSD
- Instance store considerations:

  - Instance store volumes are local to EC2 hosts
  - Instance store can be added only at launch
  - Data is lost on an instance store volume in case the instance is moved, resized or when there is hardware failure
  - Instance stores provide high performance
  - For instance store volumes we pay for it with the EC2 instance
  - Instance store volumes are temporary!

  ![Instance store volume](images/InstanceStore.png)

## Choosing between Instance Store and EBS

- For persistence storage we should default to EBS
- For resilient storage we should avoid instance store and default to EBS
- EBS is replicated behind the scenes within the same AZ => resilient
- If the storage should be isolated from EC2 instance lifecycle we should use EBS
- Resilience when done via app in-built replication (with multiple instances) - we can use both, it depends on the situation
- For high performance needs - we can also use both, it depends on the situation
- For super high performance we should use instance store
- If cost is a primary concern we can use instance store if it comes with the EC2 instance
- Cost consideration: cheaper volumes: ST1 or SC1
- Throughput or streaming: ST1
- Boot volumes: HDD based volumes are not supported (no ST1 or SC1)
- GP2 - max performance up to 16000 IOPS, GP3 - up to 80000 IOPS
- IO1/2 - up to 64000 IOPS (BlockExpress: 256000)
- RAID0 + EBS: up to 260000 IOPS (maximum possible IOPS per EC2 instance, with large instance and multiple EBS volumes to consume the whole capacity)
- For more than 260000 IOPS => use instance store

## EBS Basics

- EBS volumes are **AZ-scoped**: a volume can only be attached to instances in the same AZ. To move to another AZ/region => snapshot + restore/copy
- Replicated within the AZ only => for AZ/region resilience use snapshots (S3-backed, regional)
- **Elastic Volumes**: change size, type, IOPS, throughput live without detaching (size can only grow; wait ~6h between modifications of the same volume). After growing, extend the file system in the OS
- Delete on termination: default true for root volume, false for additional volumes
- Billing: provisioned GB-month (not used), plus provisioned IOPS/throughput (gp3 above baseline, io1/io2)

## EBS Multi-Attach

- io1/io2 only: attach one volume to up to **16 Nitro instances in the same AZ**
- Each instance gets full read/write; the application/file system must be cluster-aware (e.g. GFS2, OCFS2, Windows Server Failover Clustering with io2) - standard XFS/EXT4 will corrupt data
- Exam: "shared block storage for clustered app in one AZ" => io2 Multi-Attach; shared files across AZs => EFS/FSx

## EBS Encryption

- Uses KMS keys (AWS managed `aws/ebs` or customer managed), AES-256; encrypts data at rest, in transit between instance and volume, snapshots and volumes created from them
- Minimal performance impact, handled by the Nitro host
- **Encryption by default**: per-region account setting => all new volumes/snapshot copies encrypted
- Can't directly decrypt an encrypted volume or encrypt an unencrypted one in place => snapshot, copy the snapshot with encryption enabled, create a new volume
- Sharing encrypted snapshots cross-account requires a **customer managed key** shared with the target account (can't share snapshots encrypted with `aws/ebs`)

## EBS Snapshots

- Point-in-time, **incremental** backups stored in S3 (managed by AWS, not visible in our buckets); deleting a snapshot only removes data not needed by other snapshots
- Regional: can restore in any AZ of the region; copy snapshots across regions/accounts for DR
- First snapshot is a full copy; volumes restored from snapshots are lazily loaded from S3 => first-read latency
  - Fix with **Fast Snapshot Restore (FSR)**: fully initialized volumes immediately, enabled per snapshot per AZ (billed per hour, credit-based volume creation)
  - Or **Provisioned rate for volume initialization**: pay to hydrate a volume from a snapshot at a specified rate (predictable restore time)
  - Or read all blocks (`fio`/`dd`) to pre-warm
- **EBS Snapshots Archive**: up to 75% cheaper, for snapshots retained ≥ 90 days (minimum 90 days billing), restore takes 24-72 hours; archived snapshots are full (not incremental)
- **Recycle Bin**: retention rules to recover accidentally deleted EBS snapshots and EBS-backed AMIs (1 day - 1 year); rule lock to prevent rule changes
- **Time-based snapshot copy**: specify a completion duration (15 min - 48 h) for cross-region/account copies => predictable RPO for DR
- **EBS direct APIs**: read/write snapshot blocks directly, list changed blocks between snapshots (used by backup vendors, on-prem => EBS imports)
- Automation: **Amazon Data Lifecycle Manager (DLM)** (snapshot/AMI policies, cross-region copy, archive) or **AWS Backup** (centralized, cross-account, Vault Lock)
- Multi-volume crash-consistent snapshots: snapshot all volumes of an instance at the same point in time
- Exam: "restore volumes from snapshots with full performance immediately" => Fast Snapshot Restore

## EBS Exam Tips

- Default/general purpose (boot, dev/test, most DBs) => gp3; cheaper and more flexible than gp2
- Sub-ms latency, > 80k IOPS per volume, 99.999% durability, Multi-Attach => io2 Block Express
- Big sequential throughput (log processing, data warehouse, Kafka) => st1; cold infrequent data at lowest cost => sc1
- Temporary scratch/cache/buffer, highest IOPS, data replicated by the app => instance store
- Monitoring: CloudWatch `VolumeQueueLength`, `BurstBalance` (gp2/st1/sc1), `VolumeStalledIOCheck`; EBS-optimized instances give dedicated EBS bandwidth
