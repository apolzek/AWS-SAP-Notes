# VM Migrations Between AWS <=> On-Premises

## AWS Transform (2025+)

- Agentic AI service (GA May 2025) for migration and modernization: discovery, assessment/business case, dependency mapping, wave planning, network conversion (VMware networks => VPCs, subnets, security groups), server migration
- Flavors: AWS Transform for **VMware**, for **mainframe**, for **.NET/Windows**, custom code transformations; migration from VMware, Hyper-V, bare metal and other sources (Jul 2026)
- Brings together former Migration Hub capabilities (Strategy Recommendations, EC2 instance recommendations, Journeys, Orchestrator) and Migration Evaluator-style assessments
- Uses **AWS Transform MGN** as the replication engine for rehost
- Exam mapping: questions written before 2025 say "Migration Hub" for tracking and "ADS" for discovery; for new customers the equivalent is AWS Transform

## Application Discovery Service (ADS)

> **Status (2026):** closed to new customers since Nov 7, 2025 (stopped accepting new data from Discovery Connectors on Nov 17, 2025). Existing customers can keep using it, no new features. Use **AWS Transform** (discovery) instead

- Allows us to discover on-premises infrastructure:
  - What VMs we have
  - What CPU and memory are allocated to them
  - MAC addresses
  - Resource utilization
  - etc.
- It also tracks these properties over time for more effective migration
- ADS runs in 2 modes:
  - Agentless (Application Discovery Agentless Connector):
    - It is an OVA based virtual appliance that integrates with VMWare
    - It can measure performance and resource usage, information which can be obtained from the outside of a VM
  - Agent Based mode (**AWS Application Discovery Agent** installed in each VM/physical server - not the CloudWatch Agent): offers additional information from inside of a VM
    - Offers data gathering for network, processes, performance
    - We can see applications running on a VM
    - We can even see dependencies between VM based on network activity
    - Works for physical servers and any hypervisor (agentless only covers VMware)
- Third option: import data via CSV template (e.g. from a CMDB or third-party tools)
- ADS does not migrate anything, it helps us discover VM instances and relationships between them in order to be migrated
- This information is valuable if we have thousands of instances we want to migrate
- Data stored encrypted; can be exported to S3 and queried with **Athena** (data exploration) and visualized in QuickSight
- ADS integrates with AWS Migration Hub and Athena
- **AWS Migration Hub**: single place to track migrations of different types (VM migrations via MGN, database migrations via DMS) across tools and regions; home region concept
  > **Status (2026):** closed to new customers since Nov 7, 2025 - replaced by **AWS Transform** (same applies to Migration Hub Refactor Spaces, Strategy Recommendations, Orchestrator, Journeys)

## Migration Evaluator

- Formerly TSO Logic: builds a data-driven **business case** (TCO, right-sized EC2 recommendations, license optimization BYOL vs license-included)
- Agentless collector or import (RVTools, CMDB exports); output: Quick Insights report
- Answer for "create a business case / estimate cost of running on AWS before migrating"
- Assessment capabilities are also available in AWS Transform (expanded 2026: what-if scenarios, RVTools/CMDB inputs)

## Server Migration Service (SMS)

> **Status (2026):** AWS SMS was **discontinued on March 31, 2022**. CloudEndure Migration was also discontinued. Use **AWS Transform MGN** (Application Migration Service) for all rehost migrations. Kept here for historical context / older exam questions

- Used to migrate whole VMs into AWS (including OS, Data, Apps, etc.)
- This is the tool which actually **performs** the migration
- It runs in **agentless** mode using a connector. The connector is VM which runs on on-premises within our existing VM environment
- The connector only integrates with VMware, Hyper-V and AzureVM
- SMS does incremental replication of live volumes => can migrate a VM in the background while it is still operating
- Offers orchestration of multi-servers migrations
- Creates AMIs which can be used to create EC2 instances
- It can be used with other tooling such as CloudFormation to automate repeated deployment of instances
- It also integrated with AWS Migration Hub

![SMS](images/SMS.png)

## AWS Application Migration Service (MGN) - now AWS Transform MGN

> **Status (2026):** renamed to **AWS Transform MGN** in June 2026 (same service, replication engine behind AWS Transform; can be used via its own console or through AWS Transform agentic workflows)

- Is used to migrate servers from on-prem to AWS
- It allows companies to lift-and-shift a large number of physical, virtual, or cloud servers without compatibility issues, performance disruption, or long cutover windows
- AWS recommends agent-based replication when possible as it supports continuous data protection (CDP)
- AWS MGN provides this agent (AWS Replication Agent). MGN uses an EC2 Launch Template which is then used to launch EC2 instances
- Agentless replication is available for VMware vCenter (MGN vCenter Client) when agents can't be installed
- AWS MGN provides a way to work out what kind of dependencies do we have between databases, app servers, web servers, etc. We can group these servers (applications and waves) and migrate them together
- It can also provide CloudFormation templates for deployment
- Difference between SMS and MGN:
  - Server Migration Service utilizes incremental, snapshot-based replication and enables cutover windows in hours
  - Application Migration Service utilizes continuous, block-level replication and enables short cutover windows measured in minutes
- AWS recommends to use MGN instead of SMS
- We can migrate virtual and physical servers as well
- Sources: physical, VMware, Hyper-V, other clouds (Azure, GCP), and EC2 (cross-region/cross-AZ migrations)

### How MGN works

1. Install the replication agent on source servers (outbound TCP 443 to MGN, TCP 1500 to replication servers)
2. Data replicated continuously (block level) to lightweight **replication servers** in a **staging area subnet**, stored on low-cost EBS volumes + snapshots
3. **Launch test instances** (non-disruptive tests) => validate
4. **Cutover**: stop source, final sync, launch cutover instances; conversion server converts boot/drivers to run on EC2
5. Finalize cutover, archive source server
- Post-launch actions: SSM-based automation (install SSM agent, CloudWatch agent, convert Windows licensing, run custom scripts, modernization)
- Replication over internet, VPN, DX or private connectivity (VPC endpoints); data encrypted in transit and at rest (EBS KMS)
- Pricing: free for 90 days (2,160 hours) per source server; you pay for the staging area resources (EC2, EBS)

## Related Services

- **VM Import/Export**: import VM images (OVA, VMDK, VHD/VHDX, raw) from S3 as AMIs/EBS snapshots, or export EC2 instances to VMware/Hyper-V/Citrix. One-off, no continuous replication => use for small/offline cases, BYOL scenarios
- **AWS Elastic Disaster Recovery (DRS)**: same CDP block-level technology as MGN, but for DR (keep replicating, failover/failback, point-in-time recovery). Replaced CloudEndure Disaster Recovery
- **VMware Cloud on AWS / Amazon EVS**: relocate VMware workloads without converting VMs (HCX)
- **AWS App2Container**: containerize Java/.NET apps running on VMs => ECS/EKS
- **AWS Outposts**: retain workloads on-prem with AWS APIs

## Exam Tips

- "Migrate hundreds of servers with minimal downtime, cutover in minutes" => MGN (AWS Transform MGN)
- "Discover servers, utilization and dependencies before migrating" => AWS Transform discovery (ADS agent-based for dependencies in older questions)
- "Business case / TCO before migration" => Migration Evaluator (or AWS Transform assessment)
- "Track migration progress across tools" => Migration Hub (legacy) / AWS Transform
- "DR for on-prem servers to AWS with low RPO (seconds) and RTO (minutes)" => Elastic Disaster Recovery
- "Import a single VM image to create an AMI" => VM Import/Export
- Any answer with SMS or CloudEndure Migration for a new design is wrong today
