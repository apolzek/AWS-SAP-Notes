# EC2

## EC2 Purchase Options (Launch Types)

- **On-Demand (default)**:

  - Average of anything, no specific cons or pros
  - On-demand instances are isolated but multiple customer instances run on a shared hardware
  - Multiple instance types (different sizes) can run on the same EC2 hosts, consuming a different allocation of resources
  - Billing: **per-second** billing while an instance is running, if a system is shut down, we don't get billed for that
  - Associated resources such as storage consume capacity, we will be billed regardless the instance is running or it is stopped
  - We should always start the evaluation process using on-demand
  - With on-demand there are no interruptions. We start an instance and it should run as long as we don't decide to shut it down
  - In case of resource shortage the reserved instances receive highest priority, consider them instead of on-demand in case of business critical systems
  - On-demand offers predictable pricing without any discount options
  - If you are unsure of duration or type of workload, On-demand should be considered.

- **Spot instances**:

  - Cheapest way to get EC2 compute capacity
  - Spot pricing is selling EC2 capacity at lower price in order make use of spare EC2 capacity on the host machines
  - You can pick a maximum spot price you are willing to pay (default and recommended: leave it at the On-Demand price)
  - Since 2017 the spot price changes gradually (long-term supply/demand) - no bidding. Interruptions happen mainly when AWS **needs the capacity back**, not because of price spikes
  - We should never use the spot instances for workloads which can't tolerate interruptions
  - Anything which can tolerate interruptions and can be re-triggered is good for spot

- **Reserved Instances (Standard)**:

  - On-demand is generally used for unknown or short term usage, reserved is for long term consistent usage of EC2
  - Reservations:
    - They are commitments that we will use a instance/set of instances for a longer amount of time
    - The effect of a reservation is to reduce the per second cost or remove it entirely
    - Reservation needs to be planned in advance
    - We pay for unused reservations
    - Reservations can be bought for a specific type of instances. They can be region and AZ locked
    - Az locked instances reserve EC2 capacity
    - If an instance is reserved for a region,it doesn't reserve capacity but it can benefit any instances launched in any AZ in that region.
    - Reservations can have a partial effects in a sense the we can get discounts for larger instances compared to which the reservation was purchased
    - We can commit to reservations of 1 year of 3 years
    - Payment structures:
      - No upfront: we pay per second a lower amount of fee compared to on-demand. We pay even if the instance is not used
      - All upfront: the whole cost of the 1 or 3 years. No second per fee payment will be required. Offer the greatest discount
      - Partial upfront: we pay a reduced fee upfront for smaller per second usage
  - Reserved instances are good for components which have known usage, require consistent access for compute for a long term basis
  - **Standard vs Convertible RIs**:
    - Standard: up to ~72% discount, attributes (family, OS, tenancy) fixed, can be sold on the **RI Marketplace**
    - Convertible: up to ~66% discount, can be exchanged for another family/OS/tenancy of equal or greater value, can **NOT** be sold on the marketplace
  - Regional RIs (Linux/Unix, default tenancy) have **instance size flexibility** within the same family (normalization factor, e.g. 1x `m5.xlarge` = 2x `m5.large`)
  - RIs (and Savings Plans) are shared across all accounts of an AWS Organization by default (consolidated billing) - sharing can be turned off per account
  - Exam tip: for new purchases AWS recommends **Savings Plans** over RIs (more flexible, same discount). RIs still matter for RDS, ElastiCache, OpenSearch, Redshift (where SPs did not exist until Database Savings Plans) and for zonal capacity reservation

- **Dedicated Hosts**:

  - They are EC2 hosts allocated to a customer entirely
  - They are hosts designed for specific instances, ex. A, C, R, etc.
  - Hosts come with all of the resources we expected from a physical machine: number of cores and CPUs, memory, local storage and connectivity
  - We pay for the host, we don't pay anymore for instance usage per second in case we launch instances on the host
  - We have a capacity for a dedicated hosts, we can launch different sizes of instances based on the available capacity
  - Reasons for dedicated hosts: we want to use software which is licensed for number of cores or number of sockets
  - Host affinity: feature of dedicated hosts. Links instances to hosts, if we stop and start the instance, it will remain on the same host
  - Only our instances will run on a dedicated hosts
  - Capacity management:
    - We have to manage our capacity in terms of under utilization of the host
    - We have a limited capacity in terms of how many EC2 instances we can launch
  - Visibility of sockets/cores/host ID => BYOL (Windows Server, SQL Server, Oracle, RHEL) - track with **AWS License Manager** (host resource groups auto-allocate/release hosts)
  - Can be shared with other accounts via AWS RAM; Dedicated Host Reservations and Savings Plans give discounts
  - Exam tip: "per-socket/per-core license" or "need to see host hardware" => Dedicated Host; "just no shared hardware" => Dedicated Instance

- **Dedicated Instances**:

  - Our instances run on an EC2 host with other instances of ours. The host is not shared with other AWS customers, no other customers will use the same hardware
  - We don't pay for the host, nor do we share the host
  - There are some extra fees for this kind of purchase option:
    - One-of hourly fee for any regions in which we are using dedicated instances
    - There is a fee for the dedicated instances themselves
  - Dedicated instances are common in industries where we cannot share hardware
  - No extra capacity management required from us
  - Cheaper that dedicated host but still dedicated hardware

![Purchase options](images/dedicated.png)

- **Scheduled Reserved Instances**:

  > **Status (2026):** Scheduled RIs are no longer available for purchase (AWS has no capacity for them and no plans to bring them back). Alternative: On-Demand Capacity Reservations with a schedule (create/cancel via automation) + Savings Plans, or future-dated Capacity Reservations / Capacity Blocks.

  - Great for long term requirements which does not run constantly, ex. batch processing running 5 hours/day
  - For scheduled reserved instances we specify a time window. The capacity can be used only during the time window
  - Limitations:
    - Does not support all kind of instance types
    - Minimum purchase per year is 1200 hours, minimum commitment is 1 year

## Capacity Reservations

- AWS prioritizes any scheduled commitment for delivering EC2 capacity
- After scheduled instances on-demand is prioritized
- The leftover capacity can be used for spot instances
- Capacity reservation is different compared to reserved instances
- **Regional reservation (reserved instance):**
  - Provides a billing discount for valid instances launched in any AZ in that region
  - While this is flexible, region reservation does NOT reserve capacity within az AZ - risky if the capacity is limited during a major fault
- **Zonal reservation (reserved instance):**
  - Same billing discount as for region reservation, but the reservation applies only to specific AZs
- Regional/zonal reservation commitment is 1 or 3 years
- **On-Demand capacity reservation:**
  - On-Demand capacity reservation: can be booked to ensure we always have access to capacity in an AZ when we need it but at full on-demand price.
  - No term limits, but we pay regardless if we consume the reservation or not
  - Capacity reservations do not provide any billing benefit, we just reserve the capacity for EC2 compute
  - Combine an ODCR with a **Regional RI or Savings Plan** => capacity guarantee + discount (common exam answer for "guaranteed capacity in a specific AZ at a discount")

![Capacity Reservations](images/Capacity.png)

### Capacity Reservation details (ODCR)

- Created for a specific instance type, platform, AZ, tenancy and count; billing starts as soon as the reservation is active
- Can be created immediately or **future-dated** (book capacity for a future start date, with a minimum commitment duration)
- Instance eligibility: `open` (any matching instance uses it automatically) or `targeted` (only instances launched with the reservation ID)
- Can be shared with other accounts via **AWS RAM**, grouped in **Capacity Reservation Resource Groups** (targeted by launch templates/ASGs)
- **Capacity Reservation Fleets**: reserve capacity across multiple instance types in one request
- **Interruptible Capacity Reservations** (Nov 2025): owner lends unused ODCR capacity to other workloads in the same Organization and can reclaim it at any time (consumers get an interruption notice) - no extra cost
- ODCRs can be used in Local Zones and on Outposts

### Capacity Blocks for ML

- Reserve **GPU/Trainium instances** (P5/P5e/P5en, P6, P4d, Trn1/Trn2, UltraServers) for a **future** time window - days to weeks (short-duration ML training/fine-tuning, experiments)
- Start date up to **8 weeks** in the future, up to 64 instances per block / 256 instances across blocks (per Organization per date)
- Price is dynamic (supply/demand) and paid upfront; blocks can be extended but **not cancelled**
- Instances are placed in EC2 UltraClusters (low-latency, non-blocking network); instances must target the reservation ID; works with ASG (scheduled scaling at block start), EKS, SageMaker HyperPod, ParallelCluster
- Exam tip: "need guaranteed GPUs for 2 weeks next month without paying for a long-term reservation" => **Capacity Blocks**

## EC2 Savings Plan

- A hourly commitment ($/hour) for 1 or 3 years term
- Saving Plan types:
  - **Compute Savings Plans**: up to 66% saving versus on-demand. Apply to EC2 (any family, size, AZ, region, OS, tenancy), Fargate and Lambda
  - **EC2 Instance Savings Plans**: up to 72% saving, locked to an instance **family + region** (e.g. M7g in us-east-1), flexible on size, OS, tenancy, AZ
  - **SageMaker AI Savings Plans**: up to 64% for SageMaker AI instance usage
  - **Database Savings Plans** (Dec 2025): up to 35%, 1-year term, no upfront; apply across engines, instance families, deployment options and **regions** (Aurora, RDS, DynamoDB, ElastiCache, DocumentDB, Neptune, Keyspaces, Timestream, DMS; OpenSearch and Neptune Analytics added 2026)
- Payment options: No upfront, Partial upfront, All upfront
- Resources have an on-demand $ rate AND a savings plan rate
- When consuming resources within the savings plan commitment => billed at the reduced savings plan rate (for all EC2, Fargate, Lambda) until the commitment is reached. Then => on-demand.
- Savings Plans do **NOT** reserve capacity - use ODCRs for that
- Discount application order: RIs are applied first, then Savings Plans, then On-Demand rates
- Can't be cancelled (small-purchase return window only: within 7 days, for plans of up to $100/hour)
- Use **Savings Plans Purchase Analyzer** / Cost Explorer recommendations to size the commitment

### RI vs Savings Plan vs ODCR vs Spot (exam summary)

| Need | Choose |
| --- | --- |
| Steady-state, flexible across EC2/Fargate/Lambda | Compute Savings Plan |
| Steady-state, fixed family + region, max discount | EC2 Instance SP or Standard RI |
| Guaranteed capacity in one AZ, any duration | On-Demand Capacity Reservation (+ SP/Regional RI for discount) |
| Guaranteed GPUs for a short future window | Capacity Blocks for ML |
| Fault-tolerant, interruptible, cheapest | Spot |
| BYOL per-socket/per-core licenses, compliance | Dedicated Hosts |

## EC2 Advanced Networking

- Instances are created with a primary ENI, this can **NOT** be removed or detached from the instance
- Secondary ENIs can be added to an instance which can be in different subnets than the instance **(NOT AZs!)**
- Secondary ENIs can be detached and attached to other instances
- Security Groups are associated with an ENI, not an EC2 instances
- Every instance is allocated a primary private IPv4 address from the subnet range (assigned to the primary ENI). This IP address stays the same within the lifetime of EC2 instance
- The primary IP address is exposed to the OS
- ENIs can also have one or more secondary IP addresses depending on the instance type
- Public IP address is allocated to the instance if we launch it in a subnet where this is enabled or we explicitly enable a primary address to the instance. These public IP addresses are dynamic and they can change if the EC2 instance is moving to another EC2 host
- Public IPs are not visible to the OS
- In order to get static public IP addresses, we can associate an Elastic IP to the instance
- We can allocate one public IP per private IP on an instance (larger instances can have more private IPs)
- Public IPv4 pricing (since Feb 2024): **every public IPv4 address is charged** ($0.005/hour), in-use or idle, Elastic IP or auto-assigned (exceptions: BYOIP addresses; 750h/month free with the legacy Free Tier)
  - Previously only idle EIPs (not associated, or associated with a stopped instance) were charged - outdated exam material may still say this
  - Cost reduction: IPv6, NAT Gateway / ALB in front of private instances, EC2 Instance Connect Endpoint instead of public bastions; audit with **VPC IPAM Public IP Insights**
- ENIs can have 1 or more IPv6 addresses, 1 MAC address and 1 or more Security Groups
- IPv6 addressing if enabled, means that all IPv6 addresses are publicly routable
- IPv6 addresses are **ALWAYS** visible to the OS
- Source/destination checks:
  - Each ENI has a flag which can be disabled
  - By default => enabled, if disabled the ENI can process traffic which was not created by the EC2 instances or traffic for which the EC2 instance is not the destination
  - Must be disabled for NAT instances, virtual routers, firewall appliances (not needed behind a GWLB)
- **Enhanced Networking**: SR-IOV via **ENA** (up to 100-200+ Gbps on current instances) - higher PPS, lower latency, no extra cost
  - **ENA Express**: uses AWS SRD to raise single-flow bandwidth up to 25 Gbps between instances in the same AZ
  - **EFA (Elastic Fabric Adapter)**: OS-bypass for HPC/ML (MPI, NCCL), Linux only, tightly-coupled workloads - exam keyword "HPC with low-latency inter-node communication"
- Prefix delegation: assign /28 IPv4 or /80 IPv6 prefixes to an ENI (more IPs per instance, e.g. for EKS pods)

## Instance Platform: Nitro and Graviton

- **Nitro System**: dedicated hardware cards for VPC networking, EBS, instance storage + lightweight hypervisor => near bare-metal performance, bare metal instances, more ENIs/EBS volumes, security (no operator access)
  - Required for many features: ENA Express, EBS io2 Block Express, Nitro Enclaves, NitroTPM, hibernation on newer types, IPv6-only subnets
  - **Nitro Enclaves**: isolated compute environment inside an instance for processing highly sensitive data (no persistent storage, no network, attested with KMS)
- **Graviton** (AWS Arm CPUs: Graviton2/3/4, instance families ending in `g`, e.g. `m8g`, `c7gn`, `r8g`): up to ~40% better price/performance vs comparable x86, lower energy use
  - Requires arm64 builds (AMIs, containers - multi-arch images). Also available for Lambda, Fargate, RDS/Aurora, ElastiCache, OpenSearch
  - Exam tip: "reduce cost with minimal changes for interpreted/containerized workloads" => move to Graviton
- Instance naming: `m7i.2xlarge` => family `m`, generation `7`, attribute `i` (Intel) / `a` (AMD) / `g` (Graviton) / `d` (local NVMe) / `n` (network optimized) / `e` (extra memory) / `flex` (flex instances, cheaper, not always at full CPU)
- Burstable `t` instances: CPU credits; `unlimited` mode (default for T3/T4g) can incur extra charges

## Instance Metadata Service (IMDS)

- Endpoint `http://169.254.169.254/latest/meta-data/` (IPv6: `[fd00:ec2::254]`), exposes instance info, user data and **IAM role temporary credentials**
- **IMDSv1**: simple request/response - vulnerable to SSRF/open reverse proxies
- **IMDSv2**: session-oriented - `PUT` to get a token (TTL up to 6h), then send header `X-aws-ec2-metadata-token`; hop limit (default 1, use 2 for containers) protects against forwarding
- Defaults (2024+): new instance types (released 2024+) are IMDSv2-only by default; Amazon Linux 2023 AMIs default to IMDSv2-only; account-level default settings per region can enforce `HttpTokens=required`
- Enforce org-wide: SCP condition `ec2:MetadataHttpTokens` = `required` / `ec2:RoleDelivery` >= 2.0, AWS Config rule `ec2-imdsv2-check`; monitor with CloudWatch metric `MetadataNoToken`
- IMDS can be disabled entirely; instance tags in metadata are opt-in

## EC2 Access: Session Manager and EC2 Instance Connect

- **Systems Manager Session Manager**: shell access without SSH keys, inbound ports or bastions; IAM-controlled, logs to S3/CloudWatch; needs SSM Agent + outbound to SSM endpoints (VPC interface endpoints for private subnets)
- **EC2 Instance Connect**: pushes a one-time SSH public key (valid 60s) via API, IAM-controlled
- **EC2 Instance Connect Endpoint (EIC Endpoint)** (2023): connect via SSH/RDP to instances in **private subnets** without public IPs, IGW, bastion or agent
  - Created in a subnet, controlled by IAM + security groups, connections logged in CloudTrail, no additional cost
  - Exam tip: "remove bastion hosts / public IPs, keep SSH/RDP" => EIC Endpoint or Session Manager (Session Manager if auditing of session content is required)

## EC2 Hibernation

- Saves RAM contents to the encrypted **EBS root volume**, on start the RAM is restored => fast resume of long-initializing apps (caches, in-memory state)
- Requirements: root volume EBS and **encrypted**, large enough for RAM; RAM < 150 GB; supported families (C, M, R, T, etc.) and OSs; enabled at launch only
- Max hibernation period: 60 days; can't change instance type while hibernated; ASG **warm pools** can keep pre-initialized instances in `Hibernated` state; Spot instances can hibernate on interruption
- Billing: no instance charges while hibernated (only EBS, EIP)

## Instance Recovery, Lifecycle and Other Features

- **Simplified automatic recovery** (default on supported types): instance is recovered on another host if a system status check fails; keeps instance ID, private IPs, EIPs, EBS volumes and metadata. Not for instances with instance store volumes
- CloudWatch alarm action `recover` for explicit recovery
- **Status checks**: system status (AWS hardware/network - fix by stop/start = move host), instance status (OS/config), attached EBS status check
- Termination protection / stop protection; shutdown behavior (stop or terminate)
- **Scheduled events** (maintenance, retirement) visible in AWS Health
- **Instance store**: ephemeral NVMe, lost on stop/hibernate/terminate/host failure - survives reboot only

## Bootstrapping vs AMI Baking

- User data limit: **16 KB** (raw, before base64), runs as root, by default only on first boot
- Signal completion with `cfn-signal` / CloudFormation `CreationPolicy`, or ASG lifecycle hooks

- Bootstrapping:
  - Is a way of building EC2 instances in a flexible way. NOT fast, but super flexible.
  - Flexible, automated building of EC2 instances
  - We provision EC2 instances and add a script to the **User data**
  - **CloudInit** runs the script on the instance when the instance is launched
  - This process can take longer, although it is very flexible
  - When finished, ec2 instance is ready to use.
- AMI Baking:
  - We front-load the time and effort required to configure an instance
  - We launch an EC2 instance and perform the necessary tasks (installation + bootstrapping) and then we create an AMI from it
  - We can use that AMI to deploy many instances quickly.
  - The tradeoff is, it harder to change the AMI.
  - AMI baking and bootstrapping are not mutually exclusive.
  - Usually perform the time-consuming tasks and bake an AMI. Then leave the flexible and config parts for the bootstrapping.
  - <span style="color: orange;">If there are any exam questions related to time taken to launch the EC2 instance, think of AMI baking.</span>
- **EC2 Image Builder**: managed pipelines to build, harden, test and distribute AMIs (and container images) across regions/accounts - the managed answer for "automate golden AMI creation and patching"
- AMIs are regional: copy with `CopyImage` (can re-encrypt with a different KMS key); share with accounts/OUs/Organization (encrypted AMIs require sharing the KMS CMK, not possible with the AWS managed key)
- AMI deprecation dates, deregistration protection and **Recycle Bin** for accidental deregistration; **Allowed AMIs** account setting restricts which AMIs can be launched
- Fast launch for Windows AMIs (pre-provisioned snapshots)

![AMI Baking/Bootstrapping](images/AMI.png)

## EC2 Placement Groups

- Allow us to influence EC2 instance placements, ensuring that instances are close to each other or spread out
- An instance can be in only one placement group at a time; placement groups can't be merged; a stopped instance can be moved into/out of a placement group (`modify-instance-placement`)
- Placement groups can be shared across accounts (AWS RAM)
- Insufficient capacity error in a cluster placement group => stop and start all instances (may move them to hardware with capacity) or use an ODCR in the placement group
- There are 3 types of placements groups in AWS:
  - **Cluster**: max performance - any instances in a single placement groups are physically close
  - **Spread**: max availability & resiliency - instances are all using different underlying hardware
  - **Partition**: Spread but >7 instances per AZ - groups of EC2 instances which are spread apart on different host hardware

### Cluster Placement Groups

- Used for highest possible performance
- Best practice is to launch all of the instances at the same time which will be part of the placement group. This ensures that AWS allocates capacity in the same location
- Cluster placement groups are located in the same AZ, when the first instance is launched, the AZ is determined from that 1st instance
- Ideally the instances in a cluster placement group are located on the same rack, often on the same EC2 host
- All instances have fast direct bandwidth between each other (they can achieve single stream transfer rate of 10Gbps vs 5Gbps which is achievable normally)
- They offer the lowest latency possible and max PPS(packets per second) possible in AWS
- To achieve these levels of performance we need to use instances with high performant networking: instances with more bandwidth and with Enhanced Networking
- Cluster placement groups should be used for highest performance. They offer no HA and very little resilience
- Considerations for cluster placement groups: <span style="color: #ff5733;">Important for EXAM!!!</span>
  - We can not span over AZs, the AZ is locked when the first instance is launching
  - We can span over VPC peers, but this will impact performance negatively
  - Cluster placement groups are not supported for every instance type
  - _Recommended_: use the same type of instances and launch them at the same time
  - Cluster placement groups offer 10 Gbps for single stream performance
  - Use cases: High performace, fast speeds, low latency

### Spread Placement Groups

- They offer the maximum possible availability and resiliency
- They can span multiple AZs
- Instances in the same spread placement group are located on different racks, having isolated networking and power supplies
- There is a limit of 7 instances per AZ in case of spread placement groups. This is because each instance is a completely separate instance rack
- Considerations: <span style="color: #ff5733;">Important for EXAM!!!</span>
  - Spread placement provides infrastructure isolation
  - Hard limit: 7 instances per AZ (rack-level spread); on Outposts host-level spread is available
  - We can not use dedicated instances or hosts
  - Use cases: Small number of critical instances that need to be kept seperated from each other. May be mirrors of file server, or may be different domain controllers within an organization.

### Partition Placement Groups

- Similar to spread placement groups
- They are designed for situations when we need more than 7 instances per AZ but we still need separation
- Can be created across multiple AZs in a region
- At creation we specify the number of partitions per AZ (max 7 per AZ)
- Each partition has its own rack with isolated power and networking
- We can launch as many instances as we need in a partition group. We can select the partition by hand or we can let EC2 decide on a partition for a new instance
- Use cases for partition groups: huge scale parallel processing, HDFS, HBase, Cassandra, topology aware applications<span style="color: #ff5733;">Important for EXAM!!!</span>
- Instances can be placed in a specific partition or we can let AWS to decide
- Offer visibility in partitions and you can see which instance is in which partition.

![Partition Placement Group](images/Partition.png)

## EC2 Spot Instances

- Can get a discount of up to 90% compared to On Demand instances
- We can define a max spot price and get he instance of our price is bigger than the current price
- If the current spot price goes beyond our max price (or AWS needs the capacity back), the instance is interrupted with a **2 minutes interruption notice** (instance metadata + EventBridge event)
- Interruption behavior: `terminate` (default), `stop` or `hibernate` (stop/hibernate only for persistent requests / EC2 Fleet `maintain`)
- **EC2 Instance Rebalance Recommendation**: earlier signal that a Spot instance is at elevated risk of interruption - used by ASG **Capacity Rebalancing** to proactively launch a replacement
- If we don't want our spot instance to be reclaimed by AWS, we can use a **Spot Block**

  > **Status (2026):** Spot Blocks (defined-duration Spot) are discontinued - not available to new customers since July 1, 2021 and fully retired after Dec 31, 2022. For uninterrupted short windows use On-Demand + ODCR or Capacity Blocks for ML.

  - We can block a spot instance during a specified time frame (1 to 6 hours) without interruptions
  - In rare situations the instance may be reclaimed
- Spot best practices (exam):
  - Be **flexible**: many instance types (attribute-based instance type selection), sizes and AZs
  - Use allocation strategy **`price-capacity-optimized`** (recommended default) - pools with most capacity and lowest price => fewer interruptions
  - Use ASG / EC2 Fleet (not `RequestSpotInstances` - legacy API) and handle the 2 min notice (checkpointing, draining from ELB, ECS/EKS node termination handlers)
  - Spot placement score + Spot Instance Advisor (interruption frequency) to choose regions/types
  - Spot is not for stateful databases, single-instance critical workloads or long jobs without checkpoints
- Use cases for spot instances: batch jobs or workloads that are resilient to failure
- We can launch spot instances with a spot request. A spot request contains the following information:
  - Maximum price
  - Desired number of instances
  - Launch specification
  - Request type: on-time, persistent
  - Valid from, valid until
- Request types:
  - One time request: as soon as the request is fulfilled, the request will go away
  - Persistent request: the number of instances is attempted to be kept even if some instances are reclaimed, meaning that the request will not go away as soon as it is completed first time
- Canceling a spot instances: in order ot cancel a spot instance, it has to be in an **open**, **active** or **disabled** state
- Spot instance states:
  ![Spot instance states](images/spot_request_states.png)
- Cancelling a spot request, it will not terminate the instances themselves. In order to terminate instances, first we have to terminate the spot request, if there is one active

## Spot Fleets

> **Status (2026):** Spot Fleet (`RequestSpotFleet`) is a legacy API with no planned investment. AWS recommends **EC2 Auto Scaling** (mixed instances policy) or **EC2 Fleet** (`CreateFleet`, CLI/API only, types `instant`, `request`, `maintain`) instead.

- Spot Fleet - set of spot instances + (optional) on-demand instances
- The spot fleet will try to meet the target capacity with price constraints
- A launch pool can have the following can have different instance types, OS, AZ
- We can have multiple launch pools, so the fleet can choose the best
- Spot fleet will stop launching instances the target capacity is reached
- Strategies to allocate spot instances:
  - **lowestPrice**: the spot fleet will launch instances from the pool with the lowest price (highest interruption risk - not recommended)
  - **diversified**: distribute instances across all pools
  - **capacityOptimized**: launch instances based on the optimal capacity for the number of instances
  - **priceCapacityOptimized**: pools with most available capacity, then lowest price - **recommended** for most workloads
- Spot fleets allow us to automatically request spot instances with the lowest price
