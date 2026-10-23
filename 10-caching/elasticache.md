# ElastiCache

- It is an in-memory database for application which need high-end performance
- It is orders of magnitude faster than a classic DB, but it is not a durable primary store (Valkey/Redis OSS offer snapshots/replication, but writes can still be lost on failover)
- ElastiCache provides 3 engines: **Valkey**, **Redis OSS** and **Memcached** (managed, wire-compatible)
- Deployment options: **Serverless** or **node-based (self-designed) clusters**
- ElastiCache can be used for read heavy workloads with low latency requirements
- Can be used for reduction of database workloads, by this reducing cost accumulated by heavy database usage
- Can be used to store session data, making stateful applications stateless
- Using ElastiCache requires application code changes!
- Runs inside a VPC (no public endpoint), access controlled by security groups

![Elasticache](images/Elasticache.png)

![Elasticache Session](images/ElasticacheSession.png)

## Engines (2026)

- **Valkey**: open source (Linux Foundation) fork of Redis OSS 7.2, created after the Redis license change (2024). AWS-recommended default
  - ~20% cheaper node pricing and ~33% cheaper Serverless pricing than Redis OSS; lower Serverless minimum (100 MB vs 1 GB)
  - In-place upgrade from Redis OSS to Valkey with no downtime, no code changes (same API/protocol)
- **Redis OSS**: up to 7.1 on ElastiCache ("ElastiCache for Redis" was renamed to Redis OSS)
- **Memcached**: simple, multi-threaded key/value cache

## Redis vs Memcached

- Both offer sub-ms access to data
- Memcached supports simple data structures (string), while Redis can support more advanced type of data: lists, sets, sorted sets, hashes, bit arrays, streams, geospatial, HyperLogLog, etc.
- Redis supports replication of data across multiple AZs, Memcached supports multiple nodes with manual sharding, but it does not supports "true" replication across AZs for scalability reasons
  - Exception: **ElastiCache Serverless for Memcached** replicates data across multiple AZs automatically
- Redis supports backups and restores, Memcached does not support persistence
- Memcached is multi-threaded by design, can take better advantage of multithreaded CPUs, can offer better performance
- Redis supports transactions (multiple operations at once), pub/sub, Lua scripting, sorted sets (leaderboards)
- Both of these engines can support a ranges of instance types
- Rule of thumb (exam): need HA, persistence/backups, complex types, pub/sub, leaderboards, Global Datastore => Valkey/Redis OSS; simplest cache, multi-threaded, horizontally scaled, data loss acceptable => Memcached

![RedisMemcached](images/RedisMemcached.png)

## ElastiCache Serverless

- Launched Nov 2023, available for Valkey, Redis OSS and Memcached
- No capacity planning: scales compute and memory automatically, deployed across multiple AZs, 99.99% availability SLA
- Billed per GB-hour of data stored + **ECPUs** (ElastiCache Processing Units: ~1 ECPU per KB transferred for simple reads/writes)
- Single endpoint, TLS always on
- Limitations vs node-based: no Global Datastore, no data tiering, less control over engine parameters/instance types
- Good for spiky/unpredictable workloads, new apps, dev/test; node-based (with Reserved Nodes / Database Savings Plans) usually cheaper for steady high utilization

## Node-Based Clusters (Valkey / Redis OSS)

- **Cluster mode disabled**: 1 shard, 1 primary + up to 5 read replicas; scale reads with replicas, scale writes vertically only
- **Cluster mode enabled**: data partitioned across up to 500 shards, each with 1 primary + up to 5 replicas => scale writes horizontally; online resharding
- **Multi-AZ with automatic failover**: replica promoted when primary fails (DNS endpoint stays the same)
- Endpoints: primary endpoint (writes), reader endpoint (load-balanced reads), configuration endpoint (cluster mode enabled)
- Backups: automatic daily snapshots (retention up to 35 days) + manual; export to S3; restore into a new cluster (also used to seed migrations)
- **Data tiering** (r6gd nodes): least-used data moved to local NVMe SSD => up to ~5x capacity at lower cost, small latency hit for SSD-read items
- **Online migration** from self-managed Redis on EC2/on-prem into ElastiCache (replication-based)

## Global Datastore (Valkey / Redis OSS)

- Cross-region replication: 1 primary (read/write) cluster + up to 2 secondary (read-only) clusters in other regions
- Replication latency typically < 1 second
- Use cases: low-latency regional reads, cross-region DR; promote a secondary to primary in case of regional failure (manual)
- Node-based only (not Serverless, not Memcached)

## Security

- Encryption in transit (TLS) and at rest (KMS); Serverless always encrypted
- Authentication:
  - Valkey/Redis OSS `AUTH` token (password)
  - **RBAC** with users and user groups (ACL commands per user)
  - **IAM authentication** for Valkey/Redis OSS 7+ (short-lived IAM tokens)
  - Memcached: SASL (node-based, 1.6+)
- Network: security groups, subnet groups, no public access

## Caching Strategies

- **Lazy loading (cache-aside)**: read from cache, on miss read DB and populate cache. Only requested data cached; stale data possible; cache miss penalty (3 trips)
- **Write-through**: write to DB and cache at the same time. Data never stale; write penalty; cache churn with unused data
- **TTL**: combine with both strategies to expire stale data and limit memory
- Eviction policies (e.g. `allkeys-lru`, `volatile-lru`) when memory is full
- Session store pattern: Valkey/Redis with TTL => stateless app tier behind an ALB (alternative to sticky sessions)

## ElastiCache vs Alternatives

- **MemoryDB** (Valkey/Redis OSS compatible): durable in-memory **primary database**, Multi-AZ transaction log, microsecond reads/single-digit ms writes, multi-region (active-active) available. Use when you need durability + in-memory speed (no separate DB)
- **DAX**: in-memory cache only for **DynamoDB**, API-compatible (no app logic changes beyond the client), microsecond reads
- **CloudFront**: caching at the edge for HTTP content, not a data cache
- **RDS read replicas**: offload reads with SQL, no code change for cache logic, but ms latency
- API Gateway caching: response caching per stage

## Exam Tips

- "Reduce DB read load, sub-ms latency, code changes acceptable" => ElastiCache
- "No code changes" requirement => cache is not the answer (read replicas, DAX for DynamoDB is minimal change)
- "Durable Redis-compatible primary database" => MemoryDB, not ElastiCache
- "Cross-region low-latency reads for Redis" => Global Datastore
- "Leaderboard / sorted rankings" => Valkey/Redis sorted sets
- "Cheapest Redis-compatible option" => Valkey (on Serverless or Reserved Nodes)
- "Unpredictable traffic, no capacity management" => ElastiCache Serverless
