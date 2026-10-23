# Amazon Neptune

- Neptune is a managed graph database in AWS
- A graph database is a database type where the relationships between the data is as important as the data itself
- Neptune runs in a VPC, private by default
- It is resilient, it can be deployed in multiple AZs and scales via read replicas
- It does continuous backups and allows point-in-time recovery
- Common use cases for graph based data models:
    - Social media used databases - anything involving fluid relationships
    - Fraud prevention
    - Recommendation engines
    - Network and IT Operations
    - Biology and other life sciences
    - Knowledge graphs, identity graphs, GraphRAG (graph + LLM retrieval with Bedrock Knowledge Bases)

## Neptune Database

- Query languages: **Gremlin** (Apache TinkerPop, property graph), **openCypher** (property graph), **SPARQL** (RDF)
- Architecture similar to Aurora: cluster with 1 writer + up to **15 read replicas**, shared storage volume, 6 copies across 3 AZs, up to 128 TiB
- Endpoints: cluster (writer), reader, instance, custom
- Failover to a replica typically < 30 s
- **Neptune Serverless**: capacity in NCUs (Neptune Capacity Units), scales automatically, for variable/unpredictable workloads; can mix with provisioned instances
- **Neptune Global Database**: primary region + up to **5 secondary read-only regions**, storage-level replication, typically < 1 s lag, for DR and low-latency global reads
- **Neptune Streams**: change log of graph mutations (like DynamoDB Streams), used for replication to OpenSearch (full-text search), Lambda, etc.
- Bulk loader from S3 (CSV, RDF formats) => requires an S3 VPC endpoint
- Security: IAM database authentication (SigV4), KMS encryption at rest, TLS in transit
- **Neptune ML**: graph neural networks via SageMaker for predictions (link prediction, node classification)
- Pricing options: on-demand instances, serverless, Database Savings Plans

## Neptune Analytics

- Separate **in-memory** graph analytics engine (memory-optimized "graphs" sized in m-NCUs)
- Loads data from a Neptune Database cluster or from S3 and analyzes **tens of billions of relationships in seconds**
- Built-in graph algorithms (PageRank, shortest path, community detection, etc.) and **vector search** (store embeddings on nodes)
- Query language: openCypher
- Neptune Database = OLTP graph (transactional, high QPS). Neptune Analytics = OLAP graph (ad-hoc analysis, algorithms, GraphRAG)

## When to use Neptune (exam)

- Relationships are the core of the data and queries traverse many hops (friends of friends, fraud rings) => Neptune
- Many joins in RDS for highly connected data are slow => Neptune
- Not for simple key-value (DynamoDB) or full-text search (OpenSearch)
