# VPC Flow Logs

- Essential diagnostic tools for complex networks
- They only capture packet metadata, they do not capture packet content. For packet content a packet sniffer is required to be installed on an instance
- Metadata can include: source/destination IP, source/destination ports, packet size, other externally visible metadata, etc.
- Flow logs can capture data at various different points:
    - Applied to a VPC: all interfaces in that VPC
    - Subnet: every network interface in the subnet only
    - Network interface: only monitor traffic at a specific interface
    - Also: Transit Gateway and TGW attachments (Transit Gateway Flow Logs) for centralized network visibility
- VPC Flow Logs are NOT realtime, there is delay between traffic leaving monitored interfaces and showing up in the flow logs <span style="color: red;">EXAM</span>
- Flow logs can be configured to use S3, CloudWatch Logs or Kinesis Firehose for the destination (cross-account delivery supported for S3/Firehose - central logging account)
- Aggregation interval: 10 minutes (default) or 1 minute
- S3 destination: Parquet format and Hive-compatible partitions (cheaper/faster Athena queries); CloudWatch Logs: query with Logs Insights, metric filters + alarms (e.g. spike of REJECTs)
- Flow logs can be cofigured to capture data only at accepted connections, only at rejected connections or they can capture metadata at all connections.

## VPC Flow Logs Record Content

- `<version>`
- `<account-id>`
- `<interface-id>`
- `<srcaddr>`: source IP address
- `<dstaddr>`: destination IP address
- `<srcport>`: source port, 0 if no port is used (example in case of ICMP ping)
- `<dstport>`: destination port
- `<protocol>`: ICMP=`1`, TCP=`6`, UDP=`17`, etc. <span style="color: red;">Need to remember for EXAM</span>
- `<packets>`
- `<bytes>`
- `<start>`
- `<end>`
- `<action>`: traffic is `ACCEPT`ed or `REJECT`ed
- `<log-status>`
- The above is the default format (version 2). **Custom format** adds fields (v3-v5) such as `vpc-id`, `subnet-id`, `instance-id`, `tcp-flags`, `type`, `pkt-srcaddr`/`pkt-dstaddr` (real source/destination behind a NAT gateway or for traffic through middleboxes), `flow-direction`, `traffic-path` (e.g. IGW, VPC endpoint, TGW), `pkt-src-aws-service`, `az-id`, `region`, ECS fields (`ecs-cluster-name`, `ecs-task-id`...) <span style="color: red;">EXAM</span>
- Troubleshooting with SG vs NACL: SGs are stateful, NACLs stateless:
    - Inbound `ACCEPT` + outbound `REJECT` -> NACL is blocking the response (ephemeral ports)
    - Inbound `REJECT` -> SG or NACL denies inbound

## Notes

- VPC Flow Logs do not log all the traffic, things like the communication with the metadata IP (169.254.169.254), AWS time sync server (169.254.169.123), DHCP, Amazon DNS server and Amazon Windows license is not recorded
- Traffic mirroring is the AWS feature for full packet capture (to an ENI/NLB/GWLB target) when payload inspection is required
- Other things not logged: traffic to reserved VPC router IP, traffic between an endpoint ENI and NLB, ARP; Amazon DNS queries can be logged via Route 53 Resolver query logs instead
- Analysis tools: Athena, CloudWatch Logs Insights, Amazon Detective and GuardDuty (they consume VPC Flow Logs independently - no need to enable them for GuardDuty), Security Lake, VPC Reachability Analyzer / Network Access Analyzer for path/config analysis (not logs)
