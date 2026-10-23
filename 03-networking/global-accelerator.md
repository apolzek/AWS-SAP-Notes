# AWS Global Accelerator

- Designed to optimize the flow of data from user to AWS
- Similar to CloudFront, both improve performance when communicating with services hosted in AWS
- Global Accelerator provides 2 static **anycast** IPv4 addresses (dual-stack accelerators: 2 IPv4 + 2 IPv6). Anycast IP addresses are special type of IP addresses
- The IPs are static for the life of the accelerator (we can also **BYOIP**) => good for clients/firewalls that allow-list IPs
- Normal IP addresses are called **unicast** IP addresses, these refer to one device in the network
- In contrast anycast IP addresses can be used by multiple devices at the same time, the traffic will be routed to the device closest to the source
- Global Accelerator uses the AWS Edge Locations. Since multiple locations advertise the given anycast IP addresses, the traffic will be routed to the Edge Location closer to the user
- AWS has its own dedicated network consisting in fiber links. Edge Locations relay traffic to this network improving performance

## CloudFront vs Global Accelerator

- CloudFront moves the content closer to the customer by caching it on the Edge Location. Global Accelerator moves the customer closer to the service by providing access to the AWS global network
- Global Accelerator is a network product: works on any TCP/UDP applications including web apps (HTTP/HTTPS). CloudFront only caches HTTP/HTTPS content
- Global Accelerator does not cache anything. It does not understand the HTTP/HTTPS protocol
- Both integrate with AWS Shield for DDoS protection; only CloudFront integrates with AWS WAF directly (for Global Accelerator put WAF on the ALB endpoint)

| | Global Accelerator | CloudFront |
|---|---|---|
| Layer | L4 (TCP/UDP) | L7 (HTTP/HTTPS, WebSocket) |
| Caching | No | Yes |
| IPs | 2 static anycast IPs | Dynamic IPs, DNS name |
| Failover | Health-check based, near instant (no DNS TTL) | Origin failover (origin groups) |
| Typical use | Gaming, IoT, VoIP, MQTT, non-HTTP, static IP allow-listing, fast multi-Region failover | Static/dynamic web content, APIs, edge compute |

## Components

- **Accelerator**: standard or custom routing
- **Listener**: protocol (TCP, UDP or both) + port ranges
- **Endpoint group**: one per Region; has a **traffic dial** (0-100%) to control the % of traffic sent to that Region (blue/green, Region evacuation)
- **Endpoints**: ALB (internal or internet-facing), NLB, EC2 instances, Elastic IPs. Each endpoint has a **weight** (0-255)
- Endpoints can be in **other AWS accounts** (cross-account attachments) and can be private: GA can front an internal ALB or EC2 instances in private subnets (GA creates ENIs in the subnets, no IGW traffic needed but VPC must have an IGW attached)
- **Client affinity**: `NONE` (5-tuple) or `SOURCE_IP` (sticky to the same endpoint, stateful apps)
- **Client IP preservation**: supported for ALB and EC2 endpoints (and NLB in supported configurations); EIP endpoints do not preserve it

## Standard vs Custom Routing Accelerator

- **Standard**: traffic goes to the optimal healthy endpoint based on location, health, weights and traffic dials
- **Custom routing**: deterministic mapping of a listener port to a specific EC2 instance IP:port in a VPC subnet (no health-based routing). Use case: multiplayer game servers / VoIP sessions where our own matchmaking logic sends users to a specific instance

## Health Checks and Failover

- GA continuously health-checks endpoints (or uses ELB health checks for ALB/NLB endpoints)
- On failure, traffic moves to the next healthy endpoint/Region within seconds - **no DNS caching issue** since the IPs never change. Common exam differentiator vs Route 53 failover (bound by TTL and client caching)

## Pricing

- Fixed hourly fee per accelerator + **Data Transfer-Premium** fee (per GB, depends on source/destination Region pair, charged on the dominant direction)

## Exam Tips

- "Static IPs for a global app", "allow-list 2 IPs in a corporate firewall", "UDP global app", "fast regional failover without DNS" => Global Accelerator
- "Cache content", "HTTP edge logic", "signed URLs", "WAF at the edge" => CloudFront
- Accelerated Site-to-Site VPN uses Global Accelerator under the hood (TGW/Cloud WAN only)
- GA endpoints must be regional AWS resources - it cannot front on-premises servers directly (use an NLB with IP targets in AWS over DX/VPN)
