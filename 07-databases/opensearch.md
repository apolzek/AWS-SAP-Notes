# AWS OpenSearch (Formerly known as ElasticSearch)

- Is a managed implementation of ElasticSearch, an open source search solution. It is part of the ELK stack (ElasticSearch, Logstash, Kibana)
- OpenSearch is an open source fork of Elasticsearch 7.10 / Kibana 7.10 (2021). Amazon OpenSearch Service supports OpenSearch versions and legacy Elasticsearch up to 7.10
- OpenSearch is not serverless, it runs in a VPC using compute (managed **domains**). AWS also offers **OpenSearch Serverless** (see below)
- OpenSearch is usually an alternative to other AWS services
- Can be used for log analytics, monitoring, security analytics, full text search and indexing, click stream analytics
- Also a **vector database** (k-NN, vector engine) for semantic search and RAG (default vector store for Bedrock Knowledge Bases via OpenSearch Serverless)

![OpenSearch](images/OpenSearch.png)

## ELK Stack

- ElasticSearch (or OpenSearch on AWS): provides search and indexing services
- Kibana: visualization and dashboard tool (on AWS => **OpenSearch Dashboards**)
- Logstash: similar to CloudWatch Logs, needs a Logstash agent installed on anything to ingest data (on AWS => **OpenSearch Ingestion**, based on Data Prepper)

## OpenSearch Service Domains (managed clusters)

- Domain = cluster of data nodes + optional **dedicated cluster manager (master) nodes** (use 3 for production, odd number)
- Deploy across **2 or 3 AZs**. **Multi-AZ with Standby**: 3 AZs, one AZ is standby, **99.99% SLA**, faster failover
- Data is split into indices => shards (primary + replica shards); replicas in other AZs for HA
- Storage tiers:
  - **Hot**: EBS/instance storage on data nodes, fastest, most expensive
  - **UltraWarm**: S3-backed with caching, **read-only**, up to ~90% cheaper than hot => older logs still queried
  - **Cold**: detached S3 storage, must be attached to UltraWarm to query, cheapest => compliance/rare access
  - **Index State Management (ISM)** policies move indices hot => UltraWarm => cold => delete
- **OpenSearch Optimized instances** (OR1, newer **OR2/OM2** 2025, OI2): segments replicated to S3 (remote-backed storage) => high indexing throughput, durability, fewer replicas needed; good for indexing-heavy log workloads
- Snapshots: automated hourly (retained 14 days, free) + manual snapshots to our S3 bucket (for restore, migrations, cross-region)
- **Cross-cluster replication** (active-passive, for DR/geo-proximity) and **cross-cluster search**
- Blue/green deployment is used for most config changes (no downtime, but extra temporary nodes)

## OpenSearch Serverless

- No domains/nodes to manage, compute (**OCUs**) decoupled from storage (S3)
- **Collections** by type: time series (logs), search, vector search
- Billed per OCU-hour for indexing + search, and per GB stored. Minimum 1 OCU indexing + 1 OCU search with redundancy (0.5 + 0.5 for dev/test without redundancy)
- Security via encryption, network (VPC endpoint or public) and data access policies
- Use for spiky/unpredictable workloads, vector stores for RAG, no capacity planning
- Fewer features than managed domains (no UltraWarm/cold, limited plugins/ISM)

## Ingestion and Integrations

- **Amazon OpenSearch Ingestion (OSI)**: serverless pipelines (Data Prepper) to filter, enrich, transform and route data; sources: HTTP, S3, Kinesis, MSK, DynamoDB, DocumentDB, OpenTelemetry
- **Zero-ETL**: DynamoDB => OpenSearch (via OSI), DocumentDB => OpenSearch
- **Amazon Data Firehose** => OpenSearch (buffered, near real-time), backup failed records to S3
- **CloudWatch Logs** subscription filter => OpenSearch (via Lambda)
- **Direct query** (zero-ETL query without ingestion) for S3, CloudWatch Logs and Security Lake data
- Common pattern: DynamoDB (source of truth) + OpenSearch (search index) via Streams/zero-ETL

## Security

- Domain in a **VPC** (recommended, can't change later) or public endpoint
- **Resource-based (domain) access policies**, identity-based policies, IP-based policies
- **Fine-grained access control**: index, document and field-level security, internal user DB or IAM
- Dashboards authentication: **Amazon Cognito**, SAML, IAM Identity Center
- Encryption at rest (KMS), node-to-node encryption, HTTPS enforced

## When to use OpenSearch (exam)

- Full-text search, autocomplete, relevance scoring on data stored elsewhere (DynamoDB, RDS) => OpenSearch
- Near real-time log analytics and dashboards (Kinesis/Firehose => OpenSearch) => OpenSearch
- Ad-hoc SQL on S3 logs, cheap and serverless => Athena instead
- Vector search for RAG => OpenSearch Serverless (or Aurora pgvector, S3 Vectors for cheaper storage)
