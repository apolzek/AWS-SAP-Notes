# BGP - Border Gateway Protocol

- BGP is a routing protocol
- Used to control how data flows from point A to point B
- BGP is made up from a lot of self managing networks know as Autonomous Systems (AS)
- AS could be a large network, collection of routes etc. and is viewed as a black box from BGP perspective
- Each AS is allocated a number by IANA, named ASN
- ASNs are 16 bit in length and range from 0 to 65535, the range from 64512 to 65534 is private
- 32 bit ASNs ("long ASNs") also exist: private range `4200000000`-`4294967294`. AWS supports long ASNs on Customer Gateways (Site-to-Site VPN), DX VIFs and Cloud WAN, so they can show up in scenarios
- ASNs are used by the BGP to identify different entities on the network
- BGP is designed to be reliable and distributed, and it operates of TCP/179
- It is not automatic: BGP peerings (neighbors) between two AS are configured manually on both sides (peer IP + remote ASN, optionally MD5 password)
- Autonomous Systems do exchange network topology information between them
- BGP is a path-vector protocol: it exchanges the best path to a destination between peers, the path is called **ASPATH** (Autonomous System Path)
- BGP does not take into account link speed or condition, it focuses on path only
- iBGP - internal BGP, routing within an AS
- eBGP - external BGP, routing between AS's
- BGP example:
    ![BGP 101](images/BorderGatewayProtocol101.png)
- If all other attributes are equal, BGP chooses the shortest AS_PATH. We can influence the path by artificially extending it (**AS_PATH prepending**: the AS adds its own ASN multiple times to make a path look longer/less preferred)

## BGP Path Selection (simplified, what matters for the exam)

- Order of evaluation (vendor-neutral, simplified):
    1. Longest prefix match (not a BGP attribute, it is how routing tables work - always evaluated first)
    2. Weight (Cisco proprietary, local to the router)
    3. **Local Preference** (higher wins) - influences **outbound** traffic from our AS, shared inside our AS via iBGP
    4. Locally originated routes
    5. **AS_PATH** (shorter wins) - prepending influences **inbound** traffic to our AS
    6. Origin (IGP < EGP < incomplete)
    7. **MED** - Multi-Exit Discriminator (lower wins), a hint to a neighbor AS about which entry point to use; low priority
    8. eBGP over iBGP, then lowest IGP cost, then tie-breakers (router ID etc.)
- Rule of thumb:
    - To control how traffic **leaves** our network => Local Preference
    - To control how traffic **comes into** our network => AS_PATH prepending, more specific prefixes, MED, or communities the neighbor honors
- **BGP communities**: tags (`ASN:value`) attached to prefixes to signal a policy to the neighbor. Well-known: `NO_EXPORT` (do not advertise outside the receiving AS)
- **ECMP**: when multiple paths have identical attributes, traffic can be load-balanced across them (active/active)
- **BFD** (Bidirectional Forwarding Detection): fast failure detection (sub-second) instead of waiting for the BGP hold timer (DX default hold time 90s; BFD is enabled by default on the AWS side of DX, we must enable it on our router)

## BGP in AWS

- Where BGP is used: Site-to-Site VPN (dynamic), Direct Connect VIFs (always BGP), Transit Gateway Connect (GRE + BGP), Cloud WAN, VPC Route Server (BGP from appliances inside a VPC, 2025)
- AWS-side ASN:
    - VGW / TGW / DX Gateway: default Amazon ASN `64512`, or we pick a custom private ASN (16 or 32 bit) at creation time; it cannot be changed later without recreating the gateway
    - Public VIF: AWS uses ASN `7224`
- **Direct Connect - private/transit VIF**, AWS outbound path selection (AWS => on-premises):
    1. Longest prefix
    2. **Local preference BGP communities** set by us on our prefixes: `7224:7100` (low), `7224:7200` (medium, default for DX locations associated with the same Region), `7224:7300` (high). Evaluated **before** AS_PATH
    3. AS_PATH length (prepending works)
    4. MED (not recommended)
    5. ECMP if everything is equal (AS-path multipath relax not supported: same ASN required)
- Active/passive DX: tag primary prefixes `7224:7300` and backup `7224:7100` (or advertise more specific prefixes on the primary). Active/active: same community on both
- **Direct Connect - public VIF** communities:
    - Scope communities we apply to our prefixes: `7224:9100` (local Region only), `7224:9200` (all Regions on the continent), `7224:9300` (global, the default)
    - AWS tags its advertised prefixes with `7224:8100` (same Region as the DX location), `7224:8200` (same continent), no tag (other continents), plus `NO_EXPORT`
    - With a private ASN on a public VIF, AWS replaces it with `7224` => AS_PATH prepending is stripped and ineffective outside AWS
- Route preference DX vs VPN for the same prefix:
    - VPC route table with VGW: DX propagated routes > VPN static > VPN BGP (DX wins even if the VPN has a shorter AS_PATH)
    - Transit Gateway: static > prefix list > VPC propagated > DX gateway propagated > Connect > VPN over private DX > VPN > VPN Concentrator > Client VPN > peering (Cloud WAN). Same attachment type: shorter AS_PATH, then lower MED
- Prefix limits: max 100 prefixes per BGP session on private/transit VIF (default; can be raised up to 1,000 with prefix controls), 1,000 on public VIF; exceeding it puts the session in idle/DOWN state. VPN on VGW: 100 routes from the CGW, VPN on TGW: 1,000
- Exam tips:
    - "Prefer DX1 over DX2 for return traffic from AWS" => local preference communities or more specific prefixes, not MED
    - "Make on-premises prefer one path into AWS" => that is on-premises local preference (our side), AWS only controls AWS => on-premises
    - Static VPN cannot do ECMP (TGW) and fails over less gracefully than BGP => choose dynamic (BGP) VPN for HA and active/active
