# Snow Family (closed to new customers)

> **Status (2026):** AWS Snowball Edge is no longer available to new customers since **Nov 7, 2025** - no Snow Family device can be ordered by new customers. Existing customers can continue to use it (security/availability updates continue)
> - Snowmobile: retired (~March/April 2024)
> - Snowcone: discontinued Nov 12, 2024 (existing customers supported until Nov 12, 2025)
> - Older Snowball Edge models (e.g. 80 TB Storage Optimized, older Compute Optimized) discontinued Nov 2024
> - Replacements: **AWS DataSync** (online transfers, optionally over a temporary Direct Connect hosted connection), **AWS Data Transfer Terminal** (secure physical location to upload from your own devices), AWS Partner offline transfer solutions (Marketplace), **AWS Outposts** for edge compute
> - Exam note: SAP-C02 questions written before 2025 may still expect "Snowball Edge" for offline multi-TB/PB transfers with poor connectivity

**Replaced by AWS DataSync for online transfers, AWS Data Transfer Terminal for secure physical transfers**.

- Snowball series are designed to move large amount of data in or out of AWS
- The products in Snow series are physical storage units: suitcases and trucks
- We can order them empty, load them up and return them or vice-versa

## Snowball

> **Status:** legacy - original Snowball (storage only) devices replaced by Snowball Edge years ago

- It is a device which is ordered from AWS, log a job and device will be delivered to us
- Any data stored in Snowball is encrypted using KMS
- There are 2 types of devices with 50 TB or 80TB capacity
- In terms of network connectivity we can have 1 Gbps (RJ45 1GBase-TX) or 10 Gbps (LR/SR) networking
- Economical range for a Snowball is 10 TB to 10 PB range of data (multiple devices can be used)
- Multiple devices can be ordered and be sent to multiple business premises
- Snowball only includes storage capability

## Snowball Edge

- Includes both storage capability and compute capability
- It has a larger capacity compared to classic Snowball and has faster networking connection
- There are 3 different type of Snowball Edge (legacy specs, discontinued Nov 2024):
  - Storage optimized (with EC2 capability): 80 TB, 24 vCPU, 32 Gib RAM, 1 TB of local SSD for EC2 usage
  - Compute optimized: 100 TB + 7.68 NVME, 52 vCPU, 208 Gib RAM
  - Compute optimized with GPU: 100 TB + 7.68 NVME, 52 vCPU, 208 Gib RAM, GPU
- Current generation (existing customers only):
  - Storage Optimized 210 TB (NVMe), available for data transfer jobs
  - Compute Optimized: up to 104 vCPU, 416 GB memory, 28 TB NVMe (AMD EPYC)
- Ideal for remote sites or where data processing on ingestion is needed
- Data is imported into **S3** (S3 Standard); to land in Glacier, use an S3 Lifecycle policy after import (exam trap: Snowball cannot import directly into Glacier)
- S3 adapter / S3-compatible storage on device, NFS mount, EC2-compatible instances, EKS Anywhere; managed with **AWS OpsHub** GUI or Snowball Edge client
- Encryption with KMS keys (keys never stored on device), tamper-evident, TPM, E Ink shipping label
- Clusters of devices for durability/capacity at the edge (compute jobs)

## Snowmobile

> **Status (2026):** retired in 2024. Use multiple parallel DataSync tasks + Direct Connect, or Data Transfer Terminal / partners

- Portable data center within a shipping container on a truck
- Needs to be specially ordered from AWS
- Ideal for single location when 10 PB+ is required
- Can store up to 100PB of data per Snowmobile
- Not economical for multi-site or sub 10PB

## Snowcone

> **Status (2026):** discontinued Nov 12, 2024; support ended Nov 12, 2025

- Smallest device: 8 TB HDD / 14 TB SSD, 2 vCPU, 4 GB RAM, could run DataSync agent, battery-powered option
- Used for edge locations with limited space, IoT, rugged environments

## AWS Data Transfer Terminal

- Launched Dec 2024: physical AWS facilities (initially Los Angeles and New York, more planned) where you bring your own storage devices
- Reserve a time slot in the console, connect devices to a high-throughput network link, upload directly to AWS public endpoints (S3, etc.)
- Use when data is already on portable media (media & entertainment, autonomous vehicle data, research) and online transfer is too slow
- Billed hourly per reserved time slot; no per-GB charge for uploads to regions on the same continent (no device shipping)

## Choosing an Offline vs Online Transfer (2026)

- Time to transfer online = data size / effective bandwidth (e.g. 100 TB over 1 Gbps ~= 10 days at 100% utilization)
- Enough bandwidth / can add DX => **DataSync** (scheduled, incremental, validated)
- Data already on portable drives or need faster ingest than WAN => **Data Transfer Terminal**
- Existing Snow customers with no connectivity at the site => Snowball Edge still valid
- Edge compute in disconnected/rugged sites => **Outposts servers/racks** (new customers), Snowball Edge Compute Optimized (existing)
- Ongoing hybrid access after migration => Storage Gateway / DataSync scheduled tasks
