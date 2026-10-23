# CloudFront

- It is a content delivery network (CDN)
- Its job is to improve the delivery of content from its original location to the viewers of the content
- It is accomplishing this by caching and by using an efficient global network

## CloudFront Terms and Architecture

- **Origin**: the source location of the content, can be S3 or custom origin (publicly routable endpoint, IPv4 or IPv6 since 2025). Private ALB/NLB/EC2 in private subnets can be reached using **VPC origins** (since Nov 2024)
- **Distribution**: unit of configuration within CloudFront, which gets deployed out to the CloudFront network. Almost everything is configured within the distribution directly or indirectly
- **Edge Location**: pieces of global infrastructure where the content is cached. They are smaller than AWS regions. They are way bigger in number than AWS locations and more widely distributed (600+ PoPs). They serve static and dynamic content, and also accept uploads (`PUT`/`POST`) which are proxied to the origin
- **Regional Edge Cache**: larger version of an edge location, but there are fewer of them. Provides another layer of caching
- CloudFront Architecture:
  ![CloudFront Architecture](images/CloudFrontArchitecture1.png)
- Requests that skip the Regional Edge Cache (REC) and go directly from the edge to the origin:
  - Proxy methods: `PUT`, `POST`, `PATCH`, `OPTIONS`, `DELETE`
  - Dynamic requests (determined at request time, e.g. not cacheable)
  - S3 origin requests when the optimal REC is in the **same region as the bucket**
  - Old notes stated "S3 origins never use the REC" - this is no longer accurate
- **Origin Shield**: optional extra centralized caching layer in front of the origin (choose the region closest to the origin). Increases cache hit ratio, collapses requests from all RECs into one => reduces origin load. Good for multi-CDN or expensive/fragile origins. Extra cost
- **Origin fetch**: the content is fetched from the origin in case of a cache miss on the edge location
- **Behavior**: it is configuration within a distribution. Origins are directly linked to behaviors, behaviors are linked to distributions
  ![CloudFront Behavior](images/CloudFrontArchitecture2.png)

## CloudFront Behaviors

- Distributions are units of configuration in CF, lots of high level options are configured on the distribution level:
  - Price class
  - Web Application Firewall attachment
  - Alternate domain names
  - Type of SSL certificate
  - SNI configuration
  - Security policy
  - Supported HTTP versions
  - etc.
- A single distribution can have one (default behavior) or multiple behaviors
- Any incoming request is pattern matched against behavior's pattern
- The default behavior has a wildcard (`*`) pattern and it will match anything that was not matched by other more specific behavior
- Once a request is pattern matched against a behavior, it will become subject ot the behavior's configurations which can be the following:
  - Origin or origin group
  - Viewer protocol policy (redirect HTTP to HTTPS)
  - Allowed HTTP methods
  - Field level encryption
  - Cache directives - we can use:
    - Legacy cache settings
    - Cache policy and origin request policy (recommended by AWS)
  - Response headers policy: add/remove headers sent to viewers (CORS, HSTS, security headers) without code
  - TTL (min, max, default)
  - Restrict viewer access to a behavior: sets the entire behavior to restricted or private. If we select this , we need to specify the trusted authorization type, which can be:
    - Trusted key groups (recommended by AWS)
    - Trusted signer (legacy)
  - Compress objects automatically
  - Associate Lambda@Edge function or CloudFront Function

## TTL and Invalidations

![TTL and Invalidations](images/CloudFrontTTL.png)

- And edge location views an object as not expired when it is within its TTL period
- More frequent cache hits = lower origin loads
- Default validity period of an object (TTL) is 24 hours (default max TTL: 1 year). This is defined in the behavior (cache policy)
- Minimum TTL, maximum TTL: set lower or upper values which an individual object's TTL can have
- Object specific TTL values can be set by the origins using different headers:
  - Cache-Control `max-age` (seconds): TTL value in seconds for an object
  - Cache-Control `s-maxage` (seconds): same as `max-age`
  - Expires (Date and Time): expiration date and time
  - `stale-while-revalidate` / `stale-if-error` (supported since 2023): serve stale content while refreshing in the background or when the origin returns errors
- For all of these headers if they specify a value outside of minimum, maximum range, the min/max value will be used
- Custom headers for S3 origins can be configured in object's metadata
- Cache invalidations are performed in a distribution and it applies to all edge locations (it takes time)
- Cache invalidation invalidates every object regardless of the TTL value, based on the invalidation pattern
- Invalidation pricing: first 1,000 paths per month are free, after that a charge per path. A wildcard path (`/images/*`) counts as **one path** regardless of how many objects it matches
- Limits: up to 3,000 individual file paths or 15 wildcard paths in progress at one time per distribution
- Instead of invalidation we may consider **versioned file names**
- Versioned file names also help to:
  - Avoid using local browser cache in case of a newer file
  - Help improve logging
  - Reduce cost, no need for manual invalidation
- S3 object versioning and versioned file names should not be confused!

## CloudFront and SSL

- Each CF distribution receives a default domain name (CNAME)
- HTTPS can be enabled by default for this address
- CF allows alternate domain names (CNAME)
- Process of adding alternate domain names:
  - If we use HTTPS, we need a certificate applied to the distribution which matches that name
  - Even if we don't want to use HTTPS, we need a way verifying that we own and control the domain. This is accomplished by adding an SSL certificate which matches the the name we are adding to the CF distribution
  - The result is we need to add an SSL certificate wether we are using HTTPS or not
- SSL certificates are imported using ACM (AWS Certificate Manager)
- ACM is a regional service, because of this the certificate for global services (such as CF) needs to be imported in _us-east-1_ region
- Option we can set on a CF behavior for handling HTTP and HTTPS:
  - We can allow both HTTP and HTTPS on a distribution
  - We can redirect HTTP to HTTPS
  - We can restrict to only allow HTTPS (any HTTP will fail)
- There are two sets of connections when any individual is using CF:
  - Viewer => CF (viewer protocol)
  - CF => Origin (origin protocol)
- Both connections need valid public certificates as well as any intermediate certificates. Self-signed certificates will not work!
- If our origin is S3, we don't have to worry about this certificate for the origin protocol. S3 handles this natively on it own. We don't/can't apply certificates to S3 buckets

## CloudFront and SNI (Server Name Indication)

- Historically every SSL enabled site needed its own IP
- Encryption for HTTP/HTTPS happens on the TCP connection level
- Host header happens after that at Layer 7. It allows to specify to which application we want to connect in case multiple applications run on the same server
- TLS encryption happens before deciding which application we want to access
- In 2003 an extension was added to TLS: SNI - allowing to specify which domain we want to access. This occurs in the TLS handshake, before HTTP being involved
- This allows one server with a single IP to host many HTTPS websites which need their own certificates
- Older browser do not necessary support SNI. CF needs to allocate dedicated IP addresses for these users, at extra charge
- CF can be used in SNI mode (free) or allocating extra IP addresses ($600 per month per distribution)
- CloudFront SSL/SNI architecture:
  ![SSL/SNI architecture](images/CloudFrontSSLSNI.png)
- For S3 origin, we don't need to apply certificates for the origin protocol. For ALB/EC2/on-prem we can have public certificates which needs to match the DNS name of the origin

## Origin Types and Architecture

- Origins are the locations from where CF goes to get content
- If there is a cache miss in case of a request, than an origin fetch occurs
- Origin groups allow us to add resiliency. We can group origins together an have an origin group used by the behavior
- Categories of origins:
  - Amazon S3 buckets
  - AWS Elemental MediaPackage channel endpoint (v2 supports OAC)
  - AWS Elemental MediaStore container endpoint
    > **Status (2026):** AWS Elemental MediaStore was discontinued on Nov 13, 2025. Use S3 (simple workflows) or MediaPackage (advanced/low-latency) as origin
  - Lambda function URLs (support OAC since 2024)
  - VPC origins: private ALB, NLB or EC2 instance (no public IP/IGW-facing endpoint needed); shareable across accounts with AWS RAM (since Nov 2025)
  - everything else (web-servers) - custom origins
- If S3 is configured to be used as a web-server, CF views it as a custom origin
- S3 origin configuration options:
  - Origin Path: we can use a path from the bucket instead of the top level of the bucket
  - Original access control settings: it is used to restrict access to the bucket only to CloudFront. The legacy version of this was Origin Access Identity
  - Origin Access Identity (legacy): same purpose as the origin access control
  - Add custom headers (optional): we can pass custom headers to the origin S3 bucket
- In case of S3 the viewer protocol is matched with the origin protocol. This means if we use HTTP for the end-users, CF will also use HTTP to access the bucket
- Exam trap: S3 **static website endpoint** is a custom origin => HTTP only to the origin and **no OAC/OAI** (restrict with a custom header + bucket policy `aws:Referer` trick or switch to the REST endpoint + OAC)
- Custom origin configuration options:
  - Origin Path: we can configure to use a sub-path to access the origin
  - Minimum Origin SSL Protocol: minimum TLS protocol version to be used with the origin. Best practice is to select the latest supported by the origin
  - Origin Protocol Policy: HTTP only, HTTPS only or Match Viewer protocol policy
  - HTTP/HTTPS Port: we can use arbitrary port instead of 80 or 443 for being able to connect to the origin
  - Origin Custom Headers: pass custom headers to the origin. Can be used for security to restrict access only from CF
  - Timeouts: connection timeout, connection attempts, response (read) timeout, keep-alive timeout, response completion timeout (2025)

### Origin Groups and Origin Failover

- Origin group = primary + secondary origin, attached to a behavior
- Failover happens when the primary returns configured status codes (e.g. 500, 502, 503, 504, 403, 404) or connection/timeouts
- Failover is per request, only for `GET`, `HEAD`, `OPTIONS` (cacheable methods) - not for writes
- Typical pattern: S3 bucket in region A + replicated bucket (CRR) in region B => regional resilience for static content
- Media quality-aware resiliency (MQAR, 2024): picks the origin with the best media quality score (MediaPackage v2)
- Not the same as Route 53 failover: origin groups react per request, at the edge, with no DNS TTL

## Caching Performance and Optimization

- Cache Hit: object is available in the cache in the edge location
- Cache Miss: object is not available in the cache, origin fetch is required
- To increase performance we need the maximize the ration between cache hit and cache miss
- We can retrieve objects from CF based on these:
  1. When we require an object from CF, we usually request it using its name
  2. We can use query string parameters as well, example `index.html&lang=en`
  3. Cookies
  4. Request Headers
- When using CF all of this data reaches CloudFront first and than can be forwarded to the origin
- We can configure CF to cache data based on some or all of these request properties
- These choices affect how performant would be the data retrieval from our CF distribution
- Optimization recommendations:
  - When using CF we should forward only the headers needed by the application and cache data based only on what can change the object
  - The more things are involved in caching, the less efficient the process is
- **Cache key** vs **origin request**:
  - Cache policy: defines the cache key (which query strings/headers/cookies are part of it) + TTLs + compression (Gzip/Brotli)
  - Origin request policy: what is forwarded to the origin **without** being part of the cache key
  - Values included in the cache key are automatically forwarded to the origin
  - Managed policies: `CachingOptimized`, `CachingDisabled`, `UseOriginCacheControlHeaders`, `AllViewer`, `AllViewerExceptHostHeader`, `HostHeaderOnly`, etc.
- Other optimizations: normalize/sort query strings with CloudFront Functions, Origin Shield, longer TTLs + versioned file names, enable compression, HTTP/2 and HTTP/3

![CloudFront - Caching 1](images/Caching1.png)

![CloudFront - Caching 2](images/Caching2.png)

![CloudFront - Caching 3](images/Caching3.png)

## CloudFront Security

### OAI/OAC and Custom Origins

- S3 origins:
  - OAI - Origin Access Identity (legacy):
    - Is a type of identity, it can be associated with CloudFront distributions
    - Essentially the CloudFront distributions "becomes" the OAI, meaning that this identity can be used in S3 bucket policies
    - Common pattern is to lock the S3 bucket to be only accessible to CloudFront
    - The edge locations gains the attached OAI identity, meaning they will be able to access the bucket
    - Direct access from the end-user to the bucket content can be disabled with a bucket policy
    - OAIs can be created and used one many CF distributions and many buckets at the same time. It is easier to manage one OAI with one CF distribution

    ![CloudFront - S3 Origin Security](images/OAISecurity.png)

  - OAC - Origin Access Control (recommended):
    - Used for the same purpose as OAI - restrict access to bucket to CF only
    - If enabled CF will sign each request to S3 bucket
    - Once enabled, we will have to adjust the bucket policy to allow request from the CF distribution

  - OAC vs OAI (exam):
    - OAC supports SSE-KMS encrypted objects, all regions (incl. opt-in regions launched after 2022), dynamic requests (`PUT`, `DELETE`) and uses SigV4 with short-term credentials
    - OAI does **not** support SSE-KMS => typical exam answer: "migrate from OAI to OAC"
    - Bucket policy for OAC: principal `cloudfront.amazonaws.com` + condition `AWS:SourceArn` = distribution ARN
    - OAC also works for Lambda function URLs, MediaPackage v2 (and MediaStore while it existed)

- Custom origins:
  - We can not use OAI/OAC to control access to generic HTTP origins (except Lambda function URLs)!
  - Prefer **VPC origins**: origin stays in a private subnet, only reachable from CloudFront => strongest option for ALB/NLB/EC2
  - We can utilize custom headers, which will be protected by the HTTPS protocol. CloudFront will be configured to send this custom header
  - Other way to handle CloudFront security from custom origins is to determine the IP ranges from which the request is coming from. CloudFront IP ranges are publicly available
  - Use the AWS-managed prefix list `com.amazonaws.global.cloudfront.origin-facing` in security groups instead of maintaining IP lists
  - Common combination: SG with prefix list + secret custom header checked by ALB listener rule/WAF (rotate the header with Secrets Manager)

  ![CloudFront - Custom Origin Security](images/OriginSecurity.png)

### Private Distributions

- CloudFront can run in 2 different modes:
  - Public: can be accessed by any viewer
  - Private: requests to CloudFront needs to be made with a signed url or cookie
- If the CloudFront distribution has only 1 behavior the whole distribution is considered to be either public or private
- In case of multiple behaviors: each behavior can be either public or private
- There are 2 ways two configure private behaviors in CF:
  - The old way: in order to enable private distribution of content, we need to create a **CloudFront Key** by an Account Root User. That account is added as a **Trusted Signer**
  - The new (preferred) way:
    - Create Trusted Key Groups and assign them signers
    - They key groups determine which keys can be used to create signed urls and signed cookies
    - Few reasons we might use this compared to the legacy approach:
      - We don't need the root user from the account to manage CF keys
      - We can manage keys groups with CF API and we can associate a higher number of keys with our distribution/behavior giving us more flexibility
- Signed URLs vs Cookies:
  - Signed URLs provide access to one particular object
  - We should use signed urls if the client does not support cookies
  - Signed cookies can provide access to groups of objects or all files of a particular type
  - With signed cookies we can maintain the application's URL if this is important
- Canned policy vs custom policy:
  - Canned: one object, only expiration time
  - Custom: wildcards in resource path, start time, expiration, source IP range; reusable for multiple objects
- Key groups: up to 5 public keys per key group, up to 4 key groups per behavior. Keys are created with IAM permissions, no root user needed
- Signed URLs/cookies protect access through CloudFront; to stop bypass, also lock the origin (OAC/VPC origin)
- CloudFront signed URLs vs S3 presigned URLs:
  - CloudFront: path patterns, IP restriction, cached at edge, key group based
  - S3 presigned: single object, uses IAM credentials of the signer, expires with the credentials, goes directly to S3

![Private Distributions](images/PrivateDistribution.png)

### CloudFront Geo Restriction

- Gives a way to restrict content to a particular location
- They are 2 types of restriction:
  - CloudFront Geo Restriction:
    - Whitelist or Blacklist countries
    - **Only works with countries!**
    - Uses a GeoIP database with 99.8% claimed accuracy
    - Applies to the entire distribution
      ![Geo Restriction Architecture](images/CloudFrontGeoRestriction.png)
  - 3rd Party Geolocation:
    - Completely customizable, can be used to filter on lots of other attributes, example: username, user attributes, etc.
    - Requires an application server in front of CloudFront, which controls weather the customer has access to the content or not
    - The application generates a signed url/cookie which is returned to the browser. This can be sent to CloudFront for authorization
      ![3rd Party GeoLocation Architecture](images/CloudFront3rdPartyGeoLocation.png)

### Field-Level Encryption

- We can configure encryption ath the edge location for certain fields from the request using a public key
- Useful for encrypting sensitive data such as passwords, payment information, etc. at the edge locations
- Field-Level encryption happens separately from the HTTPS tunnel
- A private key is needed to decrypt individual fields
- Decryption of the encrypted fields can be done at the origin, if necessary
- Works only with `POST` requests, up to 10 fields, RSA 2048 public key
- Field-Level encryption architecture:
  ![Field-Level encryption architecture](images/FieldLevelEncryption2.png)

## Lambda@Edge

- Lambda@Edge allows us to run lightweight Lambda functions at the edge locations
- These Lambda functions allow us to adjust data between the viewer and the origin
- Lambda functions running at the edge don't have the full Lambda feature set:
  - Currently only NodeJS and Python are supported
  - Functions don't have access to any resources in a VPC, they run in AWS public space
  - Lambda Layers are not supported
- They have different size and duration limits compared to classic Lambda functions:
  - Viewer side: 128 MB limit in size / function timeout is 5 seconds
  - Origin side: function size is the same as classic Lambda / function timeout is 30 seconds
- Lambda@Edge use cases:
  - A/B testing - generally done with Viewer Request function. Lambda function can view the request from the viewer and can modify the response accordingly
  - Migration between S3 origins - generally done with an Origin Request function
  - Different objects based on the type of device - generally done with an Origin Request function
  - Content displayed by country - generally done with an Origin Request function
  - More examples: [https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-examples.html#lambda-examples-redirecting-examples](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-examples.html#lambda-examples-redirecting-examples)

![Lambda@Edge](images/LambdaAtEdge.png)

- Lambda@Edge details:
  - Functions are authored in **us-east-1** and replicated to the regional edge caches (they run in RECs, not in every PoP)
  - Memory: viewer triggers 128 MB, origin triggers up to 10,240 MB; max package size 50 MB
  - Supported runtimes: Node.js and Python only; no VPC, no layers, no environment variables, no ARM64, no provisioned concurrency
  - Can make network calls (e.g. to DynamoDB, S3, external APIs) and read the request body
  - Logs go to CloudWatch Logs in the region where the function ran (advanced logging controls since 2025)
- 4 trigger points: Viewer Request, Origin Request, Origin Response, Viewer Response

## CloudFront Functions

- Lightweight JavaScript (ECMAScript 5.1 + runtime 2.0 features) running in **every edge location** (PoP)
- Only **Viewer Request** and **Viewer Response** triggers
- Sub-millisecond execution, < 1 ms compute, 2 MB memory, 10 KB code size, no network access, no body access
- Millions of requests per second, ~1/6 of the cost of Lambda@Edge
- Use cases: header manipulation, URL rewrites/redirects, cache key normalization, JWT/token validation, simple A/B tests, origin selection (`selectRequestOriginById()`, `createRequestOriginGroup()`)
- **CloudFront KeyValueStore** (2023): global low-latency key/value data store readable from CloudFront Functions (redirect maps, feature flags, A/B config) - update data without redeploying the function
- **Connection Functions** (Nov 2025): run on the TLS connection for mutual TLS (viewer) - custom certificate validation logic

### CloudFront Functions vs Lambda@Edge

| | CloudFront Functions | Lambda@Edge |
|---|---|---|
| Runs at | Edge locations (PoPs) | Regional edge caches |
| Triggers | Viewer request/response | Viewer + origin request/response |
| Runtime | JavaScript | Node.js, Python |
| Max execution | < 1 ms | 5 s (viewer) / 30 s (origin) |
| Network / body access | No | Yes |
| Typical use | Header/URL manipulation, auth tokens | Calls to external services, origin selection on complex logic, image transformation, body inspection |

- Exam rule: simple, high-volume viewer manipulation => CloudFront Functions; needs network/body/longer runtime/origin events => Lambda@Edge

## Continuous Deployment

- Create a **staging distribution** linked to the primary distribution and route a subset of traffic to it
- Traffic routing: weight-based (up to 15%, optional session stickiness) or header-based (specific header => staging)
- Promote staging config to primary when validated => safe blue/green for CDN configuration changes
- Use it to test distribution config changes (behaviors, cache policies, functions, origins) with real traffic

## Security and Protection Features

- **AWS WAF** web ACL attached at distribution level (WAF for CloudFront is a global resource created in us-east-1)
- **AWS Shield Standard** included automatically; Shield Advanced for L7 DDoS response team, cost protection
- **Security dashboard** (2023): enable WAF, geo restrictions, bot control from the CloudFront console
- **Mutual TLS (viewer)** (Nov 2025): clients authenticate with certificates (IoT, B2B APIs); also mTLS towards origins
- Security policies: minimum TLS version for viewers, e.g. `TLSv1.2_2021`, `TLSv1.2_2025`, `TLSv1.3_2025`; post-quantum key exchange supported
- **Anycast static IPs** (2024): fixed set of IPs for the distribution (allow-listing by firewalls, zero-rated carrier traffic, apex domains); IPv4 or dual-stack
- **BYOIP** via IPAM (Nov 2025)
- Logging: standard logs (v2) to S3, CloudWatch Logs or Data Firehose; real-time logs to Kinesis Data Streams
- Config rules and Security Hub controls available for CloudFront

## Pricing and Other Features

- **Price classes**: All edge locations, Price Class 200 (excludes most expensive regions), Price Class 100 (North America, Europe, Israel only) => cost vs latency trade-off
- **Flat-rate pricing plans** (Nov 2025): Free / Pro / Business / Premium monthly bundles including CDN, WAF, DDoS protection, Route 53 DNS, logs, edge compute, S3 credits, no overages; API/CloudFormation support since Sep 2026
- **CloudFront Security Savings Bundle**: 1-year commitment, up to 30% savings, includes WAF credits
- **Multi-tenant distributions / CloudFront SaaS Manager** (2025): one template distribution + many distribution tenants (per-customer domains/certs) - SaaS use case
- Protocols: HTTP/2, HTTP/3 (QUIC), WebSocket, gRPC (2024)
- Data transfer from AWS origins to CloudFront is free => putting CloudFront in front of S3/ALB often reduces cost

## Exam Tips

- Global content + caching + DDoS protection => CloudFront (+ WAF + Shield)
- Static IPs or non-HTTP (TCP/UDP) acceleration => **Global Accelerator**, not CloudFront (unless Anycast static IPs fit the case)
- Restrict S3 to CloudFront => OAC (not OAI, especially with SSE-KMS)
- Restrict ALB to CloudFront => VPC origins (best) or prefix list + secret custom header
- Private content for logged-in users => signed cookies (many files) or signed URLs (single file / no cookies)
- Country-level blocking => geo restriction; more granular rules => WAF geo match / Lambda@Edge / app-generated signed URLs
- Regional failover for static sites => origin group with two S3 buckets (CRR)
- ACM certificate for CloudFront must be in **us-east-1**; for ALB origin, the cert is in the ALB's region
- Lowering origin load with many RECs => Origin Shield
