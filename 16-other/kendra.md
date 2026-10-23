# Amazon Kendra

> **Status (2026):** Amazon Kendra entered **maintenance mode on June 30, 2026** (no new features) and is **no longer open to new customers since July 30, 2026**. Existing customers remain supported (bug fixes, security). AWS recommends **Amazon Bedrock Knowledge Bases** (managed knowledge base - fully managed RAG with built-in connectors, hybrid search, `Retrieve`/`RetrieveAndGenerate` APIs) for new search/RAG workloads.

- Is an intelligent search service
- It's primary aim is to be designed to mimic interacting with a human expert
- Supports wide range if question types such as:
    - Factoid: Who, What, Where
    - Descriptive: How do I get my cat to spot being a jek?
    - Keyword: What time is the keynote address (address can have multiple meaning) - Kendra helps determine intent

## Key Concepts

- **Index**: searchable data organized in an efficient way
- **Data Source**: where the data lives, Kendra connects and indexes from this location. Example of locations are: S3, Confluence, Google Workspaces, RDS, OneDrive, Salesforce, Kendra Web Crawler, Workdocs (Amazon WorkDocs itself was discontinued in Apr 2025), FSX, SharePoint, ServiceNow, Jira, Slack, etc. (30+ native connectors)
- We configure Kendra to synchronize a data source with an index based on a **schedule**. This should keep the index current
- **Documents**: can be structured (FAQs) and unstructured (HTML, PDF, etc.)

## Others

- Kendra integrates with other AWS services: IAM, Identity Center (SSO) and others
- Access control: document-level ACLs / user context filtering (users and groups from Identity Center or token-based) - search results only show documents the user is allowed to see
- Editions/index types: **Developer** (testing), **Enterprise** (production, HA), **GenAI Enterprise Edition index** (Dec 2024 - hybrid vector + keyword index optimized for RAG)
- **GenAI Index** can be used as a retriever for **Amazon Bedrock Knowledge Bases** and **Amazon Q Business** (one index reused by multiple apps)
- `Query` API (search results, FAQ answers, suggested answers) vs `Retrieve` API (up to 100 semantically relevant passages - used as RAG context for an LLM)
- Features: incremental learning from user feedback (click-through), query suggestions, synonyms (thesaurus), relevance tuning/boosting, custom document enrichment (Lambda hooks to transform content/metadata at ingestion), facets
- Exam (current): "enterprise search / Q&A over internal documents with generative answers" -> Bedrock Knowledge Bases (RAG) or Amazon Q Business; Kendra only for existing deployments or when a legacy question explicitly expects it
