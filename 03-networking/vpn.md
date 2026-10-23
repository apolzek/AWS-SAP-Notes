# VPNs

# IPSEC VPN Fundamentals

- IPSEC is a group of protocols
- Their aim is to set up secure tunnels across insecure networks. Example: connect two secure networks (peers) over the public internet
- IPSEC provides authentication
- Any traffic transferred through IPSEC is encrypted
- IPSEC is using asymmetric encryption to exchange symmetric keys and use these of ongoing encryption
- IPSEC has 2 main phases:
  - IKE (Internet Key Exchange) Phase 1
    - It is slow and heavy
    - It is a protocol of how keys are exchanged
    - It has 2 versions v1 (older) and v2 (newer)
    - Uses asymmetric encryption to agree on and create a shared symmetric key
    - The end of this phase is an IKE SA (security association) phase 1 tunnel
      ![IPSEC phase 1 architecture](images/IPSECvpn2.png)
  - IKE Phase 2
    - It is faster and more agile
    - Uses the keys agreed in phase 1
    - Is concerned with agreeing on encryption method and keys used for bulk data transfer
    - The end result is an IPSEC SA phase 2 tunnel (runs over phase 1)
      ![IPSEC phase 2 architecture](images/IPSECvpn3.png)
- There are two types of VPNs - how they match traffic:
  - Policy based VPNs: rule sets match traffic, we can have different rules for different types of traffic
  - Route based VPNs: target matching is done based on prefix. We have a single pair of security associations for each network prefix (less functionality, much simpler to set up)
    ![Route vs Policy Based VPNs](images/IPSECvpn4.png)

## AWS Site-to-Site VPN

- A Site-to-Site VPN is a logical connections between a VPC and an on-premise network running over the public internet. The connection is encrypted using IPSec
- Can be fully HA if it is implemented correctly
- It is quick to provision, it can be provisioned in less than an hour (contrast to DX)
- Components involved in creating a VPN connection:
  - **VPC**
  - **Virtual Private Gateway (VGW)**: it is a gateway object which can be the target of one or more rules in a Route Tables. It can be attached to a single VPC at a time
  - Alternative on the AWS side: **Transit Gateway** (or Cloud WAN) VPN attachment => one VPN reaches many VPCs, supports ECMP, accelerated VPN, large tunnels
  - **Customer Gateway (CGW)**: can refer to 2 different things:
    - Often is referred to the logical configuration in AWS
    - Physical on-premises router which the VPN connects to
    - Needs a static public IP (can be behind NAT using NAT-T, UDP 4500), or a private IP for private IP VPN over DX
  - **VPN Connection** itself: the connection linking the VGW from the AWS to the CGW
  - Every VPN connection has **2 tunnels**, each terminating on a different AWS endpoint in a different AZ. For full HA we also need 2 CGWs (2 VPN connections => 4 tunnels)
- Static vs Dynamic VPN:
  - **Static VPN**:
    - Uses static network configuration: static routes are added to the route tables AWS side, static networks has to be identified on the VPN connection on-premise side.
    - It is simple, it just uses IPSec, works anywhere, having limitation on terms of load-balancing and multi-connection failover
  - **Dynamic VPN**:
    - Uses BGP protocol, if customer router does not support BGP, we can not use dynamic VPNs
    - BGP: allows routing on the fly, allows multiple links to be used at once between the same locations. Allows using HA available architectures
    - Static routes can still be added to the route tables manually
    - Route propagation: if enabled means that routes are added ro the Route Table automatically
- Considerations for VPN:
  - Speed Limitation: _1.25 Gbps_ and 140,000 pps **per tunnel** (standard tunnel), AWS limitation. Customer router limitation might also apply
    - VGW: no ECMP, only one tunnel used actively => ~1.25 Gbps per VPN connection
    - TGW / Cloud WAN: **ECMP** across tunnels/connections (BGP only) to aggregate bandwidth
    - **Large bandwidth tunnels** (Nov 2025): up to **5 Gbps** and 400,000 pps per tunnel. TGW/Cloud WAN only (not VGW), both tunnels same setting, not compatible with accelerated VPN, must be set at creation (recreate to change)
  - MTU 1446 / MSS 1406, no jumbo frames, no PMTUD
  - Latency considerations: can be inconsistent if the traffic goes through the public internet
  - Cost: hourly cost per VPN connection + data transfer out; on-premises data caps might also apply
  - Speed of setup: can done very quickly, within hours or less; IPSec is supported by a wide variety of devices, BGP support is less common. VPNs are always quicker to setup then any other private connection technologies
  - VPNs can be used for Direct Connect backup or they can be used over the Direct Connect for adding a layer of encryption

### Accelerated Site-to-Site VPN

- Performance enhancement for AWS Site-to-Site VPN that uses the AWS global network, the same network used for Global Accelerator and CloudFront
- Using a classic Site-to-Site VPN, the traffic goes through the public internet. In order to avoid this, some companies use a Site-to-Site VPN over Direct Connect. Direct Connect offers more better performance, but at a higher cost. Since DX is not an option for everybody, accelerated Site-to-Site VPN was created to improve performance compared to classic Site-to-Site VPNs
- Accelerated Site-to-Site VPN architecture:
  ![Accelerated Site-to-Site VPN](images/AcceleratedS2SVPN1.png)
- Acceleration can be enabled when creating a Transit Gateway (or Cloud WAN) attachment only! Not compatible with VPNs using Virtual Private Gateways (VGW)
- Acceleration must be chosen at creation time; it does not work over a DX public VIF and is not compatible with large bandwidth tunnels
- Accelerated Site-to-Site VPN has a fixed accelerator cost fee and a data transfer fee

# Highly-Available Site-to-Site VPN Set-up from the Demo

![alt text](images/HADemoVPN.png)

## Client VPN

- Site-to-Site VPN is generally used for one or more business premisses to connect to AWS VPCs. ClientVPN is similar, but instead of sites connecting to AWS, we have individual clients
- A ClientVPN is a managed implementation of OpenVPN
- Any client device which can use the OpenVPN software is supported
- Architecturally we connect to a Client VPN endpoint which can be associated with one VPC and with one ore more Target Networks (high availability)
- Client VPN billing is based on **endpoint association hours** (per associated subnet) + **connection hours** (per active client)
- Client VPN setup:
  - We crate a Client VPN Endpoint and associate it with a VPC and one or more subnets from the VPC
  - This association places an ENI into the subnets associated
  - We can only pick one subnet per AZ
- Client VPN authentication methods: **mutual authentication** (certificates in ACM), **Active Directory** (AWS Managed Microsoft AD / AD Connector), **SAML 2.0 federation** (e.g. IAM Identity Center, Okta). Mutual auth can be combined with AD or SAML
- **Authorization rules**: which client groups (AD group / SAML group) can reach which destination CIDRs. Plus security groups on the endpoint ENIs
- We associate a route table to the Client VPN Endpoint in order to set up routing and connectivity (to internet via NAT Gateways, other VPCs with peering, etc.)
- This route table is pushed to any client which connects to the Client VPN Endpoint
- The default behavior is the Client VPN route table replaces any local routes on the client, meaning the client devices can not access anything locally on their local network without having communication going through the Client VPN Endpoint
- We can use split tunnel VPN, meaning that any routes from the Client VPN Endpoint are added to local client route tables. This solves the problem with the default behavior
- Split tunnel is not the default behavior. It must be enabled by the user, otherwise all the data (including connection to the public internet) will go via the tunnel

![alt text](images/ClientVPN.png)

![alt text](images/SplitTunnelClientVPN.png)

### Client VPN - Additional Details

- Client CIDR: /12 to /22, must not overlap with the VPC or routes pushed; cannot be changed after creation (IPv6 client ranges are assigned by AWS)
- With VPC subnet association, client traffic is **source-NATed** to the endpoint ENI IP => target resources see the ENI IP, not the client IP
- **Native Transit Gateway integration (2026)**: associate the endpoint with a TGW (Client VPN attachment) => reach many VPCs/on-premises without an intermediate VPC, **client source IP preserved**, visible in TGW flow logs. TGW needs a CIDR block; route propagation must be enabled
- **IPv6** (Aug 2025): IPv4-only, IPv6-only or dual-stack endpoints
- **Client connect handler**: Lambda invoked on each connection to allow/deny (posture checks, custom logic)
- Other options: client-to-client communication, self-service portal for config download (SAML), login banner, configurable max session duration (8-24h), connection logging to CloudWatch Logs
- Internet access through the VPN (full tunnel): add a `0.0.0.0/0` route via the associated subnet + authorization rule + NAT gateway/IGW in the VPC

## Other Site-to-Site VPN Features

- **Private IP VPN** over Direct Connect: IPsec tunnels using private IP addresses over a **transit VIF**, terminating on a TGW (no public VIF/public IPs needed). Use for encryption over DX for compliance
- **IPv6**: IPv6 traffic inside tunnels (TGW), and IPv6 outer tunnel IPs (2025)
- **VPN Concentrator** (Nov 2025): one TGW attachment for up to 100 low-bandwidth remote sites (up to 100 Mbps per site, 5 Gbps aggregate), BGP only, TGW only (not VGW/Cloud WAN), no ECMP. Use case: 25+ branch offices
- **Tunnel options**: IKEv1/IKEv2, encryption/integrity algorithms, DH groups, DPD timeout and action, startup action (AWS can initiate the IKE), inside tunnel CIDRs, pre-shared key or private certificate (ACM PCA) authentication
- **Site-to-Site VPN logs** to CloudWatch Logs (IKE/IPsec negotiation, DPD, BGP status)
- Tunnel endpoint replacements happen during AWS maintenance => one tunnel at a time goes down; only a design with both tunnels up (and BGP) avoids impact
- **AWS VPN CloudHub**: one VGW with multiple CGWs (each unique BGP ASN) => branch offices can talk to each other through AWS (hub-and-spoke over VPN). Modern equivalent: TGW
- **Software VPN** on EC2 (e.g. strongSwan, vendor appliance): when we need features AWS VPN doesn't support (e.g. specific crypto, transitive designs); we manage HA ourselves, disable source/dest check
- Quotas: 50 CGWs and 50 VPN connections per Region (adjustable), 10 VPN connections per VGW, 5 VGWs per Region

## VPN vs Direct Connect - Exam Shortcuts

- Need connectivity in hours/days, cheap, encrypted => Site-to-Site VPN
- Need consistent latency, high throughput (10-400 Gbps) => DX (weeks to months)
- DX backup on a budget => VPN (DX routes win over VPN routes for the same prefix)
- Encryption over DX => VPN over public VIF, private IP VPN over transit VIF, or MACsec (L2, hop-by-hop)
- Better VPN performance without DX => accelerated VPN
- Remote users => Client VPN (not Site-to-Site)
