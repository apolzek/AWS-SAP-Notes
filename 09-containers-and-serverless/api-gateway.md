# API Gateway

- Service which lets us create and manage APIs
- API Gateway acts as endpoint or entry-point for applications that want to talk with our services
- Sits between the application and integrations (services)
- API Gateway is HA and scalable
- Handles authorization, throttling, caching, CORS, transformations
- It also supports the OpenAPI spec and direct integration with other AWS services
- API gateway is a **public service** (except Private REST APIs, reachable only via `execute-api` interface endpoints)
- It can provide APIs using REST, HTTP and WebSocket
- API Gateway overview:
  ![API Gateway Architecture](images/APIGateway.png)

## Authentication

- API Gateway supports a range of authentication types such as Cognito, Lambda based authentication (Custom based authentication - we can assume the client uses a Bearer token) and IAM credentials
- We can allow APIs to be open access without authentication
- Full list of options (REST API):
  - **IAM** (SigV4): best for AWS-internal / cross-account callers; combine with resource policy for cross-account
  - **Cognito user pool authorizer**: validates Cognito JWTs natively
  - **Lambda authorizer**: `TOKEN` (bearer/JWT/OAuth) or `REQUEST` (headers, query string, stage vars, context); returns an IAM policy, result can be cached (TTL up to 1h)
  - **Resource policy**: allow/deny by AWS account, source IP/CIDR, VPC / VPC endpoint (mandatory for Private APIs)
  - **API keys + usage plans**: for metering/throttling clients, **NOT** an authorization mechanism (exam trap)
  - **mTLS** on custom domain names (truststore in S3)
- HTTP APIs: IAM, native **JWT authorizer** (any OIDC/OAuth2 provider incl. Cognito), Lambda authorizer (simple response)
- WebSocket APIs: IAM or Lambda authorizer on `$connect` only

![Gateway Authentication](images/GatewayAuth.png)

## Endpoint Types

- **Edge-Optimized**: any incoming request is routed to the nearest CloudFront POP (point of presence)
- **Region**: region endpoint for clients in the same region, it does not utilize the CloudFront endpoint
- **Private**: endpoints only accessible in a VPC via interface endpoints
- Edge-optimized uses an AWS-managed CloudFront distribution which we can't customize. If we need our own CloudFront (custom caching, WAF at edge, Lambda@Edge) => use **Regional** endpoint + our own CloudFront distribution
- Edge-optimized custom domain => ACM certificate must be in `us-east-1`; Regional => certificate in the API's region
- Private APIs:
  - Need an `execute-api` interface VPC endpoint + resource policy allowing that VPCE/VPC
  - Private DNS on the VPCE breaks access to public APIs from that VPC unless we use Route 53 alias / `x-apigw-api-id` header
  - Access from on-prem: via DX/VPN to the interface endpoint
  - **Custom domain names for private APIs** are supported (since Nov 2024), shareable cross-account via AWS RAM
- Dual-stack (IPv4 + IPv6) supported for REST/HTTP APIs and custom domains

## Stages

- When we deploy an API configuration, we are doing it into a stage
- Example we can have prod/dev stage with uniq settings and urls
- Deployments can be rolled back on a stage
- On stages we can enable canary deployments:
  - When enabled, the deployment will be made to the canary not the stage itself
  - Traffic distribution can be altered between canary and base stage
  - Canary stage can be promoted to base

![Gateway Stages](images/GatewayStages.png)

## Errors

- `4XX` - Client errors: invalid request on the client side
- `5XX` - Server errors: valid request, backend issue
- `400 - Bad Request`: generic client side error
- `403 - Access Denied`: authorizer denies request, request is WAF filtered
- `429` - **API Gateway can throttle: this means we have exceeded a specified amount of requests**
- `502 - Bad Gateway Exception`: bad output returned by Lambda
- `503 - Service Unavailable`: backing endpoint is offline
- `504` - Integration Failure/Timeout (29s default integration timeout)
- The 29s timeout:
  - REST API (Regional and Private): can be raised above 29s via Service Quotas, at the cost of a lower account-level throttle quota. Edge-optimized stays at 29s
  - HTTP API: max 30s, hard limit
  - Long running work => go async (return `202`, process via SQS/Step Functions, poll or push the result via WebSocket)

## Caching

- Caching is configured **per stage**
- We can define a cache on a stage (500 MB - 237 GB)
- Cache TTL default value is 300 seconds, configurable between 0 - 3600s. 0s = disabled.
- Can be encrypted
- Calls only will reach the backend in case of a cache miss
- Only for **REST APIs** (HTTP APIs have no cache => put CloudFront in front)
- Can be overridden per method; clients can bypass with `Cache-Control: max-age=0` (control via IAM `execute-api:InvalidateCache`)
- Billed per hour per cache size, not free tier

![Gateway Caching](images/GatewayCaching.png)

## Methods and Resources

- API Gateway URL example: `https://1nj7i16t37.execute-api.us-east-1.amazonaws.com/dev/listcats`
- The URL can be represents the following: `[api-gateway-endpoint]/[stage]/[resource]`
- Stages are logical configurations. APIs are deployed into stages. Stages can be used for different application versions or lifecycle points for an API
- API changes only take effect after it is deployed into a stage
- Methods are the desired action to be performed. Methods are HTTP verbs
- Methods are where integrations are configured which provide the functionality of an API. Methods can integrate with Lambda, HTTP and other AWS services

![Gateway Resources](images/GatewayResources.png)

## Integrations

- API Gateway is capable of connecting to Lambda, HTTP Endpoints (running on-premises or on AWS), Step Functions, SNS, DynamoDB
- APIs have 3 phases:
  - Request: authorize, validate and transform the request
  - Integrations
  - Response: transform, prepare and return the response
- The request and response phases are split into 2 parts:
  - Method Request: defines everything about the client request to method (path, headers, parameters)
  - Integration Request: parameters from the method request are transferred to the integrations
  - Integration Response: converts the data from the backend to a form which can be sent back to the client
  - Method Response: how the communication is delivered back to the client
- API methods which are on the client side decide what the client request to method is like. There are integrated to a backend endpoint via integrations
- Other integrations worth knowing: Kinesis Data Streams, SQS (`SendMessage` directly, no Lambda), EventBridge (HTTP API first-class integration), S3 (proxy for uploads/downloads up to 10 MB)
- **Private integrations** (VPC Link):
  - REST API: VPC Link v1 to an **NLB** (legacy) or **VPC Link v2** to ALB or NLB directly (no more NLB in front of a private ALB)
  - HTTP API: VPC Link to ALB, NLB or **Cloud Map** services
- Integration types:
  - **MOCK**: used for testing, no backed involved. It returns a static response
  - **HTTP**: http custom integration, backend is a HTTP endpoint. We have to configure both integration request and integration response
  - **HTTP Proxy**: subtype of the HTTP integration, but where proxying is utilized. Allows the access HTTP endpoint with a streamline integration. Proxying is where the request is passed to the endpoint as is and sent back to the client as is
  - **AWS**: allows an API to expose AWS services. We have to configure both the integration request and response and setup necessary mappings from the method request to the integration request. Can be used with Lambda functions, but it is relatively complex way of using it with Lambda
  - **AWS_PROXY (LAMBDA)**: integration request/response does not have to be defined, API Gateway passes the request unmodified
- Mapping template: used for non-proxy integrations. Used for:
  - Modify or rename parameters
  - Modify the body or header of the request
  - Filtering - remove anything from the request

![Gateway Integrations](images/GatewayIntegrations.png)

## Mapping Templates

- Used for AWS and HTTP (non proxy) integrations
- It is able modify and rename parameters between the integrations
- It can modify the body or the headers of a request
- It can provide filtering by removing anything which is not needed
- Mapping uses VTL (Velocity Template Langue) for editing the request
- Use cases for mapping templates:
  - Integrate a REST API on API Gateway with a SOAP API

## Stages and Deployments

- Editing an API, we are editing settings which are not live (not published)
- The current state of the API needs to be deployed to a stage
- Each stage has its own configuration. Configurations are not immutable, can be modified, overwritten or rolled back
- Stage variables: environment variables for stages
- Stage variables + Lambda aliases: `function:${stageVariables.alias}` => each stage invokes a different Lambda alias

![Gateway Variables](images/GatewayVariables.png)

## Swagger and OpenAPI

- OpenAPI (OAS) defines a standard language-agnostic interface to RESTful APIs
- OpenAPI v2 is formerly known as Swagger
- OpenAPI v3 is a more recent version
- OpenAPI defines endpoints, operation (GET, POST, etc.), input and output parameters and authentication methods
- API Gateway is capable of import OpenAPI format and generating it. Useful for backups and migrations

## REST vs HTTP vs WebSocket APIs

- **REST API** (v1): full feature set
  - Usage plans and API keys, per-client throttling
  - Caching, request validation, VTL mapping templates
  - AWS WAF integration, resource policies, Private endpoint type
  - Edge-optimized endpoint, canary deployments, X-Ray tracing
  - Response streaming (since Nov 2025)
- **HTTP API** (v2): cheaper (up to ~70% less) and lower latency
  - Lambda proxy, HTTP proxy, private integrations (ALB/NLB/Cloud Map), some AWS service integrations (SQS, EventBridge, Kinesis, Step Functions, AppConfig)
  - Native JWT/OIDC authorizer, automatic deployments, CORS built-in
  - No WAF, no caching, no usage plans/API keys, no Private endpoint type, Regional only
- **WebSocket API**: persistent two-way connections (chat, dashboards, games)
  - Routes: `$connect`, `$disconnect`, `$default` and custom routes selected by a JSON field (route selection expression)
  - Backend pushes messages to clients via the `@connections` API (`POST https://{api-id}.execute-api.{region}.amazonaws.com/{stage}/@connections/{connectionId}`)
  - Idle timeout 10 min, max connection duration 2h
- Exam pattern: need WAF / API keys / caching / private => **REST**; cheapest simple Lambda/JWT proxy => **HTTP**; server push => **WebSocket** (or AppSync)

## Throttling and Quotas

- Account-level default: **10,000 RPS** steady-state with **5,000 burst** per region (soft limit, token bucket)
- Throttling levels: account => stage/method => usage plan per client (API key). Exceeding returns `429 Too Many Requests`
- Max payload: **10 MB** (request and response) - use S3 pre-signed URLs for bigger files
- Lambda proxy: Lambda sync payload limit is 6 MB, so 6 MB is the real limit for Lambda backends
- Regional APIs per account: 600 (default), edge-optimized: 120
- Mapping templates / VTL: REST only

## Response Streaming

- REST APIs can stream the integration response (`responseTransferMode = STREAM`) for `HTTP_PROXY` and `AWS_PROXY` (Lambda) integrations
- Streams up to 15 min; idle timeout 5 min (Regional/Private), 30s (edge-optimized)
- First 10 MB unrestricted, after that bandwidth capped at 2 MB/s
- Not compatible with caching, content encoding or VTL response transformations
- Use case: LLM/GenAI token streaming, large responses, avoiding the 29s limit with long-lived responses

## Custom Domains

- Custom domain name + base path mappings (or API mappings for HTTP APIs) to route `api.example.com/v1` and `/v2` to different APIs/stages
- **Routing rules** (2025): route on headers and base paths to different APIs/stages under one custom domain (e.g. header-based API versioning, A/B)
- Certificates from ACM (or imported), TLS security policies selectable

## Security Extras

- AWS WAF (REST API only): rate-based rules, SQLi/XSS, geo-blocking, Bot Control
- AWS Shield Standard is automatic; edge-optimized benefits from CloudFront edge
- Client certificates (API Gateway => backend) to prove the call comes from API Gateway
- CloudTrail for management events, access logs (CloudWatch Logs or Firehose) and execution logs per stage
- Cross-account: Lambda in another account => Lambda resource policy must allow `apigateway.amazonaws.com` from our API ARN

## Exam Tips

- "Throttle per customer / monetize API" => usage plans + API keys (REST)
- "Protect API from SQL injection / rate based blocking" => WAF on REST API (not possible on HTTP API) or CloudFront + WAF in front
- "API accessible only from VPC / on-prem" => Private REST API + interface endpoint + resource policy
- "Request takes > 29s" => async pattern (SQS/Step Functions) or raise the REST integration timeout, or response streaming
- "Reduce Lambda invocations for repeated GETs" => stage cache (REST) or CloudFront
- "Expose SQS/DynamoDB/Step Functions without Lambda" => AWS service integration with mapping templates (fewer moving parts, cheaper)
- Canary on stage vs Lambda alias weighted routing: both are valid, alias routing via CodeDeploy is the typical serverless blue/green
