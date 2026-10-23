# ELB - Elastic Load Balancer

## ELB Architecture

- It is the job of the load balancer to accept connections from customers and distribute those connections to any registered backend compute
- ELBs support many different types of compute services
- LB architecture:
  ![LB Architecture](images/ELBArchitecture1.png)
- Initial configurations for ELB:
  - IPv4 or dual stack (IPv4 + IPv6)
  - We have to pick the AZs which the LB will use, specifically we are picking 1x subnet in 2 or more AZs
  - When we pick a subnet, AWS places one or more load balancer nodes in that subnet
  - When an LB is created, it has a DNS A record. This A record points to all the nodes provisioned for the LB => all the incoming connections are distributed equally
  - The nodes are HA: if the node fails, a different one is created. If the load is to high, multiple nodes are created
  - We have to decide on creation if the LB is internal or internet-facing.
- Load Balancers are configured with listeners which accept traffic on a port and protocol and communicate with the targets
- An internat facing load balancer can connect to both public and private instances
- Subnet size for an ALB: at least a **/27** with at least **8 free IP addresses** per subnet (older material says /28 - outdated); NLB needs at least 1 free IP per subnet (one node per AZ)
- ALBs scale by adding nodes (IPs change, use the DNS name - never hardcode ALB IPs); NLBs have **one static IP per AZ** (or an Elastic IP / your BYOIP)
- **LCU Capacity Reservation** (ALB/NLB): pre-provision a minimum capacity ahead of known traffic spikes (launches, events) - replaces the old "pre-warming" support ticket
- Internet-facing LB are same as internal LB but they additionally have public IP address assigned to their nodes. BUT nodes are deployed the same way.

## Cross-Zone Load Balancing

- An LB by default has at least one node per AZ that it is configured for
- Historically, each LB node could **ONLY** distribute connections to instances in the same AZ -** Cross-Zone Load Balancing:** allows any LB node to distribute connections equally across all registered instances in all AZs.
- This help with the uneven distribution of load and could be helpful in <span style="color: #ff5733;">EXAM</span>
- ALB: ON by default at the LB level, **no charge** for inter-AZ traffic; since 2022 it can be turned off per **target group**
- NLB and GWLB: OFF by default, can be enabled (LB or target group level) - **inter-AZ data transfer is charged** when enabled
- CLB: ON by default when created from the console, OFF when created via API/CLI
- Turning cross-zone OFF keeps traffic zonal (AZ affinity, cost) and is recommended for **zonal shift** (Application Recovery Controller) so an impaired AZ can be fully isolated; zonal shift / zonal autoshift are supported for ALB and NLB
- With cross-zone OFF, make sure each AZ has enough healthy targets (uneven targets per AZ => uneven load)

![CROSS-ZONE LB Architecture](images/ELBArchitecture2.png)

## User Session State

- Session state:
  - A piece of server side information specific to one single user of one application
  - It does persist while the user interacts with the application
  - Examples of session state: shopping cart, workflow position, login state
- The data representing a sessions state is either stored internally or externally (stateless applications)
- Externally hosted session:
  - Session data is hosted outside of the back-end instances => application becomes stateless
  - Offers the possibility to do load balancing for the back-end instances, the session wont get lost in case the LB redirects the user to a different instance

## ELB Evolution

- Currently there are 4 different types of LB in AWS: CLB, ALB, NLB and GWLB
- Load balancers are split between v1 and v2 (always preferred)
- Classic Load Balancer (v1)

  > **Status (2026):** CLB is a previous-generation product (EC2-Classic networking was retired in Aug 2022). Still supported in VPCs but gets no new features - use the migration wizard to move to ALB/NLB.

  - CLBs can load balance HTTP/HTTPS and lower level protocols as well, although they can not understand the http protocol, they can't make decision based on HTTP protocols features
  - CLBs can have **only 1 SSL certificate** per load balancer
  - They can not be considered entirely a layer 7 product
  - We should default to using v2 load balancer for newer deployments
- Version 2 (v2) load balancers:
  - Application Load Balancer (ALB): layer 7 devices, they support HTTP(S) and WebSocket protocols
  - Network Load Balancer (NLB): supports lower level protocols such as TCP, TLS and UDP. These could be used for applications like Email servers, Games or applications which don't use HTTP/s protocols.
  - Gateway Load Balancer (GWLB): layer 3, GENEVE, for inline virtual appliances (see [GWLB](gwlb.md))
- In general v2 load balancers are faster and they support target groups and rules, this allow to use single LB for multiple things.

## ALB vs NLB

- Consolidation of load balancers:
  - Classic load balancers do not scale, they do not support multiple SSL certificates (no SNI support) => for every application a new load balancer is required
  - V2 load balancers support rules and target groups
  - V2 load balancers can have host based rules using SNI => for ex you could host two websites example1.com and api.example2.com on the same ALB, so hosted on same IP + port. In R53, 2x records pointing to ALB, and then configured on the ALB to route traffic differently based on hostname (Host Rule).
- **Application Load Balancer (ALB)**:

  - ALB is a true layer 7 load balancer, configured to listen to either HTTP or HTTPS protocols
  - Supports HTTP/1.1, **HTTP/2**, **gRPC** (target group protocol version) and WebSockets
  - ALB can not understand any other layer 7 protocols (such as SMTP, SSH, etc.) and layer 4 (TCP, UDP, TLS)
  - ALB requires HTTP and HTTPS listeners
  - It can understand layer 7 content, such as cookies, custom headers, user location, app behavior, etc.
  - Any incoming connection (HTTP, HTTPS) is always terminated on the ALB - no unbroken SSL - followed by new connection to the app
  - All ALBs using HTTPS must have SSL certificates installed
  - ALBs are slower than NLBs because they require more levels of networking stack to process. Any <span style="color: #ff5733;">EXAM</span> question which talks about performance, NLB should be considered instead of ALB.
  - ALBs offer health checks evaluation at application layer.
  - ALB Rules:
    - Rules direct connection which arrive at a listener
    - Rules are processed in a priority order, default rule being a catch all (last one to be processed)
    - Rule conditions: host-header, http-header, http-request-method, path-pattern, query-string and source-ip
    - Rule actions: forward, redirect, fixed-response, authenticate-oidc and authenticate-cognito
    - `forward` can use **weighted target groups** (blue/green, canary) with optional target group stickiness
    - Default quota: 100 rules per ALB (adjustable), up to 25 certificates per HTTPS listener via **SNI** (adjustable), ACM or IAM certs
  - Target types: `instance`, `ip` (incl. on-premises IPs over DX/VPN, peered VPCs), `lambda` (Lambda function as target)
  - Routing algorithms: round robin (default), **least outstanding requests**, weighted random with **Automatic Target Weights** (anomaly mitigation - shifts traffic away from targets returning errors)
  - **Slow start mode**: linearly ramps up traffic to newly registered targets
  - Security:
    - **AWS WAF** integration (ALB only, not NLB), Shield Advanced; TLS security policies (incl. TLS 1.3, post-quantum hybrid policies)
    - **Authentication** (OIDC / Cognito user pools) offloaded to the ALB for browser users
    - **JWT verification** (Nov 2025): ALB validates bearer tokens (signature, `iss`, `exp`, extra claims) for machine-to-machine / OAuth client-credentials flows - no app code
    - **Mutual TLS (mTLS)** (2023): `verify` mode (ALB validates client certs against a **trust store** of CA bundles in S3 + optional CRLs) or `passthrough` mode (ALB forwards the client cert chain in `X-Amzn-Mtls-Clientcert-*` headers to the targets)
  - Header modification (add/rename/remove response headers like HSTS, CORS), `X-Amzn-Trace-Id` added for tracing (X-Ray)
  - Desync mitigation mode (HTTP request smuggling) - `defensive` by default
  - Access logs to S3 (optional), connection logs (client cert info for mTLS), health check logs
  - Internet-facing ALB behind CloudFront: restrict with the CloudFront managed prefix list in the SG + custom header; or use **CloudFront VPC origins** with an internal ALB
  - The connection from the LB and the instance is a separate connection
  - If you need to forward connections without terminating on the LB, then you need to consider NLB. (<span style="color: #ff5733;">EXAM</span>)

- **Network Load Balancer (NLB)**:

  - NLBs are layer 4 load balancers, meaning they support TPC, TLS, UDP, TCP_UDP connections
  - They have no understanding of HTTP or HTTPS
  - No headers, no cookies; stickiness only based on **source IP** (target group attribute)
  - They are really fast, can handle millions of request per second with ultra-low latency (no L7 processing) and handle sudden spikes without pre-warming
  - Recommended for SMTP, SSH, game servers, financial apps (not HTTP(S)) <-- <span style="color: #ff5733;">EXAM</span>
  - Health checks: TCP, HTTP or HTTPS (the HTTP(S) check can be app aware, e.g. `/health`), but traffic routing itself is not app aware. ICMP is **not** supported
  - They can be allocated with static IP addresses (1 per AZ, Elastic IPs for internet-facing, or BYOIP) which is useful for whitelisting for corporate clients
  - They can forward TCP straight through the instances => unbroken encryption <-- <span style="color: #ff5733;">EXAM</span>
  - They can also **terminate TLS** (TLS listener with ACM certs, SNI, ALPN) => offload TLS while keeping L4
  - NLBs can be used for PrivateLink <-- <span style="color: #ff5733;">EXAM</span>
  - **Security groups** on NLB supported since Aug 2023 (must be associated at creation; previously NLBs had no SGs and targets saw client IPs directly)
  - **Client IP preservation**: on by default for `instance` targets (and UDP), off by default for `ip` targets with TCP/TLS => use PROXY protocol v2 if needed
  - Target types: `instance`, `ip`, `alb` (**ALB as target of NLB** => static IPs/PrivateLink in front of L7 routing)
  - **QUIC passthrough** (Nov 2025): `QUIC` / `TCP_QUIC` listeners, stickiness via QUIC Connection ID (mobile clients changing IPs)
  - Zonal DNS names per AZ, DNS failover setting (`dns_record.client_routing_policy`) to keep clients AZ-affine
  - Global Accelerator in front of NLB/ALB for global static anycast IPs

- **Scenarios for NLB (otherwise => default to ALB)**:
  - Unbroken encryption
  - Static IP for whitelisting
  - Fast performance
  - Protocols other tha HTTP or HTTPS
  - Privatelink
  - Non-HTTP protocols with TLS offload, UDP, QUIC, or millions of connections / sudden spikes
- **Scenarios for ALB**: path/host/header based routing, microservices/containers (dynamic port mapping with ECS), Lambda targets, WAF, OIDC/Cognito auth, JWT validation, mTLS with header passthrough, redirects HTTP->HTTPS, fixed responses

## Session Stickiness

- Stickiness: allows us to control which backend instance to be used for a given connection
- With no stickiness => connections are distributed across all backend services
- Enabling stickiness:
  - CLB: we can enable it per LB
  - ALB: we can enable it per target group
  - NLB: per target group, based on source IP (no cookies)
- ALB cookie types:
  - **Duration-based**: LB-generated cookie `AWSALB` (and `AWSALBCORS`)
  - **Application-based**: custom cookie set by the target, ALB adds `AWSALBAPP`
- When stickiness is enabled, the LB generates a cookie: `AWSALB` for ALB / `AWSELB` for CLB which is delivered to the end-user
- This cookie has a duration defined between 1s and 7 days
- When the user accesses the LB, it provides the cookie to the LB
- The LB can then decide to route the connection to the same backend instance every time while the cookie is not expired
- Change of the backed instance if the cookie is present:
  - If the instance to which the cookie maps to fails, then a new instance will be selected
  - If the cookie expires => the cookie will be removed, new cookie is created while a new instance is chosen
- Session stickiness problems: load can become unbalanced (with heavy users)
- Enable session stickiness if an application does **NOT** use external sessions

## Connection Draining and Deregistration Delay

- **Connection draining (CLB)**
  - Setting which controls what happens when instances are unhealthy or deregistered
  - Default behavior: LB closes all connections and the instance receives no new connections
  - Allows in-flight requests to complete for a certain amount of time, while no new connections are sent to the instance
  - Only supported on **Classic Load Balancers**. It is defined on the load balancer itself
  - Connection draining is a timeout between 1 and 3600 seconds (default 300s)
  - If the instance become unhealthy because if a failed health check, connection draining settings do not apply to it
  - If an instance is taken out of service manually or by an ASG, it is listed "InService: Instance deregistration currently in progress". If we use an ASG, it will wait for all connections to complete before terminating or for the timeout value
- **Deregistration delay (ALB, NLB, GWLB)**
  - Essentially the same feature as connection draining, but it is supported by ALB, NLB and GWLBs
  - It is defined on target groups, not on the LB
  - It works by stopping sending connections to deregistering targets. Existing connections can continue until they complete naturally or the deregistration delay is reached
  - Deregistration delay is enabled by default on all the new LBs, default value: 300s (configurable between 0-3600 seconds)

## `X-Forwarded-For` and PROXY protocol

- In case a client connects to a backend without any load balancing in the front of the backend, the IP address of the client is visible and can be recorded
- With load balancers this can be more complicated, this is where `X-Forwarded-For` header and the PROXY protocol become handy
- `X-Forwarded-For` is a HTTP header, it only works with HTTP/HTTPS. This is a layer 7 header.
  - ALB also adds `X-Forwarded-Proto` and `X-Forwarded-Port`; XFF processing modes: `append` (default), `preserve`, `remove`
  - This header is added/appended by proxies/load balancers. It can have multiple values in case the request is passing multiple proxies/load balancers. E.g `X-Forwarded-For: 1.3.3.7(ClientIP), proxy1, proxy2, ...` => client is the element on the left
  - The backend server needs to be aware of this header and needs to support it
  - Supported on CLB and ALB, NLB does not supports it because they don't support the layer 7 of the OSI stack.
- **PROXY protocol** works at Layer 4, it is an additional layer 4 (tcp) header => works with a wide range or protocols (including HTTP/HTTPS)

  - There are 2 versions of PROXY protocol:
    - v1: human readable, works with CLB
    - v2: binary encoded, works with NLB
      - v2 can support an unbroken HTTPS connection (tcp listener). Use case for this: end to end encryption
  - With unbroken HTTPS => headers are encrypted, only possible to use PROXY protocol

## ELB Monitoring and Pricing

- CloudWatch metrics (1 min): ALB `RequestCount`, `TargetResponseTime`, `HTTPCode_Target_5XX_Count`, `HTTPCode_ELB_5XX_Count`, `RejectedConnectionCount`, `SurgeQueueLength`/`SpilloverCount` (CLB); NLB `ActiveFlowCount`, `NewFlowCount`, `TCP_Client_Reset_Count`; `HealthyHostCount` / `UnHealthyHostCount`
- 502 = bad gateway (target closed connection / bad response), 503 = no registered/healthy targets, 504 = target timeout (check idle timeout, default 60s on ALB; NLB TCP idle timeout 350s, configurable)
- Pricing: hourly + **LCU** (ALB: new connections, active connections, processed bytes, rule evaluations - highest dimension billed; NLB: NLCU per TCP/UDP/TLS flows and bytes)
- Health check failure on all targets => ALB/NLB "fails open" and routes to all targets
- Integrations: ACM, WAF (ALB), Shield, Global Accelerator, CloudFront, Route 53 alias records, PrivateLink (NLB/GWLB), ECS/EKS (AWS Load Balancer Controller), VPC Lattice (alternative for service-to-service)

## EC2 Advanced Demo

![EC2Demo](images/EC2Demo.png)
