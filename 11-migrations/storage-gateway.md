# Storage Gateway

- Normally runs as a VM on-premises (VMware ESXi, Hyper-V, KVM), as an EC2 instance, or as a hardware appliance
  > **Status (2026):** the Storage Gateway **Hardware Appliance** is no longer offered since May 12, 2025 (existing appliances supported until May 2028)
- Acts as bridge between storage that exists on-premises and AWS
- Presents storage using iSCSI, NFS or SMB
- On AWS integrates with EBS, S3 and Glacier
- Storage gateways is used for migrations, extensions, storage tiering, DR and replacement of backup systems
- 4 options:
  - Volume Gateway
  - Tape Gateway
  - S3 File Gateway
  - FSx File Gateway
    > **Status (2026):** Amazon FSx File Gateway is no longer available to new customers since Oct 28, 2024. Alternatives: access FSx for Windows File Server directly over DX/VPN, use FSx for NetApp ONTAP (FlexCache/SnapMirror) or S3 File Gateway
- Volume Gateway, Tape Gateway and S3 File Gateway remain available
- Gateway connects to AWS over the internet, DX or a **VPC endpoint (PrivateLink)**; data encrypted in transit (TLS) and at rest (SSE-S3/SSE-KMS)
- Local disks: **cache** (frequently accessed data) + **upload buffer** (staging for async upload)

## Volume Gateway

- Offers 2 different types of operation:
  - Volume Stored Mode:
    - The virtual appliance presents volumes over iSCSI to servers running on-premises (similar to what NAS/SAN hardware would)
    - Servers can create files systems on top of these volumes and use it in a normal way
    - These volumes consume capacity on-premises
    - Storage gateway has local storage, used as primary storage, everything is stored locally
    - Upload buffer: any data written to the local storage is also copied in the upload buffer and it will be uploaded to the cloud asynchronously via the storage gateway endpoint
    - The upload data is copied into S3 as EBS snapshots which can be converted into EBS volumes
    - It is great to do full disk backups, offering excellent RTO and RPO values
    - Volume Stored Mode does not allow extending the data center capacity! The full copy of the data is stored locally
    - Limits: up to 32 volumes per gateway, 16 TiB each => 512 TiB per gateway
      ![Volume Stored Mode architecture](images/StorageGatewayVolumeStored.png)
  - Volume Cached Mode:
    - Volume Cached Mode shares the same basic architecture with Stored Mode
    - The main location of data is no longer on-premises, it is on AWS S3
    - It has a local cache for the data only storing the frequently accessed data, the primary data will be in S3
    - The data will be stored in AWS managed area of S3, meaning it wont be visible using the AWS console. It can be viewed from the storage gateway console
    - The data is stored in raw block state
    - We can create EBS volumes out of the data
    - Volume Cached Mode allows for an architecture know as data center extension
    - Limits: up to 32 volumes per gateway, 32 TiB each => 1 PiB per gateway
    - Exam: stored = low-latency access to the whole dataset locally + cloud backup; cached = only hot data local, extend capacity
- Snapshots of volumes can be scheduled and managed with **AWS Backup**; restore as EBS volume in AWS (DR to EC2) or as a new gateway volume
      ![Volume Cached Mode architecture](images/StorageGatewayVolumeCached.png)

## Tape - VTL Mode

- VTL - Virtual Tape Library
- Examples of tape backups: LTO-9 (Linear Tape Open) Media which can hold 24TB raw data per tape
- A Library is 1 or more drives, 1 or more loaders and slots
- Tape Loader (Robot): robot arm can insert/remove/swap tapes
- 1 Tape Drive can use 1 tape at a time (all read or all write)
- Traditional tape backup architecture:
  ![Traditional tape backup architecture](images/TraditionalTapeBackup.png)
- Storage Gateway Tape (VTL) Mode architecture:
  ![Storage Gateway Tape (VTL) Mode architecture](images/StorageGatewayVTL.png)
- A Virtual tape can be from 100 GiB to 15 TiB
- A Storage Gateway can handle at max 1 PiB of data across 1500 virtual tapes (archive: unlimited)
- When virtual tapes are not used, they can be exported in the backup software marking them as not being in the library (equivalent of ejecting them and moving them to the offsite storage) => unlimited Archive storage
- When exported, the virtual tape is archived in the Virtual Shelf which is backed by Glacier (S3 Glacier Flexible Retrieval or S3 Glacier Deep Archive)
- Retrieval from archive back to the library: ~3-5 hours (Flexible Retrieval) or ~12 hours (Deep Archive)
- **Tape retention lock** (governance/compliance) and **write-once-read-many (WORM)** tape pools for compliance
- Works with existing backup software (Veeam, Veritas NetBackup/Backup Exec, Commvault, etc.) => no change to backup processes
- Storage Gateway in VTL Mode pretends to be a iSCSI tape library, tape change and iSCSI drive (physical tape backup system)
- Use cases:
  - On-premises data storage extension into AWS
  - Migration of historical sets of tape backups

## File Mode

- Storage Gateway manages files in File Mode
- File Gateway bridges on-premises file storage and S3
- With File Gateway we create one or more mount points (shares) available via NFS or SMB
- File Gateways maps directly onto on S3 bucket above which we have visibility from the AWS console
- File Mode uses Read an Write Caching ensuring LAN-like performance
- File Gateway architecture:
  ![File Gateway architecture](images/StorageGatewayFile.png)
- For Windows environments we can use AD authentication to access the File Gateway
- Each file share maps to one S3 bucket (optionally a prefix); one gateway can serve multiple shares/buckets
- Objects are written to S3 using the configured storage class (Standard, Standard-IA, One Zone-IA, Intelligent-Tiering); Glacier classes are not directly readable through the share
- File Mode can be used for multiple contributors (multiple shares on-premises)
- File paths in a File Gateway map directly to S3 object names
- `NotifyWhenUploaded`: API to get notified (via EventBridge) when a file has been fully uploaded to S3 - useful to trigger downstream processing
- `RefreshCache`: API to refresh the gateway's listing/metadata when objects are changed in S3 by something else (other gateways or apps). Other gateways do not update by default. Can be automated (automated cache refresh with TTL)
- File Gateway does not support any kind of object locking => one gateway can override files from another gateway. We should use a read only mode on other shares or tightly control file access
  ![GatewayMulti](images/GatewayMulti.png)
- The bucket backing the File Gateway can be used with cross-region replication (CRR)
- The lifecycle policies can also be used for files to be moved automatically between classes

  ![FileGateway](images/FileGateway.png)
- Objects written to the bucket directly (not via gateway) are visible on the share only after a cache refresh
- Audit logs for SMB shares to CloudWatch Logs

## When to use what (exam)

- Replace tape backup infrastructure, keep backup software => **Tape Gateway**
- Block storage (iSCSI) with cloud backups as EBS snapshots, DR to EC2 => **Volume Gateway** (stored or cached)
- On-prem apps need NFS/SMB access to objects in S3, data lake ingestion, backups to S3 with lifecycle => **S3 File Gateway**
- One-time bulk migration of files => **DataSync** (then optionally File Gateway for ongoing access)
- Low-latency Windows file shares in AWS for on-prem users => FSx for Windows File Server directly (FSx File Gateway closed to new customers)
- Gateway for branch offices with limited bandwidth => local cache sizing + bandwidth throttling/schedule on the gateway
