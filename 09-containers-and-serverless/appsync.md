# AWS AppSync

- AppSync is a managed service which uses *GraphQL*
- It is used for building APIs on AWS which use *GraphQL*
- *GraphQL* makes it easy for applications to get the exact data they need. This includes combining data from multiple resources
- Datasets behind *GraphQL* can include:
    - NoSQL data stores
    - RDS databases
    - HTTP APIs
    - etc.
- AppSync integrates with (resolvers) DynamoDB, Aurora (via RDS Data API), OpenSearch (formerly Elasticsearch), HTTP endpoints, EventBridge, Amazon Bedrock, etc.
- Supports custom resources/data sources using Lambda
- Provides support for real time data retrieval using **subscriptions over WebSockets** (the old MQTT over WebSocket protocol is legacy)
- Mobile applications: replacement for Cognito Sync
- Requires a GraphQL schema for getting started
- Example for GraphQL schema:

    ```
    type Query {
        human(id: ID!): Human
    }

    type Human {
        name: String
        appearsIn: [Episode]
        starships: [Starship]
    }

    enum Episode {
        NEWHOPE
        EMPIRE
        JEDI
    }

    type Starship {
        name: String
    }
    ```

## Security

- Five ways we can authorize applications to interact with AppSync (multiple can be enabled on one API, default + additional modes):
    - API_KEY (dev/public data, keys expire, max 365 days)
    - AWS_IAM (backend services, Cognito identity pools for guest/federated users)
    - OPENID_CONNECT (OpenID Connect provider/ JWT)
    - AMAZON_COGNITO_USER_POOLS (fine grained with `@aws_auth` / group directives)
    - AWS_LAMBDA (custom authorization logic)
- AppSync supports **custom domain names natively** (ACM certificate in `us-east-1`); CloudFront in front is only needed for extra edge features
- AWS WAF can be attached to AppSync APIs
- **Private APIs**: accessible only from a VPC via an interface endpoint (`appsync-api`)

## Resolvers

- Resolver = connects a GraphQL field to a data source
- Unit resolvers (one data source) or **pipeline resolvers** (sequence of functions, e.g. authz check then write)
- Runtimes: **APPSYNC_JS** (JavaScript, recommended) or VTL (legacy)
- Direct Lambda resolvers: no mapping template, Lambda gets the full context
- Server-side **caching** (ElastiCache-backed, per-resolver or full request) to reduce data source calls
- Conflict detection/resolution and offline sync (used by Amplify DataStore)

## Merged APIs

- Combine multiple source GraphQL APIs (owned by different teams/accounts, shared via RAM) into one endpoint
- Solves the "single GraphQL schema owned by many teams" problem (federation-like)

## AppSync Events

- Serverless **WebSocket pub/sub API** (Oct 2024), no GraphQL needed
- Concepts: Event API => **channel namespaces** => channels (e.g. `/chat/room1`, wildcard subscriptions `/chat/*`)
- Publish via HTTP or directly over the WebSocket connection; subscribe via WebSocket
- Namespace handlers (JS) to filter/transform events; data source integrations (Lambda, DynamoDB, Aurora, etc.) on publish/subscribe
- Same auth modes (API key, IAM, Cognito, OIDC, Lambda)
- Use case: broadcasting real-time updates to millions of clients (live scores, notifications, chat) without managing connections

## AppSync vs API Gateway (Exam)

- Need GraphQL, aggregating many data sources in one request, mobile/web real-time + offline => **AppSync**
- Need REST, usage plans/API keys for monetization, VTL integrations with any AWS service => **API Gateway REST**
- Simple serverless WebSocket broadcast fan-out => **AppSync Events** (managed connections & fan-out) vs API Gateway WebSocket (we track `connectionId`s ourselves and push via `@connections`)
