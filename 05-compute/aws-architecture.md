# Regional and Global AWS Architecture

- There are 3 main type of architectures:
  - Small scale architectures: one region/one country
  - Small architecture with DR: one region + backup region for disaster recovery
  - Multiple region based systems
- Global components:
  - Global Service Location and Discovery
  - Content Delivery (CDN) and optimization
  - Global health checks and Failover
- Regional components:
  - Regional entry point
  - Scaling and resilience
  - Application services and components

![Regional and Global Architecture](images/RegionalandGlobalInfrastructure2.png)

- **Web Tier:** Customer facing layer. There would be regional based services like ALB or API gateway depending on application architecture. Abstracts customers from underlying architecture.
- **Compute Tier:** The infra to web tier is provided by compute tier using EC2, lambda or containers.
- **Storage Services:** Services like EBS, EFS or S3.
- **Data Storage:** Products like RDS, Aurora, DynamoDB, and RedShift.
- **Caching:** ElastiCache (Valkey/Redis OSS/Memcached) for general caching, DynamoDB Accelerator (DAX) for DynamoDB
- **App Services:** Kinesis, Step Functions, SQS, SNS

## Global Components - Which Service

- **Global service location and discovery**: Route 53 (latency, geolocation, geoproximity, weighted, failover routing + health checks), Route 53 Application Recovery Controller (routing controls, readiness checks, zonal shift)
- **Content delivery**: CloudFront (caching, origin failover with origin groups, Lambda@Edge / CloudFront Functions, VPC origins)
- **Global static entry point**: Global Accelerator (2 anycast static IPs, AWS backbone, fast regional failover < 1 min, no DNS caching issues, TCP/UDP)
- Global data: DynamoDB global tables, Aurora Global Database (RPO ~1s, managed failover/switchover), S3 Cross-Region Replication (RTC: 15 min SLA), ElastiCache Global Datastore, DocumentDB global clusters

## Regional Components - Resilience

- Always design for **multiple AZs** within a region (ALB/NLB across AZs, ASG across AZs, Multi-AZ databases)
- Static stability: pre-provision capacity in each AZ so the loss of one AZ doesn't require launching new capacity (N+1 per AZ)
- Cell-based architecture / shuffle sharding to limit blast radius
- Zonal isolation: cross-zone LB off + zonal shift / zonal autoshift to evacuate an impaired AZ

## DR Strategies (RTO/RPO)

| Strategy | RPO / RTO | Cost | Description |
| --- | --- | --- | --- |
| Backup & Restore | hours / hours | $ | backups copied to another region (AWS Backup cross-region/cross-account), restore on disaster |
| Pilot Light | minutes / tens of minutes | $$ | data replicated live, core infra off/minimal, scale up on failover |
| Warm Standby | seconds / minutes | $$$ | scaled-down but fully functional copy running, scale out on failover |
| Multi-site Active/Active | near zero / near zero | $$$$ | full capacity in multiple regions, traffic split with Route 53 / Global Accelerator |

- **AWS Elastic Disaster Recovery (DRS)**: block-level replication of servers (on-prem or cloud) to a staging area, launch in minutes (RPO seconds, RTO minutes) - the managed answer for lift-and-shift DR
- Exam tip: choose the cheapest strategy that meets the stated RTO/RPO; "minimal cost + hours RTO" => backup & restore; "data current, infra off" => pilot light
