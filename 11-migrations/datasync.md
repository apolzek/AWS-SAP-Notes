# AWS DataSync

- It is a data transfer service which allows data to be transfered into or out of AWS
- Can be used for workflows such as migrations, data processing transfers, archival, cost effective storage, DR/BC
- Each task can use up to 10 Gbps with an agent (5 Gbps for agentless transfers). Basic mode tasks: up to 50 million files/objects per execution (on-prem/other cloud <=> AWS) or 25 million (AWS <=> AWS); Enhanced mode: virtually unlimited
- It also handles the transfer of metadata (permissions, timestamps)
- It provides built-in data validation
- Recommended AWS service for online bulk transfers now that Snow Family is closed to new customers

## Key Features

- Scalable: 10 Gbps per task (~100 TB of data per day); multiple tasks/agents in parallel for more throughput
- Bandwidth Limiters: used to avoid link saturation
- Incremental and scheduled transfer options
- Compressions and encryption (TLS in transit)
- Automatic recovery from transit errors
- Service integration: S3 (all storage classes, incl. Glacier on write), EFS, FSx (Windows, Lustre, OpenZFS, ONTAP), service-to-service transfer
- Pay as you use service: per GB of data transferred (+ per task execution / per object for Enhanced mode)
- The DataSync agent runs on a virtualization platform such as VMware ESXi, KVM, Microsoft Hyper-V, or as an EC2 instance (AMI)
- Filters (include/exclude), manifests (exact list of files/objects to transfer), task reports
- Verification options: verify only transferred data, verify all data, or none

![DataSync](images/DataSync.png)

## DataSync Components

- Task: a job within DataSync, defines what is being synced, how quickly, from where and to where
- Agent: software used to read or write to on-premises data stores using NFS or SMB (also HDFS and S3-compatible object storage)
- Location: every task has two locations from and to, examples: Network File System (NFS), Server Message Block (SMB), HDFS, self-managed object storage, Azure Blob, Google Cloud Storage, other clouds, Amazon EFS, Amazon FSx and Amazon S3
- Task execution: one run of a task

## Task Modes

- **Basic mode**: original mode, requires an agent for non-AWS storage, file count quotas per execution, sequential prepare => transfer => verify
- **Enhanced mode** (2024-2025): parallel listing/transfer/verification, virtually unlimited number of objects, higher performance, detailed metrics
  - Agentless transfers between S3 and other clouds' object storage (Google Cloud Storage, Azure Blob, etc.) since May 2025
  - Agent-based Enhanced mode for on-prem NFS/SMB (Dec 2025), HDFS, Azure Blob and object storage, and Hyper-V agents (Jul 2026); EFS and FSx for Lustre support (Jul 2026)
- Agent needed when the source/destination is not an AWS storage service reachable directly (on-prem, Basic mode cross-cloud). AWS <=> AWS transfers (S3, EFS, FSx) don't need an agent

## Networking

- Agent connects to the DataSync service endpoint via: public internet, **VPC endpoint (PrivateLink)** over Direct Connect/VPN, or FIPS endpoints
- For large migrations with limited bandwidth: DataSync + **Direct Connect** (temporary hosted connection from a DX partner)
- Run the agent close to the source storage (LAN) - the WAN hop is the optimized part
- For Basic mode cross-cloud transfers, run the agent as a VM in the other cloud to reduce egress cost

## DataSync vs Other Services

- **DataSync**: one-time or scheduled bulk copy/sync of files/objects, migration, replication between storage systems
- **Storage Gateway (S3 File Gateway)**: ongoing hybrid access to cloud storage from on-prem apps via NFS/SMB with local cache. Common pattern: DataSync for initial migration, then File Gateway for ongoing access
- **AWS Transfer Family**: managed SFTP/FTPS/FTP/AS2 endpoints for partners/users pushing files into S3/EFS
- **S3 Replication / S3 Batch Operations**: S3 => S3 copies (same/cross region), Batch Replication for existing objects
- **Data Transfer Terminal / Snowball Edge**: offline/physical transfer when the network is not an option (Snow only for existing customers)
- **rsync/robocopy scripts**: no built-in validation, retries, monitoring or encryption => exam prefers DataSync

## Monitoring and Security

- CloudWatch metrics/logs, EventBridge events for task state changes, task reports to S3
- IAM role for DataSync to access S3; KMS encryption at destination
- **DataSync Discovery** (storage system analysis/recommendations):
  > **Status (2026):** DataSync Discovery reached end of support on May 20, 2025 and is no longer available

## Exam Tips

- "Migrate NFS/SMB data to EFS/FSx/S3 quickly and securely, preserve metadata, validate" => DataSync
- "Copy data between EFS file systems / across regions / accounts" => DataSync (no agent)
- "Move data from Google Cloud Storage / Azure Blob to S3" => DataSync Enhanced mode (agentless)
- "Limit bandwidth during business hours" => DataSync bandwidth throttling + scheduling
- "Private transfer without internet" => DataSync agent + VPC endpoint over DX/VPN
- Calculate: 100 TB over 1 Gbps ~= 9-10 days at full utilization => consider DX or offline options for very large data sets
