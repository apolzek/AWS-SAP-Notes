# Gateway Load Balancers (GWLB)

- It is a product to help us run and scale third party security appliances
- Things like firewalls, intrusion detection and prevention systems or even data analysis tools
- We can use GWLB to perform inbound and outbound transparent traffic inspection and protection
- At high level a GWLB has two major components:
  - GWLB endpoints: run from a VPC where traffic enters/leaves via these endpoints. These are similar to interface endpoints with some key improvements
  - GWLB itself: there are normal EC2 instances running security software
- The GWLB needs to forward traffic without any alteration, the security appliance needs to review packets as they are sent/received
- GWLB use a protocol named GENEVE (UDP port **6081**), this is a traffic and metadata tunneling protocol - appliances must support GENEVE (most vendor firewalls on the Marketplace do)
- GWLBs are layer 3/4 devices, similar to NLB, but they integrate with GWLB endpoints and they encapsulate all traffic between them and the target using the GENEVE protocol. When the scanning is finished, the traffic is returned on the same tunnel to the GWLB from where, using the GWLB endpoint, it is returned to the intended destination
- GWLB will load balance between security appliances, so we can horizontally scale
- GWLB manage flow stickiness, one flow of data will always use the same appliance
  - Default: 5-tuple (src/dst IP, src/dst port, protocol); configurable to 3-tuple or 2-tuple
  - Traffic is **symmetric**: forward and return packets of a flow go through the same appliance (required for stateful firewalls)
- Ingress route tables:
  - They are placed on the internet gateway
  - They influence what happens with the traffic arriving to the VPC
  - They can redirect traffic to GWLB endpoints

![GWLB](images/GWLB.png)

## GWLB Key Facts

- GWLB listens on all IP ports/protocols - there are no listeners with ports, one target group with protocol `GENEVE`
- GWLB endpoints (`GatewayLoadBalancer` VPC endpoint type) are powered by **PrivateLink** => the appliance VPC can be in a **different account** (security/network account, or a Marketplace vendor offering firewall-as-a-service)
- GWLB endpoint is a route table target (`vpce-xxx`) - used in subnet route tables and IGW/VGW **ingress route tables** (edge association)
- One GWLB endpoint per AZ; for HA deploy one per AZ and keep routing AZ-local
- Health checks: TCP, HTTP or HTTPS against the appliances; on appliance failure existing flows can be rebalanced (target failover setting `rebalance`) or kept (`no_rebalance`, default)
- Cross-zone load balancing: OFF by default (inter-AZ charges if enabled)
- Target types: instance or IP; appliances typically in an ASG
- Source/destination check NOT required on appliances (traffic arrives inside GENEVE tunnels addressed to them)
- MTU: GWLB supports up to 8500 bytes; appliances must handle the GENEVE overhead (68 bytes)
- Pricing: hourly + GLCU for GWLB, plus hourly + per-GB for each GWLB endpoint

## Centralized Inspection Patterns

- **Distributed**: GWLB endpoint in each spoke VPC, routes through it via ingress route table (north-south, internet ingress/egress per VPC)
- **Centralized with Transit Gateway**: inspection VPC attached to TGW with **appliance mode enabled** on the attachment (keeps both directions of a flow in the same AZ => symmetric) - east-west and egress inspection for many VPCs
- With Cloud WAN: service insertion / network function groups route traffic through an inspection VPC
- GWLB vs **AWS Network Firewall**:
  - Network Firewall: AWS-managed stateful firewall (Suricata rules, domain filtering, TLS inspection), no appliances to manage - also uses GWLB-like endpoints internally
  - GWLB: bring your own third-party appliances (Palo Alto, Fortinet, Check Point...), existing licenses/skills, IDS/IPS, DPI
  - Exam tip: "managed, no third-party" => Network Firewall; "existing vendor appliances, scale them transparently" => GWLB

## GWLB Architecture

Below is an architecture where EC2 instances are running in a pair of private subnets followed by ALB which is runig in pair of public subnets. On thie right side we have a VPC running a set of security applicances inside ASG which ca grow or sink based on the load.

1. The traffic hits the IG which is configure with route table which determines what happens if the traffic arrives at VPC.
2. Traffic is routed to GWLB endpoint.
3. Traffic is forwarded to GWLB itself which is running in security VPC. At this point the packets still have original IP address.
4. Packets are encapsulated using GENEVE protocol and forwarded to security appliances.
5. Once the packets are analyzed, they are returned to the GWLB.
6. The packets are stripped and returned back to GWLB endpoint via the internet.
7. Since the original IP are maintained, the traffic is routed to ALB using local route table.
8. The traffic is forwarded to the choosen application instance.
9. The traffic is returned back from instance to the GWLB in steps from 9 to 14, following the same flow as it came to the instance.
10. The data is sent back to the original client using the IG.

![GWLB Architecture](images/GWLB3.png)
