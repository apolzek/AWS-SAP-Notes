## AWS PrivateLink

- Lets you access services defined in other VPC using private IP
- Traffic never leaves AWS network
- VPC Service Provider: VPC where the consumed services are defined (third-party account here)
  - Services that you want to expose
  - NLB at the front (or GWLB for appliances)
  - NLB configured as VPC endpoint service
  - Provider controls **allowed principals** (accounts, roles, `*`) and optionally **acceptance required** for each connection request
- VPC Service Consumer: VPC where the resources are consumed
  - VPC Interface Endpoint (ENI) in each subnet pointing to the provider's NLB service
- Permissions: the provider allow-lists IAM principals/accounts on the endpoint service; network access is then controlled with SGs on the consumer endpoint ENIs (and SGs on the NLB, NACLs). The consumer's ENIs can then route traffic to the NLB
- Traffic is **unidirectional**: consumer => provider only. The provider cannot initiate connections to the consumer
- **Overlapping CIDRs are fine** (no routing between VPCs, only ENIs) - key exam differentiator vs VPC peering/TGW
- The provider sees the NLB private IPs as source, not the consumer IPs => enable **Proxy Protocol v2** on the NLB to get client IP + endpoint ID

Facts:

- PrivateLink can be picked as an option when creating an endpoint to integrate with a 3rd-party for example.
- Highly-available only via multiple endpoints (one ENI per AZ; consumer can only use AZs in common with the provider - match by **AZ ID**, not AZ name)
- IPv4, IPv6 and dual-stack supported (dual-stack NLB required for IPv6); TCP and **UDP** (UDP support since Oct 2024)
- Private DNS is supported => possible to use the service's public hostname instead of the private IP to reach the provider VPC.
  - Behind the scenes => AWS creates a private hosted zone in R53
  - Record that translates service's public hostname => private ENI IP
  - From within a VPC attached to that private hosted zone => record is resolved with private hosted zone
- Private DNS name for a custom endpoint service requires the provider to **verify domain ownership** (TXT record)
- Direct Connect, Site-to-Site VPN, VPC Peering and Transit Gateway are supported: on-premises clients can reach interface endpoints (with Route 53 inbound resolver endpoints for the private DNS names)
- **Cross-Region PrivateLink**:
  - Endpoint services (customer/SaaS) - Nov 2024: provider enables extra "Service Regions", consumer creates an interface endpoint in its own Region pointing to the remote-Region service
  - AWS services (select ones like S3, ECR, Route 53...) - Nov 2025: interface endpoint in Region A for a service in Region B
  - Interface endpoints only (not gateway/GWLB/resource endpoints), no zonal DNS, inter-Region data transfer charges apply
- Pricing: per endpoint per AZ-hour + per GB processed (both consumer side)

![PrivateLink](images/PrivateLink.png)

## VPC Endpoints

### Gateway Endpoints

- Gateway endpoints provide private access to supported services: **S3** and **DynamoDB**
- They allow any resource in a private only VPC to access S3/DynamoDB (public services)
- We create a gateway endpoint per service per region and associate it to one or more subnets in a VPC
- We allocate a gateway endpoint to a subnet, a _Prefix List_ is added to the route table for the subnet. This prefix lists targets the gateway endpoint
- Any traffic targeted to S3/DynamoDB will go through the gateway endpoint and not through the internet gateway
- Gateway endpoints are highly available across all AZs in a region by default. You can select which subnets it should be deployed into, and AWS configures it on your behalf.
- **Endpoint policy**: allows what things can be connected to from the endpoint (example: a particular subset of S3 buckets)
- Gateway endpoints can be used to access services in the **same region only**
- Gateway endpoints allow private only S3 buckets: S3 buckets can be set to private allowing only access from the gateway endpoint. This will help prevent _Leaky Buckets_
- Gateway endpoints are logical gateway objects, they can be only accessed from inside the assigned VPC => **NOT reachable from on-premises (VPN/DX), peered VPCs or via TGW**. For those cases use S3/DynamoDB interface endpoints
- Gateway endpoints are **free** (no hourly or data processing charge) - prefer them for in-VPC S3/DynamoDB access to save NAT gateway processing costs
- Bucket policies can restrict access to a specific endpoint with `aws:SourceVpce` or VPC with `aws:SourceVpc` (only works for traffic through a VPC endpoint, `aws:SourceIp` does not see private IPs)

![w/o gateway endpoint](images/WithoutGateway.png)

![with gateway endpoint](images/WithGateway.png)

### Interface Endpoints

- Interface endpoints provide private access to AWS public services similar to Gateway Endpoints
- Historically => any other service than S3 & DynamoDB => interface, but now, interface endpoints are available for S3 and DynamoDB (2024) too
- Difference between gateway endpoints and interface endpoints is that interface endpoints are **not HA by default**. Interface endpoints are added to subnets as an ENI
- In order to have HA, we have to add an interface endpoint to every subnet per AZ inside of a VPC
- Interface endpoints are able to have security groups assigned to them (gateway endpoints do not allow SGs)
- We can also use endpoints policies, similar to gateway endpoints
- Interface endpoints support TCP (and UDP for endpoint services), IPv4/IPv6/dual-stack
- Interface endpoints use **PrivateLink** behind the scenes
- Gateway endpoints use prefix lists, interface endpoints use DNS. Interface endpoints provide a new DNS name for every service they are meant communicate with
- Interface endpoints are given a number of DNS names:
  - Endpoint Regional DNS: works from whatever AZ used to access the endpoint (good for HA) => vpce-<id>.<service>.<region>.vpce.amazonaws.com
  - Endpoint Zonal DNS: resolves to the IP from the AZ of the DNS record => vpce-<id>-<az>.<service>.<region>.vpce.amazonaws.com
  - PrivateDNS: overrides the default service DNS with a new version pointing to interface endpoint (**enabled by default**). AWS creates a private hosted zone in VPC => use s3.amazonaws.com directly instead of the vpce hostname and it will resolve to the private IP.
    - Requires VPC attributes `enableDnsSupport` and `enableDnsHostnames`
    - For S3, private DNS can be set to "inbound endpoint only" so in-VPC traffic still uses the free gateway endpoint while on-premises uses the interface endpoint
    - The AWS-managed private hosted zone is not shareable with other VPCs => for centralized endpoints (hub VPC shared by spokes via TGW) disable private DNS and create our own PHZ, associate it with the spoke VPCs (or use Route 53 Profiles)

## Gateway vs Interface Endpoint

Gateway:

- HA by design
- Resource: gateway in route table
- Uses Route Tables, not DNS. The public hostname is routed privately thanks to the route added to the RT
- Only S3 and DynamoDB
- Cannot be shared/used from other VPCs or accounts' networks (but the endpoint can reach buckets owned by other accounts if the policies allow it)
- Free

Interface:

- Uses ENIs => not HA by design
- Resource: ENI in a subnet
- Uses DNS names to resolve into private IP
- Cross-account supported via PrivateLink sharing
- Reachable from on-premises, peered VPCs and TGW-connected VPCs
- Hourly + per-GB cost

## VPC Endpoints Policies

- Endpoints policies don't grant access to any AWS services => the underlying identities accessing the resources still need the permissions to access them
- An endpoint policy only limits access if the service is accessed to the specific endpoint
- The endpoint policy contains a policy and conditions (who has access to what)
- Policies are commonly used to limit what private VPCs can access
- By default => giving full access to all actions in all services to the endpoint, and letting all principals in the AWS account use the endpoint:

```
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```

- Work like normal IAM policies:
  ![Endpoint policy](images/EndpointPolicy.png)

## Demo: Restricting Access to S3 Content

![S3 Restriction with Endpoint](images/S3EndpointRestrict.png)

## Gateway Load Balancer Endpoints

- Third type of VPC endpoint: `GatewayLoadBalancer` endpoint, targets a GWLB endpoint service (PrivateLink)
- Used as a **route table target** to transparently insert third-party firewalls/IDS (GENEVE, port 6081) in the traffic path - including ingress routing from the IGW
- See `05-compute/gwlb.md`

## Resource Endpoints and Service Network Endpoints (Dec 2024)

- PrivateLink + VPC Lattice extended beyond NLB-fronted services:
  - **Resource gateway**: created in the provider VPC, entry point for resources
  - **Resource configuration**: represents a TCP resource - an IP address, a DNS name, or an AWS resource ARN (e.g. an RDS database); can be a group of resources; shareable via **RAM**
  - **Resource endpoint** (`Resource` VPC endpoint type): consumer accesses one resource privately, **no NLB required** on the provider side
  - **Service network endpoint** (`ServiceNetwork` VPC endpoint type): access a whole VPC Lattice service network (services + resources) from a VPC, including from on-premises over DX/VPN
- Exam trigger: "share access to an RDS database / on-premises server in another account privately without creating an NLB" => resource configuration + resource endpoint

## Amazon VPC Lattice

- Managed **application networking** service (L7) for service-to-service communication across VPCs and accounts
- Components:
  - **Service network**: logical boundary; VPCs are associated with it (with SGs), services and resource configurations are associated with it
  - **Service**: listeners (HTTP, HTTPS, gRPC, TLS passthrough) + rules + target groups (EC2 instances, IPs, Lambda, ALB, ECS, EKS via Gateway API controller)
  - **Auth policies**: IAM resource policies on the service network and/or service; clients sign requests with SigV4 => zero trust, authN/authZ per request
- Works with **overlapping CIDRs**, no need for peering/TGW routing, no sidecar proxies
- Sharing across accounts via **RAM** (service networks, services, resource configurations)
- Monitoring: access logs (CloudWatch Logs, S3, Firehose)
- Pricing: per service-hour + per GB + per request
- When to use:
  - **Lattice**: many microservices across accounts/VPCs, need L7 routing + IAM auth, mixed compute (EC2/ECS/EKS/Lambda)
  - **PrivateLink (NLB)**: expose one service (L4) to many consumers/SaaS
  - **TGW/peering**: full network connectivity (any protocol, any port)
