# Amazon Comprehend

> **Status (2026):** Comprehend **topic modeling**, **event detection** and **prompt safety classification** are no longer available to new customers (since Apr 30, 2026; accounts that used them in the last 12 months keep access). AWS points to Amazon Bedrock LLMs for topics/events and **Bedrock Guardrails** for prompt safety. All other Comprehend features (entities, sentiment, PII, classification, etc.) are unaffected.

- It is a natural language processing (NLP) system available in AWS
- We input a document and Comprehend will develop insights by recognizing the entities, key phrases, language, sentiments and other common elements of the document
- How the system works:
    - Input = Document
    - Output = Entities, phrases, language, PII, sentiments
- Comprehend is ML services and it is based on pre-trained and custom models
- Product is capable of doing real-time analysis for small workloads and asynchronous analysis for large workloads in form of jobs
- The text analysis type can be:
    - Built-in: using AWS built-in models
    - Custom: using custom models from endpoints we have created
- Built-in APIs: `DetectDominantLanguage`, `DetectEntities`, `DetectKeyPhrases`, `DetectSentiment`, `DetectTargetedSentiment` (sentiment per entity), `DetectSyntax`, `DetectPiiEntities`/`ContainsPiiEntities`, `DetectToxicContent`
- **PII**: detect and **redact** PII (names, addresses, SSN, credit cards) - real-time APIs or async batch jobs on S3 (redacted copies written to S3)
- **Custom classification** (classify documents into our own labels, e.g. support ticket routing) and **custom entity recognition** (our own entity types, e.g. policy numbers) - trained with our labeled data, no ML expertise required; served via real-time endpoints (provisioned inference units) or async jobs
- Supports plain text, and PDF/Word/images for custom models (uses Textract internally for document parsing)
- **Flywheels**: manage continuous retraining/versioning of custom models
- **Comprehend Medical** (separate HIPAA-eligible service): extract medical entities (medications, conditions, dosage), PHI detection, ontology linking (ICD-10-CM, RxNorm, SNOMED CT)
- Integrations: S3 (input/output), KMS encryption, VPC for jobs, Kinesis/Lambda for streaming pipelines, Athena (UDFs), QuickSight for dashboards
- Typical pipeline: Transcribe (audio -> text) -> Translate (to English) -> Comprehend (sentiment/entities) -> OpenSearch/QuickSight
- Exam: "detect sentiment/entities/PII in customer reviews, emails, tickets" -> Comprehend; "summarize or free-form reasoning on text" -> Bedrock
