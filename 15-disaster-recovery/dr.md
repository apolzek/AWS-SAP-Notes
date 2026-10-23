# DR/BC Architecture

- Effective DR/BC costs money all of the time
- We need some type of extra resources which will increase costs
- Executing disaster recover/business continuity process takes time. How long it takes depends on the type of DR/BC in usage
- DR/BC is trade-off between the time and costs
- **RPO** (Recovery Point Objective): max acceptable data loss, measured in time (how old the last good copy can be) - driven by backup/replication frequency
- **RTO** (Recovery Time Objective): max acceptable time to restore service after the disaster - driven by how much is pre-provisioned and automated <span style="color: red;">EXAM</span>
- DR is about **region-level** (or site-level) events; HA (Multi-AZ) is not DR. Also protect against **data corruption/deletion/ransomware** - replication copies bad data too, so backups with point-in-time recovery are still needed

## Types of Disaster Recovery

- **Backup and Restore**:
    - Data is constantly backup up at the primary site
    - The only costs are backup media and management, no ongoing space infrastructure costs
    - Has little or no upfront costs, but implies a significant time for recovery
    - Expected recovery time is counted in hours
    - RPO = time since last backup (hours); RTO = hours (restore data + redeploy infra with IaC)
    - AWS: AWS Backup cross-region/cross-account copies, EBS snapshots/AMI copy, S3 CRR, RDS automated backups replication, CloudFormation/CDK to rebuild
- **Pilot Light**:
    - Primary site is running at full
    - Pilot Light implies running a secondary environment only having the absolute minimum services running
    - In the event of a disaster the shutdown services can be spined up. Ongoing cost is low (only the "pilot light": live data replication + core always-on components), app servers are off/not provisioned until needed
    - Expected recovery time is a few 10s of minutes
    - Data is live (replicated continuously, RPO seconds-minutes); compute is "off" (AMIs/launch templates/IaC ready, ASG at 0)
    - AWS: Aurora Global Database / RDS cross-region read replica, DynamoDB global tables, S3 CRR, AWS Elastic Disaster Recovery (DRS) staging area
- **Warm Standby**:
    - Primary site is running at full, everything is replicated on the backup site at a smaller scale
    - Ready to be increased in size when failover is required
    - It is faster than pilot light approach and cheaper than active/active approach
    - Expected recovery time is a few minutes
    - Difference vs pilot light: warm standby can serve (some) traffic immediately - full stack is running, just smaller; scale out (ASG) during failover
- **Active/Active (Multi-site):**
    - Primary site is entirely replicated on a secondary site
    - Data is constantly replicated from the primary site to the backup
    - Costs are generally 200%
    - There is no concept of recovery time (RTO/RPO near zero)
    - Requires multi-region writes or a clear write region: DynamoDB global tables (multi-active), Aurora Global Database with write forwarding, Route 53 latency/weighted routing or Global Accelerator, data conflict handling
    - Additional benefits:
        - Load balancing across environments
        - Improved HA and performance
- Summary:
    - Backups: cheap and slow
    - Pilot Light: fairly cheap but faster
    - Warm Standby: costly, but quick to recover
    - Active/Active: expensive, 0 recovery time

| Strategy | RPO | RTO | Cost |
|---|---|---|---|
| Backup & Restore | hours | hours (up to 24h) | $ |
| Pilot Light | minutes | 10s of minutes | $$ |
| Warm Standby | seconds-minutes | minutes | $$$ |
| Multi-site Active/Active | near zero | near zero | $$$$ |

- Exam tip: pick the **cheapest** strategy that meets the stated RTO/RPO <span style="color: red;">EXAM</span>
- Failover must not depend on the failed region's control plane: prefer **data plane** actions (Route 53 health checks, ARC routing controls) over control plane operations (creating resources, changing DNS records via API) during an event

## DR Architecture - Storage

- Instance Store volumes:
    - Most high risk form of storage available
    - If the host fails, the instance store volumes will also fail
    - Should be viewed as temporary and unreliable storage
- EBS:
    - Volumes are created an run in a single AZ (failure of AZ means failure of EBS volumes)
    - Snapshots of EBS are stored in S3, will increase reliability
- S3: 
    - Data is replicated across multiple AZs
    - One-Zone: not regionally resilient
- EFS:
    - EFS file systems are replicated across multiple AZs
    - They are by default regionally resilient - failure of a region means failure of EFS volumes
    - EFS One Zone: single AZ. **EFS Replication**: async replica in another region (or same region), RPO typically 15 minutes; replica is read-only until failover
- S3 cross-region protection: **Cross-Region Replication** (needs versioning; S3 RTC - Replication Time Control - 99.99% of objects in 15 minutes, SLA), **Multi-Region Access Points** with failover controls (active/passive routing), Object Lock for immutability
- EBS: copy snapshots cross-region (manually, Data Lifecycle Manager or AWS Backup); fast snapshot restore to avoid first-read latency
- FSx: backups + cross-region copy via AWS Backup; FSx for NetApp ONTAP SnapMirror replication
- DR Architecture - Storage:
    ![DR Architecture - Storage](images/DRArchitectureStorage.png)

## DR Architecture - Compute

- EC2:
    - If the host fails, EC2 instances fails as well
    - An EC2 instance by itself is not resilient in any way
    - If the failure is limited to one host, the instance can move to another host in the AZ. The EBS volume can be presented to the new instance
    - Auto Scaling Group: can be placed in multiple AZs, if the instance fails in one AZ, the ASG's role is to recreate them in another
- ECS:
    - Can run it 2 modes: EC2 and Fargate
    - EC2 mode: DR architecture is similar as above
    - Fargate mode: containers are running on a cluster host managed by AWS being injected in VPCs
    - Fargate can provide automatic HA by running things in different AZs
- Lambda:
    - By default runs in public mode
    - In VPC mode (private) Lambdas are injected in VPCs. If an AZ fails, Lambda can be automatically injected in another subnet in a different AZ
    - It will take the failure if a region in order for Lambda to be impacted
- DR Architecture - Compute:
    ![DR Architecture - Compute](images/DRArchitectureCompute.png)

## DR Architecture - Databases

- Running databases on EC2 should be done in certain cases only!
- DynamoDB:
    - Data is replicated between multiple nodes in different AZs
    - Failure can occur only if the entire region fails
- RDS:
    - Requires creation of a subnet group which specifies which subnet can be used in a VPC for a DB
    - Normal RDS (not Aurora) involves a single instance or primary and standby instance running in different AZs
    - Data is stored in local storage for each instance, data is replicated **synchronously** to the standby (Multi-AZ instance - standby not readable). Read replicas use **asynchronous** replication <span style="color: red;">EXAM</span>
    - Multi-AZ DB cluster (MySQL/PostgreSQL): 1 writer + 2 readable standbys in 3 AZs, semi-synchronous, failover typically < 35 s
    - If the primary instance fails, automatic fallback is done to the standby
    - In case of Aurora, we can have one or more replicas in each AZs
    - Aurora uses a cluster storage architecture, storage is shared between running DB instances
    - Aurora storage keeps 6 copies across 3 AZs - it resists AZ failures but not the failure of the entire region (for that use Aurora Global)
- Global Databases:
    - DynamoDB Global Table: multi master replication between regional replicas.
    - Aurora Global Databases: read-write cluster in one region, secondary read cluster in other regions (up to 10 secondary regions). Replication happens at the storage layer, no additional load placed on the DB
        - Typical replication lag < 1 s (RPO ~1 s), cross-region failover RTO ~1 minute
        - **Switchover** (planned, no data loss) vs **failover** (unplanned, allow data loss); global writer endpoint follows the new primary; write forwarding from secondaries
    - DynamoDB global tables: multi-active, async replication (typically < 1 s) with last-writer-wins - or **multi-region strong consistency (MRSC)** mode (2025, 3 regions, RPO zero). Point-in-time recovery (35 days) for corruption
    - Cross Region Read Replicas for RDS: asynchronous replication but not done on the storage layer
- DR Architecture - Databases:
    ![DR Architecture - Databases](images/DRArchitectureDatabases.png)

## DR Architecture - Networking

- Networking at local level:
    - VPC are regionally resilient
    - Certain gateway objects like VPC Router and IGW are also regionally resilient
    - Subnets are tied to AZ they are located in, if the AZ fails, the subnet fails as well
    - LB: regional services, nodes are deployed into each AZ we select
    - By using a LB we can route traffic to AZs which are healthy
- Interface endpoint:
    - Are tied to an AZ
    - Multiple interface endpoints can be deployed into different AZs
- DR Architecture - Networking:
    ![DR Architecture - Networking](images/DRArchitectureNetworking.png)
- Global Networking:
    - Route53 can route globally to different regions (failover routing)
    - Route 53 health checks (endpoint, calculated, CloudWatch alarm-based) drive failover; Route 53 data plane is designed for 100% availability - avoid changing records via API (control plane, hosted in us-east-1) during a DR event
    - Global Accelerator: static anycast IPs, health-based failover between regional endpoints in seconds (no DNS caching issues)
    - CloudFront origin groups: origin failover (primary/secondary origin) for GET/HEAD requests

## AWS Backup

- Centralized, policy-based backup across services: EC2/EBS, RDS/Aurora, DynamoDB, EFS, FSx, S3, DocumentDB, Neptune, Redshift, Storage Gateway, VMware/on-prem, SAP HANA on EC2, EKS, etc.
- **Backup plan** (frequency, window, lifecycle to cold storage, retention, copy rules) + **backup vault** (KMS encrypted container for recovery points) + resource assignment (tags/resource IDs)
- **Organizations integration**: backup policies applied from management/delegated admin account to all accounts; cross-account monitoring <span style="color: red;">EXAM</span>
- **Cross-region and cross-account copies** for DR (copy recovery points to a vault in a DR account/region)
- **Vault Lock** (WORM): *governance* mode (can be removed with permissions) or *compliance* mode (immutable after grace period - not even root can delete) - ransomware protection
- **Logically air-gapped vault** (GA Aug 2024): vault in an AWS-owned service account, Vault Lock compliance mode by default, share recovery points with other accounts/orgs via **AWS RAM** and restore directly from them; **multi-party approval** (Jun 2025) to regain access even if the owning account is compromised/closed; supports primary backups directly into the vault for some resource types
- **Restore testing**: automated periodic restores to validate recoverability and measure restore duration (RTO evidence)
- **Audit Manager (AWS Backup Audit Manager)**: frameworks/controls to report backup compliance (e.g. "all resources backed up daily, retained 35 days, copied cross-region")
- Malware scanning of recovery points with **GuardDuty Malware Protection for AWS Backup** (Nov 2025)
- Continuous backup + PITR for RDS, S3, Aurora (RPO down to ~5 minutes for RDS)
- Exam: "centrally manage and enforce backups for all accounts" -> AWS Backup + Organizations backup policies; "immutable backups for ransomware" -> Vault Lock compliance / logically air-gapped vault

## Amazon Application Recovery Controller (ARC)

- Formerly Route 53 Application Recovery Controller
- **Zonal shift**: temporarily move traffic away from an impaired AZ for ALB/NLB, EC2 ASG, EKS (no cross-zone traffic) - expires automatically; **zonal autoshift**: AWS shifts traffic automatically when it detects an AZ impairment (requires practice runs) <span style="color: red;">EXAM</span>
- **Routing control**: highly available on/off switches (cluster of 5 regional endpoints, data plane) that change Route 53 health check state -> failover DNS without depending on control planes; **safety rules** (assertion/gating) prevent both cells being off
- **Region switch** (2025): fully managed multi-region recovery orchestration - plans with steps (Aurora Global failover, Route 53/ARC routing, ASG scaling, EC2/ECS/EKS, custom Lambda), active/passive or active/active, execution dashboards
- **Readiness check** (audit replica capacity/config across cells)
    > **Status (2026):** readiness check is no longer open to new customers (since Apr 30, 2026); AWS recommends ARC Region switch instead. Zonal shift, zonal autoshift, routing control and Region switch are unaffected.

## Other DR Tools

- **AWS Elastic Disaster Recovery (DRS)**: block-level continuous replication of servers - see `drs.md`
- **AWS Resilience Hub**: define RTO/RPO resiliency policies per application, assess the architecture (from CloudFormation, Terraform, EKS, AppRegistry), recommendations + SOPs + FIS experiments, drift detection
- **AWS Fault Injection Service (FIS)**: chaos engineering experiments (stop instances, AZ power interruption, cross-region connectivity disruption, API throttling) to validate DR/HA with stop conditions (CloudWatch alarms)
- **CloudFormation StackSets / CDK**: deploy identical infrastructure to the DR region in advance (pilot light/warm standby) or on demand (backup & restore)
- **AMI copy + Parameter Store / launch templates** per region; **ECR cross-region replication**; **Secrets Manager replicated secrets**; **KMS multi-region keys** - common "forgotten" dependencies in DR questions
