# AWS Network and DNS Firewalls

## AWS Network Firewall

- AWS Network Firewall is a stateful, managed, network firewall and intrusion detection and prevention service for your virtual private cloud
- It is a managed network appliance used to filter incoming/outgoing traffic
- In a VPC it is deployed in a separate subnet with a Firewall Endpoint
- Has a flexible rules engine which gives a fine-grained control over network traffic
- Resources send traffic to the firewall subnet, the firewall will redirect traffic to the gateway (internet gateway, transit gateway, DX connection, IPSEC VPN)
- The firewall endpoint uses gateway load balancer
- Route table configuration is needed for the protected subnet to be able to send outgoing traffic to the gateway load balancer
- Network Firewall deployment can be managed with AWS Organizations and AWS Firewall Manager
- Network Firewall features:
    - Stateful and stateless firewall
    - Intrusion Prevention System (IPS)
    - Web filtering
- In the firewall subnet we should not deploy any resources
- For HA we should allocate a subnet per AZ
- Routing must be **symmetric** - traffic of a flow must go through the same firewall endpoint (AZ) in both directions; use TGW appliance mode for centralized inspection <span style="color: red;">EXAM</span>
- Rule engine:
    - **Stateless** rules: evaluated first, per packet (5-tuple), actions pass / drop / forward to stateful engine
    - **Stateful** rules: **Suricata**-compatible IPS rules, domain lists (allow/deny by SNI/HTTP Host - e.g. egress allow-list of FQDNs), 5-tuple rules; strict rule ordering recommended; alert/drop/reject/pass
    - Managed rule groups: AWS managed threat signatures and domain lists (malware, botnet, crypto-mining), **active threat defense** (AWS threat intelligence - MadPot - indicators, auto-updated)
    - TLS inspection: decrypt/inspect/re-encrypt inbound and outbound TLS using certificates from ACM / Private CA
    - Geographic IP filtering, IP sets and port sets (can reference VPC prefix lists)
- Deployment models:
    - Distributed: a firewall in each VPC (between IGW and workloads, or NAT gateway and IGW)
    - Centralized: inspection VPC attached to Transit Gateway (east-west and egress inspection) - since 2025 Network Firewall can be **attached natively to a Transit Gateway** (no inspection VPC/subnets/route tables to manage) <span style="color: red;">EXAM</span>
    - Ingress inspection with ALB/NLB, IGW ingress routing (edge association) to send inbound traffic through the firewall
    - Multiple VPC endpoint associations: share one firewall across several VPCs/accounts (via RAM)
- Logging: alert logs, flow logs and TLS logs to S3, CloudWatch Logs or Firehose
- Network Firewall vs others:
    - Security Groups / NACLs: L3/L4 only, no domain filtering, no IPS
    - WAF: L7 HTTP(S) only, attached to CloudFront/ALB/API GW - not a VPC egress firewall
    - Gateway Load Balancer + third-party appliances (Palo Alto, Fortinet): when a specific vendor appliance is required
    - Route 53 DNS Firewall: blocks DNS resolution only (can be bypassed by direct IP connections) - combine with Network Firewall for defense in depth
- Exam: "restrict outbound internet access from private subnets to a list of approved domains" -> Network Firewall stateful domain list (or DNS Firewall for DNS-level), NAT Gateway alone can't do it

## Route 53 Resolver DNS Firewall

- Allows us to filter and regulate outbound DNS traffic for VPCs
- Requests route through Route 53 Resolver for DNS queries, the DNS Firewall will help prevent DNS exfiltration of data
- We can monitor and control the domains application can query
- We can use AWS Firewall Manager to centrally configure and manage DNS Firewall
- Building blocks:
    - **Domain lists**: custom lists or AWS managed domain lists (malware, botnet C2, aggregate threat list, Amazon GuardDuty threat list)
    - **Rule groups**: ordered rules with actions ALLOW, BLOCK (NODATA, NXDOMAIN or OVERRIDE to a custom CNAME/walled garden), ALERT; can inspect query type
    - Rule groups are associated to VPCs (priority order); share across accounts via **AWS RAM** or **Route 53 Profiles**
- **DNS Firewall Advanced** (Nov 2024): ML/heuristic protections against threats without a domain list - **DNS tunneling** (data exfiltration), **DGA** (domain generation algorithms) and **Dictionary DGA** (Nov 2025); confidence threshold per rule
- Fail mode: **fail-closed** (block queries if firewall can't respond - security first) or **fail-open** (availability first) - configured per VPC
- Only filters queries sent to the Route 53 Resolver (`.2` VPC resolver) - instances using a custom/external DNS server bypass it -> block outbound port 53 with SG/NACL/Network Firewall <span style="color: red;">EXAM</span>
- Query logging via Route 53 Resolver query logs (CloudWatch Logs, S3, Firehose); GuardDuty also analyzes DNS logs
