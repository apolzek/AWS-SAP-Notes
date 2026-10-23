# VPC - Virtual Private Cloud

## Public vs Private Services

- Public service: a service which is accessed by using public endpoints
- Private service: a service which runs inside a VPC
- Either private or public, every service can have permissions in order to be accessible
- VPC: private network isolated from the internet. Can't communicate to the network unless we are allowing it. Nothing from the internet can reach the services from a VPC as long as we do not configure it otherwise
- Internet Gateway: we can connect it to a VPC, this will allow the services in the VPC to communicate with the public internet

## DHCP in a VPC

- DHCP - Dynamic Host Configuration Protocol: offers auto configuration for network resources
- Every device has a hard-coded MAC address (Layer 2 address)
- DHCP begins with a L2 broadcast to discover a DHCP server on the local network
- Once discovered a DHCP server and a DHCP clients communicate, meaning that the client will get in the end an IP address, a Subnet Mask and Default Gateway address (L3 configuration)
- DHCP also configures which DNS server should a resource use in a VPC
- Also configures NTP servers, NetBios Name Servers and Node types
- When we are setting which DNS service to use in a VPC we can either explicitly provide values or we can set `AmazonProvidedDNS`
- We also get allocated 1 or 2 DNS names for the services in the VPC. One can be public if the instance has a public IP address allocated
- Custom DNS names: we can give custom DNS names to EC2 instances if we use our own custom DNS servers. To accomplish these we can use DHCP option sets
- DHCP options sets:
    - Once created option sets can not be changed
    - Can be associated with 0 or more VPCs
    - Each VPC can have a max of 1 option set associated (it can have 0)
    - We we change a DHCP option set associated to the VPC, the change is immediate, but any new setting will only affect anything once a DHCP renew occurs
    - What we can configure in an option set:
        - DNS server (Route 53 resolver) what we can use in the VPC
        - NTP server

## VPC Router Deep Dive

- Is at the core of any network which involves AWS
- Is a virtual router in a VPC
- It is HA across al AZs in a region, no management overhead is required
- It is scalable, no management overhead required
- VPC routes routes traffic between subnets in a VPC
- Routes traffic from external network into the and vice-versa
- VPC router has an interface in every subnet in a VPC: `subnet+1` address (Default Gateway), the first IP address in each subnet after the network address itself
- We control how the VPC routes traffic using Route Tables

## VPC Route Tables

- Every VPC is created with a main Route Table (RT), which is the default for every VPC
- Custom route tables can be created for each subnet
- Subnets can be associated with only one RT which can be the main one or custom
- If we disassociate a custom RT form a subnet, the main RT will be attached to it
- Main RT should not be changed, custom RT should be used for any routing changes
- RT have routes, routes have an order, the most specific route wins
- Edge Association: a RT tables is associated with network gateway
- All RTs have at least one route: the local route which matches the VPC cidr range. These routes are un-editable

## NACL - Network Access Control Lists

- A NACL can be considered to be a traditional firewall in an AWS VPC
- NACLs are associated with subnets, every subnet has a NACL associated to it
- Connection inside a subnet are not affected by NACLs
- NACls can be considered stateless firewalls, so we can talk about the following type of rules:
    - Inbound rules: affect data coming into the subnet
    - Outbound rules: affects data leaving from the subnet
- Rules can explicitly **ALLOW** and explicitly **DENY** traffic
- Rules are processed in order:
    1. A NACL determines if a the inbound or outbound rules apply
    2. It starts from the lower rule number, evaluates traffic against each rule until is a match (based on IP range, port, protocol)
    3. Traffic is allowed/denied based on the rule
- Last rule is an implicit deny in every NACL, if no rule before that applies, traffic will be denied
- Default NACL: when a VPC is created, a default NACL is attached to it. The default NACL is allowing all traffic
- Custom NACLs: 
    - We can create them and attach them to subnets
    - Each NACL has a default rule that denies all traffic. This has the lowest priority
- NACLs can be associated to many different subnet, however each subnet can have only one NACL associated to it at any time
- NACL are not aware af any logical resources within a VPC, they are aware of IPs, CIDRs and protocols

## SG - Security Groups

- Security Groups are stateful firewalls, meaning they detect response traffic to a request and they automatically allow that traffic
- SGs do not have explicit **DENY** rules, they can not be used to block bad actors (use NACLs for this)
- SGs support IP/CIDR rules and also allow to reference logical resources
- SGs are attached to Elastic Network Interfaces (ENI), when we attach a SG to an EC2, the SG will be attached to the primary ENI
- SGs are capable to reference logical resources, ex. other security groups or self referencing
- SG referencing across VPCs: works over **VPC peering (same Region)** and **Transit Gateway (same Region, inbound rules, Sep 2024)**; also for shared VPC participants
- SGs can reference **managed prefix lists** (AWS-managed, e.g. CloudFront origin-facing, or customer-managed)
- Limits: 60 inbound + 60 outbound rules per SG (adjustable), 5 SGs per ENI (up to 16), rules x SGs per ENI <= 1,000
- NACL limits: 20 rules each direction (up to 40). NACLs only see IPs => remember ephemeral ports (1024-65535) for return traffic
- NACLs don't filter traffic to the VPC Resolver (.2), instance metadata (169.254.169.254), Time Sync, Windows license activation

## VPC Sizing and Addressing

- VPC IPv4 CIDR: between `/16` and `/28`; up to 5 CIDRs per VPC by default (primary + secondary, up to 50)
- **5 reserved IPs per subnet**: network address, +1 VPC router, +2 DNS, +3 reserved for future use, broadcast (last)
- Secondary CIDRs can be added later (e.g. from `100.64.0.0/10` for EKS pods) but cannot overlap; CIDR of an existing subnet cannot be changed
- Plan non-overlapping ranges across accounts/Regions/on-premises => required for peering, TGW, DX, VPN

## Public IPv4 Pricing (Feb 2024)

- Since **Feb 1, 2024** every public IPv4 address is charged **$0.005/hour** (~$3.6/month), whether in use or idle: EIPs (attached or not), auto-assigned public IPs, NAT gateway, ALB/NLB nodes, etc.
- BYOIP addresses are not charged
- **Public IP Insights** (VPC IPAM, free tier) shows public IPv4 usage
- Cost optimizations: IPv6 (dual-stack/IPv6-only), private subnets + NAT/ALB, EC2 Instance Connect Endpoint instead of public bastions, VPC endpoints

## Amazon VPC IP Address Manager (IPAM)

- Plan, allocate, track and audit IP addresses across accounts and Regions (integrates with **AWS Organizations**, delegated admin)
- **Pools** hierarchy (top-level => Regional => per environment/account) with allocation rules (min/max/default netmask, required tags); share pools with RAM
- VPCs can be created from an IPAM pool (`--ipv4-ipam-pool-id`) => guaranteed non-overlapping CIDRs
- Monitors compliance/overlap, historical IP usage, BYOIP (IPv4/IPv6) management, Amazon-provided contiguous IPv6 blocks
- Tiers: **Free tier** (Public IP Insights, basic) and **Advanced tier** (paid, per active IP)
- Exam trigger: "prevent overlapping CIDRs across hundreds of accounts" => IPAM pools + SCP/IAM condition requiring IPAM

## VPC Block Public Access (Nov 2024)

- Account/Region-level setting that blocks internet traffic through IGWs and egress-only IGWs for all VPCs, overriding route tables/SGs
- Modes: **bidirectional** (block in and out) or **ingress-only** (allow outbound via NAT/EIGW)
- **Exclusions** per VPC or subnet (bidirectional or egress-only), 50 per account per Region
- Centrally enforced with **AWS Organizations declarative policies**
- Use Network Access Analyzer / Reachability Analyzer to assess impact before enabling

## VPC Encryption Controls (2025)

- Audit (**monitor mode**) and enforce (**enforce mode**) **encryption in transit** for all traffic within and across VPCs in a Region
- AWS uses Nitro hardware encryption between instances; enforce mode blocks creation of resources that cannot encrypt (non-Nitro instances, etc.)
- Existing VPCs: monitor mode first (flow logs show encryption status), fix non-compliant resources, then enforce. New VPCs can start in enforce mode
- Some gateways can be excluded (IGW, NAT GW, VGW, etc.); TGW supports it for attached VPCs
- Pricing: free during preview, **charged per non-empty VPC-hour since Mar 1, 2026**
- Exam trigger: "prove/enforce encryption in transit for compliance (HIPAA, PCI, FedRAMP) inside VPCs without managing TLS everywhere"

## NAT Gateway

- Managed SNAT for IPv4 private subnets; zonal resource => one per AZ for HA (avoid cross-AZ data and AZ dependency)
- Scales 5 Gbps => 100 Gbps; up to 8 IPs per NAT GW (each IP: 55,000 simultaneous connections per unique destination)
- Public NAT GW (needs EIP, in public subnet) vs **Private NAT GW** (no internet; SNAT for overlapping/non-routable CIDRs towards TGW/VGW)
- **Regional NAT gateway** (Nov 2025): one NAT gateway for the whole VPC that automatically expands/contracts across AZs where we have workloads; no public subnet needed; automatic or manual IP management; simpler route tables
- Pricing: per hour + **per GB processed** => use gateway endpoints for S3/DynamoDB to avoid NAT processing charges
- NAT64 + DNS64 let IPv6-only workloads reach IPv4 destinations

## VPC Peering

- Private 1:1 connection between 2 VPCs (same/different account, same/different Region), on AWS backbone, encrypted inter-Region
- **Not transitive**: A-B and B-C does not give A-C; no edge-to-edge routing (cannot use peer's IGW, NAT GW, VGW/DX/VPN, gateway endpoints)
- No overlapping CIDRs; routes must be added on both sides; 50 active peerings per VPC (up to 125)
- No bandwidth limit, no data processing charge (only data transfer; free within the same AZ) => cheapest for high-volume pairs
- Same Region: SG referencing supported; jumbo frames (9001) supported intra-Region
- DNS resolution of peer's public hostnames to private IPs must be enabled on the peering

## VPC Sharing (RAM)

- Owner account shares **subnets** with participant accounts in the same Organization; participants launch resources into shared subnets
- Owner manages VPC, subnets, route tables, NACLs, gateways; participants manage their own resources and SGs
- Fewer VPCs, fewer TGW attachments, lower cost; billing stays with participants for their resources
- Limit: 100 participant accounts per VPC

## VPC Observability and Troubleshooting

- **VPC Flow Logs**: VPC/subnet/ENI (also TGW) => CloudWatch Logs, S3, Firehose. Metadata only (no payload), not real time. Custom format fields (e.g. `pkt-srcaddr`, `tcp-flags`, encryption status)
- **Traffic Mirroring**: copy actual packets from ENIs to an NLB/GWLB/ENI target for IDS/deep inspection
- **Reachability Analyzer**: static config analysis "can A reach B?" (no packets sent)
- **Network Access Analyzer**: find unintended network access paths against requirements (e.g. "no internet path to DB subnets")
- **VPC Route Server** (2025): BGP peering from appliances in the VPC to dynamically update VPC route tables (floating routes for active/standby appliances)

## AWS Local Zones

- Parent Region: regular AWS region
- Local Zones are attached to parent regions and they operate in the same geographical region
- Local Zone naming: `us-east-las-1` - < parent region > - < Local Zone identifier (international city code) >. Examples: `us-west-2-lax-1a`, `us-west-2-lax-1b`
- We can have multiple Local Zones in the same city
- Local Zones operate as their own independent points, they have their own independent connection to the internet
- Generally, they support Direct Connect
- A VPC in a parent region can be extended with subnets from a Local Zone. In these subnets we create our resources as normal
- These resources will benefit from super low latencies (in case we want to access them from a business premises nearby)
- Some things within a Local Zone will still utilize the parent region: for example Local Zones will have private networking with the parent region, however if we create backups for an EBS in the Local Zone, this will utilize the S3 from the parent region
- Local Zones can be considered as one additional AZ (but near our location => lower latency), they don't have builtin AZ
- Not all AWS products support Local Zones. From the ones which do support, many of them are opt-in an also many of them have limitations
- Local Zones should be used when we need the highest performance
- Related edge options: **Wavelength Zones** (inside 5G telco networks, carrier gateway), **Outposts** (AWS racks/servers on-premises, needs reliable connection to the parent Region)

## Advanced VPC Routing

- **Subnets are associated with 1 route table (RT) only, no more noe less!**
- This route table is either the main route table from the VPC or a custom route table
- In case of a custom route table association with a subnet, the main route table is disassociated. In case the custom RT is removed, the main RT is associated again with the subnet
- RT can associated with an internet gateway (IGW) or virtual private gateway (VGW)
- IPv4/6 are handled separately within a RT
- Routes send traffic based on a destination to a target
- Route tables have a maximum of 500 non-propagated (static) routes by default (adjustable up to 1,000) and **100 propagated (dynamic) routes (hard limit)** - if on-premises has more prefixes, summarize or advertise a default route
- When a traffic arrives to an interface (IGW, VGW), it is matched to the relevant route table
- All routes from a route table are evaluated - highest-priority matching is used
- Route tables can contain 2 types of routes:
    - Static routes: added manually by us
    - Propagated routes: added when enabled by us on the VPC or on any individual RT
- Evaluation rule for the routes: 
    1. Longest prefix wins, example /32 wins over /24, /16 or /0. More specific routes always win!
    2. Static routes take priority over propagated routes
    3. For any routes learned by propagation:
        1. DX
        2. VPN Static
        3. VPN BGP
        4. AS_PATH (BPG term used to represent the path between two ASNs; it is the distance within two different autonomous systems): routes with a shorter AS_PATH would win over the longer AS_PATH ones

## Ingress Routing

- All outgoing traffic is routed to a security appliances
- The security appliance is sitting in the public subnet which has a RT assigned to it. This RT sends all unmatched traffic out through the IGW and anything for the corporate network through the VGW
- Ingress routing allows to assign route tables to gateways (Gateway route tables). **Gateway route tables** can be attached to internet gateways or virtual gateways and can be used to take action on inbound traffic (route to a security instance for assessment)
![Ingress Routing](images/AdvancedRouting5.png)

## IPv6 Capability in VPCs

- IPv6 addresses are all publicly routable
- NAT is not used for IPv6, IPv6 does not need network address translation simply because of the huge number of available IPv6 addresses
- IPv6 needs to be manually enabled on a VPC. We can either bring our own IP address in a VPC, utilize an AWS provided range, or allocate from IPAM
- In case of AWS provided IPv6 addresses, AWS will allocate an uniq /56 range to the VPC. This range will be entirely uniq and all addresses will be publicly routable
- If we chose to allocate an IP range for a VPC, AWS will use a hex pair to uniquely allocate IP addresses to the subnets
- Routing is handled separately for the IPv6 addresses, we will have IPv4 routes and IPv6 routes
- Egress only internet gateway: similar to NAT gateway, allows outbound traffic denying inbound traffic in case of IPv6 addressing. NAT gateways or instances do not support IPv6!
- Only one internet gateway can be associated with a VPC, but we can have both internet gateway and egress only internet gateway associated to the same VPC. They are 2 different things
![IPv6 Architecture](images/IPv6EOIGW.png)
- IPv6 can be set up while creating a VPC/subnet or we can migrate an existing VPC to IPv6
- We can enable IPv6 on specific subnets only
- We can point IPv6 traffic to internet gateway and egress only internet gateways as well
- Not every service in AWS supports IPv6!
- **IPv6-only subnets** are possible (Nitro instances); use DNS64 + NAT64 on the NAT gateway to reach IPv4-only destinations
- IPv6 is a common answer to public IPv4 cost and IPv4 exhaustion (e.g. large EKS clusters)

## Advanced VPC Structure - How many AZs for HA?

- The number of AZ required for HA:
    - Buffer AZs: number of AZ-failures tolerated (usually 1 for exam questions)
    - Nominal AZs: the number of AZs we can use for normal operations: the number fo AZs available in a region - Buffer AZs (example: 6 AZs available, failure tolerated is 1 AZ => 6 - 1 = 5)
- Sometimes this calculation can influence which region we can use, since number of AZs can differ per region
- Nominal instances: the number instances required for the application for the business load
- Most efficient HA in with optimal costs: Nominal Instances / Nominal AZs => optimal number of instances per AZ

## Advanced VPC Structure - Subnets and Tiers

- Public subnets can be configured to not give public IP addresses to all instances by default. We can explicitly allocate public IP addresses to some resources
- If no public IP is addressed to a resource in a public subnet, it wont be accessible from the outside
- Security groups: we can restrict inbound traffic by allowing traffic from only selected instances
- How many subnets does an app need:
    - We don't need public and private subnets for addressing and security. This can be configured within one subnet. Exception to this: filter traffic using a NACL
    - We need different subnets for different routing
    - Internet-facing load balancers can communicates with private instances. Internet facing load balancer needs to run in a public subnet
    - Number of subnets needed: number of subnets needed for the APP * AZs
    - NAT Gateway: we cannot have the NAT Gateway in the same subnet in which we would want the resources to also use it. Reason: we cannot have 2 default routes in the Route Table
