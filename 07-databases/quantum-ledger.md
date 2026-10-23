# Amazon Quantum Ledger Database - QLDB

> **Status (2026):** QLDB reached **end of support on July 31, 2025** (closed to new customers since July 2024). It can no longer be used. AWS recommends migrating to **Amazon Aurora PostgreSQL** (journal/history replaced by audit tables + `pgAudit`, **Database Activity Streams**, export to S3). Still possible in exam questions written before the deprecation; for new designs choose the alternatives below

- Part of AWS Blockchain part of products
- It as an immutable append-only ledger-only database
- It provides a cryptographically verifiable transaction log
- It is transparent: full history is always accessible
- It is a serverless product, it provides Ledgers and Tables. We have no servers to manage
- It is resilient through 3 AZs, replicates data within each of those AZs
- It can stream data to Amazon Kinesis, it can stream any changes to data into Kinesis in real-time
- It is a document database model, storing JSON documents (key-value pairs with a nested structure)
- Provides ACID transactions
- Query language: PartiQL. Verification via SHA-256 hash chaining (digest)
- Use cases for QLDB:
    - Anything related to finance: account balances and transactions
    - Medical application: full history of data changes matters
    - Logistics: track movement of objects
    - Legal: track the usage and change of data (custody)

## Alternatives (2026)

- **Aurora PostgreSQL** with history/audit tables, triggers, `pgAudit`, Database Activity Streams => AWS-recommended QLDB replacement
- **Amazon Managed Blockchain** (Hyperledger Fabric / Ethereum) => decentralized, multiple untrusted parties (QLDB was centralized, single trusted owner)
- **S3 Object Lock** (WORM) + CloudTrail Lake / AWS Backup Vault Lock => immutable audit storage for compliance
- DynamoDB Streams / Kinesis to S3 with Object Lock => change history pipeline
