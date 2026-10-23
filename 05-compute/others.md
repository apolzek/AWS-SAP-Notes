# Other Compute Related Products

## AWS Outposts

- AWS Outposts is a fully managed service that extends AWS infrastructure, services, APIs, and tools to customer premises
- An Outpost is a pool of AWS compute and storage capacity deployed at a customer site
- AWS operates, monitors, and manages this capacity as part of an AWS Region
- We can create subnets on our Outpost and specify them when we create AWS resources such as EC2 instances, EBS volumes, ECS clusters, and RDS instances
- Instances in Outpost subnets communicate with other instances in the AWS Region using private IP addresses, all within the same VPC
- Not every AWS service is supported within an Outpost, for a list see: https://docs.aws.amazon.com/outposts/latest/userguide/what-is-outposts.html#services
- Form factors:
  - **Outposts racks**: 42U racks, scale to multiple racks; EC2, EBS, S3 on Outposts, ECS, EKS, RDS, EMR, ElastiCache, ALB, Route 53 Resolver, etc. (second-generation racks launched 2025 with newer instance types)
  - **Outposts servers**: 1U/2U servers for space-constrained sites (retail stores, branch offices) - EC2, ECS, IoT Greengrass; no EBS (instance storage only)
- Connectivity:
  - **Service link**: encrypted VPN connection to the parent region (control plane, monitoring, intra-VPC traffic) - needs a reliable connection (DX recommended)
  - **Local gateway (LGW)** (racks): routes traffic between the Outpost and the on-premises network; **CoIP** (customer-owned IP pools) for direct on-prem access
  - If the service link is lost: running instances keep working, but no control plane actions (can't launch new instances, no API calls) => design for disconnected operation
- Capacity is fixed (what you ordered) - plan for N+1, no elastic scale beyond installed hardware; AWS handles hardware replacement
- Pricing: 3-year term (all/partial/no upfront)
- Use cases: **data residency**, low latency to on-prem systems, local data processing, migration with on-prem dependencies (exam: "must stay in the customer data center but use AWS APIs" => Outposts)

## AWS Local Zones

- Extension of a region placed close to large population/industry centers (e.g. Los Angeles, Lagos) - single-digit ms latency for end users
- Opt-in per Local Zone; you create a subnet in the Local Zone inside a VPC of the **parent region**
- Supported services subset: EC2, EBS, ECS, EKS, ALB, FSx, ElastiCache, RDS (varies per zone), Direct Connect in some zones
- Use cases: media rendering, real-time gaming, live streaming, ML inference, hybrid migrations with latency-sensitive on-prem dependencies
- **Dedicated Local Zones**: AWS-managed infrastructure placed in a customer/partner facility, exclusive to one customer (sovereignty, regulated workloads)
- Local Zones vs Outposts: Local Zones = AWS-owned facility near users, elastic, pay-as-you-go; Outposts = in **your** data center, fixed capacity

## AWS Wavelength

- AWS Wavelength is just like having an Availability Zone in a phone carrier's 'edge' network
- Wavelength deploys standard AWS compute and storage services to the edge of telecommunication carriers' 5G networks
- We can extend a virtual private cloud (VPC) to one or more Wavelength Zones
- We can then use AWS resources such as Amazon Elastic Compute Cloud (Amazon EC2) instances to run the applications that require low latency or edge resiliency within the Wavelength Zone
- AWS resources on Wavelength:
    - EC2 Auto Scaling
    - EKS clusters
    - ECS clusters
    - Systems Manager
    - CloudWatch, CloudTrail
    - CloudFormation
    - Application Load Balancers
- The services in Wavelength are part of a VPC that is connected over a reliable connection to an AWS region
- Traffic from mobile devices on the carrier network reaches instances via a **carrier gateway** and **carrier IP addresses** (no internet hop) - lowest latency for 5G devices
- Use cases: connected vehicles, AR/VR, real-time gaming, smart factories, video analytics at the edge
- Exam tip: "ultra-low latency for **5G mobile** users" => Wavelength; "low latency for users in a metro area" => Local Zones; "on-premises" => Outposts

## Edge Compute Comparison

| Option | Location | Typical latency target | Notes |
| --- | --- | --- | --- |
| Region / AZ | AWS data centers | tens of ms to users | full service catalog |
| Local Zone | AWS facility in a metro area | single-digit ms to metro users | parent region VPC, opt-in |
| Wavelength Zone | Telco 5G network | single-digit ms to 5G devices | carrier gateway |
| Outposts | Customer data center | on-prem LAN latency | fixed capacity, service link |
| IoT Greengrass / Outposts servers | Remote/branch sites | local | Snow Family closed to new customers (Nov 2025) |
| CloudFront Functions / Lambda@Edge | Edge locations | CDN edge | request/response logic only |

> **Status (2026):** AWS Snowball Edge (and all Snow Family devices) are no longer available to new customers since Nov 7, 2025 (Snowcone retired Nov 2024). Alternatives: DataSync (online transfer), AWS Data Transfer Terminal (physical transfer), Outposts (edge compute). Older exam questions may still point to Snowball Edge for disconnected edge compute.
