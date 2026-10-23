# AWS Transit Gateway

- It is a network transit hub which connects VPCs to each other and to on-premise networks using Site-to-Site VPNs and Direct Connects
- It is designed to reduce the network architecture complexity in AWS
- It is a network gateway object, it is HA and scalable
- Attachments: we create attachments in order for the TGW to connect to VPCs and on-premise networks. Valid attachments are:
    - VPC attachments
    - Site-to-Site VPN attachments (incl. VPN Concentrator attachments for many small sites, 2025)
    - Direct Connect Gateway attachments (via Transit VIF)
    - Transit Gateway Connect attachments (GRE + BGP to SD-WAN/virtual appliances)
    - Peering attachments (TGW-TGW intra/inter-Region, or TGW-Cloud WAN)
    - Network function attachments (native AWS Network Firewall integration, 2025)
    - Client VPN attachments (native Client VPN integration, 2026)
- VPC attachment: we pick **exactly one subnet per AZ** (TGW places an ENI there); traffic can then reach all subnets in that AZ. Resources in an AZ without an attachment subnet cannot use the TGW
- Best practice: use small dedicated /28 subnets for TGW attachments, with their own route table and permissive NACL
- A TGW cannot have 2 attachments to the same VPC; a VPC can be attached to max 5 TGWs
- We can also peer transit gateways across cross regions and/or cross accounts
- We can also attach transit gateways to the DX connections
- Transit Gateway Considerations:
    - Supports transitive routing: single transit gateway with multiple attachments using route tables
    - Can be used to create global networks with peering
    - We can share transit gateways using AWS RAM
    - Transit Gateways offer less complex architectures compared to VPC peering solutions

## Transit Gateway - Deep Dive

- Transit gateway is a hub-and-spoke architecture, it can connect to various types of networking objects within AWS
- Integration with Direct Connect:
    - A Transit VIF is required which goes through a DX Gateway
    - The DX Gateway can be attached to the Transit Gateway with a Transit Gateway Attachment
    - 1 DX Gateway can be associated with up to 6 Transit Gateways (was 3); 1 TGW supports up to 20 DX Gateways
    - Allowed prefixes on the DXGW-TGW association define what is advertised to on-premises (VPC CIDRs are NOT advertised automatically, max 200 prefixes per TGW to on-premises)
- Transit Gateway has a default route table which is populated from the attachments:
    - For the VPCs we have the CIDR ranges of these VPCs
    - For VPNs we have the routes learned via BGP
    - For DX Gateways with the Transit Gateway Attachment we define the networks within the attachment configured at the DX Gateway side
- We can peer TGWs with other TGWs within the same Region or between Regions. We can have up to 50 peering attachments per TGW (adjustable), only 1 peering between the same pair of TGWs. Peering is NOT transitive: routing across chains of peers needs static routes on every hop
- A TGW by default has one route table
- All attachments use this RT for routing decisions, by default all attachments propagate routes to this route table, exception peering attachments
- All attachments can route to all other attachments by default

## Transit Gateway Peering

- In case of peering attachments routes are not shared, we need to use static routes, similar to VPC peering (AWS recommends using unique ASNs for future enhancements for route advertisements). Need dynamic routing between Regions => Cloud WAN
- Resolution of public DNS to private addressing is not supported over inter-region peers
- Data transfer over peering connection is encrypted and is sent over AWS network
- We can peer up to 50 peering attachments per TGW, these can be in different regions, different AWS accounts

## Transit Gateway Isolated Routing

- By default:
    - All attachments are associated with the same route table
    - All attachments propagate to the same route table, all attachments are aware of any other attachments
- Attachments can only be associated with 1 route table, route tables can be associated to many attachments
- Attachments can propagate to many RTs, event to those they are not associated with
- If we would want to isolate networks:
    - We create a route table and we configure all attachments to propagate to the route table
    - We associate the route table with only the attachments we would want to communicate with each other
    - We create another route and associate it to the attachment we don't want to communicate with each other. We configure other attachments to propagate to this route table
- **Blackhole routes**: static routes that drop matching traffic (e.g. block spoke-to-spoke for a CIDR)
- Supports referencing **customer-managed prefix lists** in TGW route tables

## Key Limits and Numbers

- 5,000 attachments per TGW, 20 route tables per TGW (adjustable), 10,000 routes total per TGW
- Bandwidth: up to **100 Gbps per VPC attachment per AZ** (each direction), same for DXGW and peering attachments; 7.5M pps per attachment per AZ
- VPN attachment: 1.25 Gbps per standard tunnel (5 Gbps with large bandwidth tunnels) - use **ECMP** across multiple VPN tunnels/connections (BGP only) to scale
- **MTU 8500** for VPC, DX, Connect and peering attachments; **1500** for VPN. Migrating from VPC peering (9001) can drop jumbo packets
- Routes from on-premises: 1,000 per VPN on TGW (vs 100 on VGW)
- Pricing: per attachment-hour + per GB processed (data processing charge applies to traffic sent to the TGW). VPC peering has no processing charge => cheaper for high-volume VPC-to-VPC flows

## Route Evaluation Order (same prefix, different attachment types)

1. Longest prefix match always first
2. Static routes (incl. VPN static)
3. Prefix list referenced routes
4. VPC propagated
5. DX gateway propagated
6. Connect propagated
7. VPN over private DX propagated
8. VPN propagated (then VPN Concentrator, Client VPN)
9. TGW peering propagated (Cloud WAN)
- Same attachment type: shorter AS_PATH, then lower MED
- ECMP is not supported between different attachment types (e.g. DX vs VPN). Supported for VPN (if enabled, BGP), Connect and DXGW attachments with identical prefix/AS_PATH

## Appliance Mode

- Problem: TGW keeps traffic in the source AZ by default. With a centralized inspection VPC (stateful firewalls in multiple AZs), request and response can hit appliances in different AZs => asymmetric routing => dropped flows
- Enable **appliance mode** on the inspection VPC attachment: TGW uses a flow hash to pick one AZ/ENI for the life of the flow, for both directions (symmetric)
- Exam trigger: "centralized inspection VPC with TGW, traffic intermittently dropped between AZs" => enable appliance mode
- Alternatives: Gateway Load Balancer (GWLB) endpoints in the inspection VPC; or native **Network Firewall attachment** to TGW (2025) - AWS manages the inspection VPC, appliance mode enabled automatically, static routing only

## Transit Gateway Connect

- Attachment type that runs on top of a VPC or DX attachment (transport)
- Uses **GRE tunnels + BGP** to third-party SD-WAN/virtual appliances (no IPsec overhead)
- Up to 4 Connect peers (GRE tunnels) per Connect attachment, **5 Gbps per peer** => 20 Gbps per Connect attachment; ECMP across peers/attachments
- Requires a TGW CIDR block for the TGW-side GRE addresses

## Multicast

- TGW is the only native AWS way to do IP **multicast** between VPCs (VPC itself does not support multicast/broadcast)
- Multicast domains, group members/sources registered statically or via IGMPv2
- Limits: 1 Gbps per flow, 20 Gbps aggregate per AZ - not for HFT

## Other Features

- **Security group referencing** across VPCs attached to the same TGW (same Region, inbound rules only; GA Sep 2024). Must be enabled on the TGW and on each VPC attachment
- **TGW Flow Logs**: capture traffic per attachment (like VPC Flow Logs) to CloudWatch Logs/S3/Firehose; useful for centralized visibility and chargeback
- **AWS Network Manager**: global view, topology, route analyzer, events for TGWs and on-premises sites
- **Inter-Region peering** traffic is encrypted and stays on the AWS backbone
- **Sharing via RAM**: share a TGW from a network account to the Organization; spoke accounts create attachments (auto-accept optional). The owner controls the route tables
- **Encryption**: TGW supports VPC Encryption Controls (enforce encryption in transit for attached VPCs)

## Transit Gateway vs VPC Peering vs Cloud WAN vs PrivateLink

- **VPC Peering**: 1:1, non-transitive, no bandwidth limit, no processing charge, lowest latency. Best for few VPCs or very high-volume pairs. 50 active peerings per VPC by default (up to 125); no overlapping CIDRs
- **Transit Gateway**: regional hub, transitive, thousands of VPCs + VPN + DX, route table segmentation. Inter-Region with static peering
- **Cloud WAN**: global, policy-driven WAN built on TGW technology. Use for multi-Region with dynamic routing and segmentation managed centrally
- **PrivateLink**: expose a single service (not a network), works with overlapping CIDRs, unidirectional consumer => provider

## AWS Cloud WAN

- Managed global network: we define a **core network policy** (JSON) with **segments** (e.g. prod, dev, shared) and attachment rules (tag-based), AWS deploys **Core Network Edges (CNEs)** in the chosen Regions
- Routing between CNEs/Regions is **dynamic (BGP, automatic)** - main difference vs TGW peering (static)
- Attachments: VPC, Site-to-Site VPN, Connect (SD-WAN), TGW route table (peering with existing TGWs), **Direct Connect gateway** (native, 2024)
- Segment sharing and **service insertion** (send traffic between segments through a firewall/inspection network function group)
- Central visibility through Network Manager
- Pricing: per CNE-hour + per attachment-hour + data processing
- When to use: many Regions, need for centralized policy and segmentation, global WAN replacement/SD-WAN integration. A single Region with a few dozen VPCs => TGW is simpler and cheaper
