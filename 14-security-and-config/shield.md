# AWS Shield

- Provides protection against DDoS attacks
- Provides a custom designed set of protection against DDoS attacks
- Offers protection against all known infrastructure Layer 3 and Layer 4 DDoS attacks: Network Volumetric attacks (L3), Network Protocol Attacks (L4) for example TCP SYN Floods
- Comes in 2 forms:
    - **Shield Standard**:
        - It is free of charge for all AWS customers
        - Protection is at the perimeter of the network (this can be either at the region/VPC or AWS edge in case of CloudFront usage)
        - Protects against common Network (L3) and Transport (L4) layer attacks
        - We get the best protection if we use Route 53, CloudFront or Global Accelerator
        - It does not provide anything against proactive capability of any form of explicit configurable configuration
    - **Shield Advanced**:
        - Costs $3000 per month per organization, 1 year lock-in + charge for data (OUT) / month. Cost is not per account, if we want protection for multiple accounts, we have to make sure they are in the same organization (consolidated billing - one subscription fee covers all accounts in the org)
        - Expands the range of products which can be protected: CloudFront, Global Accelerator, Route53, anything associated with am EIP (example EC2 instances), load balancers (ALB, CLB, NLB)
        - Protection offered by Shield Advanced is not automatic. We need to enable it in Shield Advanced or as part as AWS Firewall Manager Shield Advanced policy
        - Shield Advanced provides access to 24/7 advanced response team named AWS Shield Response Team (SRT)
        - Provides financial insurance for any increase of payments in case of DDoS attacks (**DDoS cost protection**: service credits for scaling charges on protected EC2, ELB, CloudFront, Global Accelerator, Route 53 caused by an attack - request via support) <span style="color: red;">EXAM</span>
        - Additional Shield Advanced features
            - Integration with WAF:
                - Shield Advanced integrates with WAF to protect against Application Layer (L7) attacks
                - Includes basic AWS WAF fees for web ACLs, rules and web requests
            - Real time visibility of DDOS events and attacks
            - Health-based detection: application specific health checks used by proactive engagement team to provide faster detection and mitigation of any issues
            - Protection groups:
                - We can create grouping of resources that Shield Advanced protects
                - We can define the criteria of membership for groups, any new resource will automatically be added
            - Automatic application layer DDoS mitigation: Shield Advanced creates/updates WAF rate-based rules in a Shield-managed rule group in the web ACL during an attack (Count or Block mode)
                - **Status (2026):** this legacy L7 automatic mitigation is being replaced by the AWS WAF **Anti-DDoS managed rule group** (`AWSManagedRulesAntiDDoSRuleSet`); Shield Advanced adds it to eligible web ACLs (Count mode) from Jul 2026 and the legacy feature is scheduled to be discontinued Jan 1, 2027
            - **Proactive engagement**: SRT contacts us proactively when a Route 53 health check associated with a protected resource becomes unhealthy during an event (requires health checks + emergency contacts, and Business/Enterprise support)
            - Access to SRT requires **Business or Enterprise Support** plan; grant SRT an IAM role (`AWSShieldDRTAccessPolicy`) and optionally access to WAF log S3 buckets
            - For protected EC2 EIPs, Shield Advanced enforces the VPC network ACLs at the AWS network border during an attack (drops traffic before it reaches the VPC)
            - Global threat dashboard, CloudWatch metrics (`DDoSDetected`) and attack diagnostics

## Shield Standard vs Advanced - Exam Notes

- Standard: automatic, free, L3/L4, no SRT, no cost protection, no L7
- Advanced: when the question mentions DDoS **cost protection**, **24/7 DDoS response team**, **L7 DDoS with automatic mitigation**, attack visibility/reporting, or protection for EC2 EIPs/NLB <span style="color: red;">EXAM</span>
- Best practice DDoS-resilient architecture (AWS DDoS Best Practices whitepaper): CloudFront + Route 53 + Global Accelerator at the edge, WAF rate-based rules, ALB/ELB to absorb, Auto Scaling, minimize exposed attack surface (private subnets, SG/NACL), origin protection (CloudFront VPC origins or origin custom header + WAF / managed prefix list)
- Use Firewall Manager to enforce Shield Advanced protection on all accounts/resources in the org
- Shield Advanced includes WAF charges (web ACLs, rules, requests) for protected resources - but not paid managed rule groups like Bot Control
- **AWS Shield network security director** (preview 2025): discovers network resources and topology, analyses configurations against best practices (SG, NACL, WAF, Network Firewall) and recommends remediation