# DX - Direct Connect

- It is a physical entry point into the AWS network
- It is a physical fibre connection (1, 10, 100 or 400 Gbps dedicated) through which we can access AWS services without sending traffic through the public internet
- The connection is between the business premises => DX Location => AWS Region
- When we order a DX connection, what we order is actually a Port Allocation at a DX Location
- The port has 2 costs:
  - Hourly cost based on the DX location and speed of the port
  - Outbound data transfer, inbound data transfer is free of charge
- In order to get a Direct Connect we have to create a connection (logical entity) inside an AWS account. This connection will have an unique ID
- Provisioning time: weeks/months of extra time
- DX provides low and consistent latency + high speeds (1/10/100/400 Gbps)
- In the DX location we will have to install a cross-connect (physical cable) in order to connect our own network to the AWS network
- DX has connectivity to AWS Private Services (VPCs) and Public Services but NOT to the Internet.
- DX is **NOT encrypted** by default => MACsec (L2) or IPsec VPN on top (public VIF or private IP VPN over transit VIF)
- Data transfer out over DX is charged at a lower rate than over the internet => DX can also be a cost optimization for large egress volumes

![DX Concepts](images/DXConcepts.png)

## DX Physical Connection Architecture

- A Direct Connect is a physical port allocated at a DX Location
- This physical port provides 1, 10, 100 or 400 Gbps speed (400 Gbps at select locations)
- DX can be ordered directly from AWS or through partners (wider range of speeds, with less options)
- **Dedicated connection**: physical port dedicated to us, 1/10/100/400 Gbps, ordered from AWS (console), supports up to 51 VIFs (50 public/private + up to 4 transit within the 51), LAG, MACsec
- **Hosted connection**: provisioned by an AWS Direct Connect Partner on their port, 50 Mbps up to 25 Gbps, **only 1 VIF per hosted connection** (exam trap: need several VIFs => multiple hosted connections or a dedicated connection). Capacity changes may require a new connection
- **Hosted VIF**: a VIF created by another account (e.g. partner or network account) and accepted in our account
  Things that are needed to be used on the router on our side in order to work with DX:
- The DX port requires single-mode fibre, NO copper cable connection supported
- Depending on the speed we order we will interface with DX using the following standards:
  - **1000BASE-LX** (1310 nm) Transreceiver for 1 Gbps
  - **10GBASE-LR** (1310 nm) Transreceiver for 10 Gbps
  - **100GBASE-LR4** - 100 Gbps
  - **400GBASE-LR4** - 400 Gbps
- Auto-Negotiation should be disabled for the connection
- We configure the port speed, full-duplex should be manually set on the network connection
- Our router within the DX location should support Border Gateway Protocol (BGP) and BGP MD5 Authentication
- Optional configurations:
  - MACsec
  - Bidirectional Forwarding Detection (BFD) - asynchronous BFD is enabled by default on AWS side, we need to enable it on our router for fast failover

## Direct Connect - MACsec

- It is a security feature that improves/partially improves a long-standing problem with DX: lack of built-in encryption
- It is a standard which allows frames on the network to be encrypted. Frames are the unit of data which occur at the layer 2 of the OSI model
- MACsec provides a hop by hop (not end-to-end) encryption architecture => 2 devices need to be next to each other at layer 2 to order MACsec (layer 2 adjacency)
- MACsec features:
  - Confidentiality: strong encryption at layer 2 by encrypting the frame's EtherType and payload
  - Data integrity: adds additional fields to ensure that data cannot be modified in transit without both parties being able to detect the modification
  - Data origin authenticity: both parties can see that frames were being sent by other trusted peers
  - Replay protection
- MACsec does not replaces IPSEC over DX, it is not end-to-end!
- It is designed to allow transfer for super high speeds networks
- MACsec key components:
  - **Secure Channel** (unidirectional): each MACsec participant creates a MACsec channel that is used to send traffic
  - **Secure Channels** are assigned an identifier (SCI): uniquely identifies a secure channel
  - **Secure Associations**: communication that occurs on each secure channel, takes place as a series of transient sessions, multiple secure associations will take place on each secure channel over the lifetime of the connection. Each secure channel generally has 1 secure association at a time (exception when the associations are being replaced)
  - **MACsec encapsulation**: MACSec modifies Ethernet frames by inserting two tags: 16 bytes MACsec tag & 16 bytes of Integrity Check Value (ICV).
  - **MACSec Key Agreement** protocol: manages discovery, authentication and key generation
  - **Cipher Suite**: controls how the data is encrypted: algorithm, packets per key, key rotation
- MACsec is supported on **dedicated** connections of **10, 100 and 400 Gbps** at select DX locations (not on 1 Gbps or hosted connections). We may need to order a new MACsec-capable port
- MACsec can be defined either on a DX connection or on a Link Aggregation Group (LAG)
- Encryption mode: `should_encrypt` / `must_encrypt` / `no_encrypt`
- Configuring MACsec: we associate a CAK/CKN pair with the connection on both the AWS DX router(s) and customer side's router
- It is possible to extend MACsec from the DX location ot the customer side
- This requires a dedicated physical extension of the cross connect to the business premises; this type of extension requires Layer 2 Adjacency

MACsec is way faster, but not end-to-end like IPSec (VPN).

![MACsec](images/MACsec.png)

## DX Connection Process

- A DX connection begins in a DX Location, which contains AWS equipment and also customer/provider equipment
- AWS does not own this facility (neither does the provider) - it is a data center owned by a third party
- A large data center is collection of cages, these cages are areas that specific customers rent
- Only the staff at the data center can connect stuff together, only they have access to the space in-between the cages to connect different cages together
- Connecting these cages can be done only by staff members only when they have authorization from all parties
- The authorization is called **Letter of Authorization Customer Facility Access (LOA-CFA)**
- It is form that gives the access from one customer to get the data center staff to connect to the equipment of another customer
- DX connection process:
  ![DX Connection Process](images/DXConnectionProcess.png)

## DX Virtual Interfaces BGP Sessions + VLAN

- DX Connections are a layer 1 connections (physical cables) running layer 2 protocols (Data Link)
- We need a way to connect to multiple types of layer 3 (IP) networks (VPCs and public zones) over a single DX connection => this is where Virtual Interfaces (VIFs) come in handy
- VIFs allows us to run multiple layer 3 networks over a layer 2 direct connect
- A VIF is simply a BGP Peering Session => something which exchanges prefixes which allow traffic to be router to one point to the other
- All of this is isolated within a VLAN
- A VLAN isolates different layer 3 networks using VLAN tagging
- BGP exchanges prefixes, which means each end knows which networks are at each side; BPG MD5 authentication means that only authenticated data will accepted by either type
- 3 types of VIFs can be run over DX:
  - Private VIF: used to connect ot AWS private networks, so VPCs
  - Public VIF: used to connect to Public Zone Service
  - Transit VIF: allow integration between DX and Transit Gateways
    ![DX VIFs - BGP Session + VLAN](images/DXBGPSessonVLAN.png)
- A single dedicated DX can have in total 50 public/private VIFs, plus transit VIFs: max **4 transit VIFs**, all within a hard limit of **51 VIFs** per connection (originally only 1 transit VIF)
- A hosted connection supports only 1 VIF (of any type)
- Hosted VIFs: we can share VIFs with other AWS accounts. Can be connected to a virtual private gateway in a VPC of the other account

## Private VIFs

- They are used to access the resources inside 1 AWS VPC using private IP addresses
- Resources can be accessed with their private IP using private VIFs, public IPs and Elastic IPs wont work
- Private VIFs are associated with a Virtual Private Gateway (VGW) which can be associated to 1 VPC. This has to be in the same region where the DX location connection terminates
- 1 Private VIF = 1 VGW = 1 VPC (there are ways around this using Transit VIFs)
- There is no encryption on private VIFs, DX is not adding encryption and neither is the private VIFs (there are ways around this, example using HTTPS)
- With private VIFs we can use normal or Jumbo Frames (MTU of 1500 or 9001)
- Using VGW, route propagation is enabled by default
- Creating private VIFs:
  - Pick the connection the VIF will run over
  - Chose VGW (default) or Direct Connect Gateway
  - Chose who owns the interface (this account or another account)
  - Choose a VLAN id (= 802.1Q part of the Ethernet Frame) which needs to match the customer config
  - We need to enter the BGP ASN of on-premises (public or private). If private use 64512 to 65534 (or long ASN 4200000000 to 4294967294)
  - We can choose IPs or auto generate them
  - AWS will advertise the VPC CIDR range and the BGP Peer IPs (`/30`)
  - We can advertise default or specific corporate prefixes (**max 100** per address family by default; if exceeded the BGP session goes into an idle state. Can be raised up to 1,000 with DX prefix controls - otherwise summarize or advertise a default route)
- Private VIFs architecture:
  ![Private VIFs Architecture](images/DXPrivateVIFS.png)
- Key learning objectives:
  - Private VIFs are used to access private AWS services
  - Private VIF => 1 VGW => 1 VPC
  - VPC needs to be in the **same region** as the DX location
  - VGW has an AWS ASN assigned - or we can configure one
  - Over the private VIF runs the BGP with IPv4 or IPv6 (separate BPG peering connections)
  - We configure our own ASN on the VIF, which can be private ASN or public ASN

## Public VIFs

- Are used to access AWS public zone services: both public services and services which have a public Elastic IP
- They offer no direct access to private VPC services
- We can access **all public zone regions** with one public VIF across AWS global network as opposed to Private VIFs
- AWS advertises all AWS public IP ranges to us, all traffic to AWS services will go over the AWS global network
- We can advertise any public IPs we own over BGP, in case we don't have public IPs, we can work with AWS support to allocate some to us
- Public VIFs support bi-directional BGP communities
- Advertised prefixes are not transitive (can't access the ones from different accounts), our prefixes don't leave AWS
- **NOT going over the internet**
- Create a public VIF:
  - We pick the connection the VIF will run over
  - We chose the interface owner (this account or another)
  - Chose VLAN - 802.1Q, which needs to match the customer configuration
  - Chose the customer side BGP ASN (ideally this is public ANS for full functionality offered by public VIFs)
  - Configure MD5 authentication and select optional peering IP addresses
  - We have to select which prefixes we want to advertise
- Public VIFs architecture:
  ![Public VIFs Architecture](images/DXPublicVIFS.png)

## Direct Connect Public VIF + VPN

- Using a VPN gives us an encrypted and authenticated tunnel
- Architecturally, having a VPN over DX uses a Public VIF + VGW/TGW public endpoints
- With a VPN we connect to public IPs which belong to a VGW or TGW
- A VPN is transit agnostic: we can connect using a VPN to VGW or a TGW over the internet or over DX
- A VPN is end-to-end encryption between a Customer Gateway (CGW) and TGW/VGW, while MACsec is single hop based
- VPNs have wide vendor support
- VPNs have more cryptographic overhead compared to MACsec
- A VPN can be provided immediately, can be used while DX is provisioned and/or as a DX backup
- Using a Public VIF, it's possible to target multiple AWS regions using a single VIF and multiple VPN tunnels, when with Private VIFs, it is bound to a single region.
  ![VPN over DX](images/DXPublicVIFVPN.png)

## Direct Connect Gateways

- Direct Connect is a regional service
- Once a DX connection is up, we can use public VIFs to access all AWS Public Services in all AWS regions
- Private VIFs can only access VPCs in the same AWS regions via VGWs
- Direct Connect Gateway is a global network device: it is accessible in all regions
- We integrate with it on the on-premises side by creating a private VIF and associate this with a DX Gateway instead of the Virtual Private Gateway (VGW). This integrates the on-premises router with the DX Gateway
- On the AWS side we create VGW associations in any VPC in any AWS regions
- DX gateways allow to route through them to the on-premises environments and vice-versa. They don't allow VPCs connected to the gateway to communicate with each other
- We can have **20 VGW associations per DX Gateway** (was 10), and up to 30 private/transit VIFs per DX Gateway
- 1 DX connection can have up to 50 private VIFs, each of which support 1 DX gateway and 1 DX gateway supports 20 VGW associations => we can connect up to 1,000 VPCs per connection
- DX Gateways are free, global, 200 per account; the DXGW must have an ASN different from the on-premises ASN
- **Allowed prefixes**: on each DXGW association we define which prefixes are advertised to on-premises (for VGW associations defaults to the VPC CIDR; for TGW associations we must list them explicitly, max 200 per TGW)
- No overlapping CIDRs among VPCs associated to the same DXGW
- DX gateway don't have a cost, we have cost for data transit only
  ![DX Gateway Architecture](images/DirectConnectGateway3.png)
- Cross-account DX Gateways: multiple account can create association proposal for a DX gateway; the DXGW owner accepts it (and can override allowed prefixes)
- **SiteLink** (2021): enable on private/transit VIFs to send traffic **between on-premises sites** over the AWS backbone through the DXGW (bypassing AWS Regions). Charged per hour per VIF + data transfer. Use case: replace a private WAN between branches/data centers that already have DX
- DXGW can also attach to an **AWS Cloud WAN** core network (2024) via transit VIFs (up to 5,000 prefixes advertised to on-premises)

## Transit VIFs and TGW

- A DX Gateway does not route between the associated VPCs to that gateway, it only routes from on-premises to AWS side or vice-versa
- Transit Gateways are regional, it is possible to peer TGWs allowing connections between regions
- Transit Gateways are hub-and-spoke architecture, anything associated with a TGW is able to communicate with anything other associated to that TGW
- This architecture also works within peered TGWs
- DX-TGW Architecture:
  ![DX-TGW Architecture](images/DXGateway4.png)
- A DX supports up to 50 public and private VIFs and up to 4 Transit VIFs (51 VIFs total)
- We can connect up to 6 Transit Gateways to a Direct Connect Gateway (was 3)
- An individual DX gateway can be used with VPCs and private VIFs or with Transit Gateways and transit VIFs, **NOT BOTH** at the same time!
- Consideration:
  - DX gateway does not route between its attachments, this is why the peering connection between TGWs is required
  - Each TGW can be attached up to 20 DX gateways
  - Each TGW supports up 5000 attachments, up to 50 peering attachments
- DX Gateway routing problems:
  - DX gateway only allows communications from a private VIF to any associated virtual private gateways
  - With a transit gateway we can solve this, if we connect the DX gateway to a transit gateway (works only in one region)

## Direct Connect Resilience and HA

- To improve resilience:
  - Order 2 DX ports instead of one => 2 cross connects, 2 customer DX routes connecting to 2 on-premises routes
  - Connect to 2 DX locations, have to customer routers and 2 on-premises routers in different buildings (geographically separated)
- Not resilient DX architecture:
  ![DX resilience NONE](images/DirectConnectResilience1.png)
- Resilient DX architecture:
  ![DX resilience OK](images/DirectConnectResilience2.png)
- Improved resilient DX architecture:
  ![DX resilience BETTER](images/DirectConnectResilience3.png)
- Extreme resilient DX architecture:
  ![DX resilience GREAT](images/DirectConnectResilience4.png)

## Direct Connect Link Aggregation Groups (LAG)

- LAG: allows to take multiple physical connections and configure them to act as one by using the LACP (Link Aggregation Control Protocol)
- From speed perspective a LAG can have:
  - 2 ports, each 100 Gbps or 400 Gbps
  - 4 ports, the speed of each being less than 100 Gbps (1 or 10 Gbps)
- We can create a LAG with the maximum speed of 800 Gbps (2 x 400 Gbps), 200 Gbps with 100 Gbps ports
- LAG do provide resilience, although AWS does not market them as such. They do not provide any resilience regarding hardware failure or the failure of entire location
- LAGs use an Active/Active architecture, maximum 4 connection can be part of the LAG
- All connections must have the same speed and terminate at the same DX location
- A LAG has an attribute called `minimumLinks`: the LAG is active as long as the number of working connections is greater or equal to this value
  ![DX LAG](images/DirectConnectLAG.png)
- LAGs are dedicated connections only (not hosted); we can have 10 LAGs per Region

## Direct Connect Resiliency Toolkit and SLA

- The **DX Resiliency Toolkit** (console wizard) offers models:
  - **Maximum Resiliency**: 2 connections at each of 2 DX locations (4 connections, separate devices) - SLA **99.99%**. For critical workloads
  - **High Resiliency**: 1 connection at each of 2 DX locations - SLA **99.9%**
  - **Development and Test**: 2 connections on separate devices at 1 location - no SLA for location failure
- **Failover testing**: we can bring down the BGP sessions of a VIF for a defined time to validate failover
- Cheaper backup option: DX primary + Site-to-Site VPN backup (VPN routes are less preferred than DX routes for the same prefix). Watch the bandwidth difference: a 10 Gbps DX can't fail over to a 1.25 Gbps VPN tunnel without degradation

## Routing and BGP over DX (summary, see bgp.md)

- Active/passive from AWS => on-premises: more specific prefixes on the primary, or local preference communities `7224:7300` (high) / `7224:7100` (low) on private/transit VIFs. Local pref is evaluated before AS_PATH
- AS_PATH prepending also works (lower priority than local preference)
- Public VIF scope communities: `7224:9100` local Region, `7224:9200` continent, `7224:9300` global (default)
- DX prefers DX locations whose **associated Region** is the destination Region by default
- Public VIF: we must own the public prefixes (RIR registered); AWS filters traffic whose source is not from our advertised prefixes

## Key DX Quotas (exam numbers)

| Item | Limit |
|---|---|
| Private/public VIFs per dedicated connection | 50 (hard) |
| Transit VIFs per dedicated connection | 4 (within 51 total) |
| VIFs per hosted connection | 1 |
| Prefixes from on-premises per private/transit VIF BGP session | 100 (up to 1,000 with prefix controls) |
| Prefixes from on-premises per public VIF | 1,000 |
| VGWs per DX gateway | 20 |
| TGWs per DX gateway | 6 |
| DX gateways per TGW | 20 |
| VIFs per DX gateway | 30 |
| Prefixes from TGW to on-premises per transit VIF | 200 |
| Connections per LAG | 4 (<100G) / 2 (100G, 400G) |

## Exam Tips

- "Need connectivity next week" => VPN now, DX later (DX takes weeks/months)
- "Access VPCs in multiple Regions over one DX" => DX Gateway (private VIF) or DXGW + TGW (transit VIF)
- "Access S3 / public AWS endpoints over DX" => public VIF (or private VIF/transit VIF + S3 interface endpoint)
- "Encrypt DX at line rate 10/100 Gbps" => MACsec; "end-to-end encryption" => IPsec VPN over DX
- "VPC to VPC through DXGW" => not supported, use TGW/peering
- "Branch-to-branch over AWS backbone using DX" => SiteLink
