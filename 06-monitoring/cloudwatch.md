# CloudWatch

- Provides services to ingest, store and manage metrics
- It is a public service - provides public space endpoints
- Many services have native management plan integration with CloudWatch, for example EC2. Also, EC2 provides external gathered information only, for metrics from inside an EC2 we can use CloudWatch agent
- CloudWatch can be used from on-premises using the agent or the CloudWatch API
- CloudWatch stores data in a persistent way
- Data can be viewed from the console, CLI or API, but also CloudWatch also provides dashboards and anomaly detection
- CloudWatch Alarms: react to metrics, can be used to notify or to perform actions
- Instances in public subnet can connect to cloudwatch using Internet gateway and the instances in the private subnets can connect to the cloudwatch usig Interface Endpoint (`monitoring` for metrics, `logs` for CloudWatch Logs).
- EC2 default metrics: CPU, network, disk (instance store), status checks - **NOT memory, disk space or processes** => need the CloudWatch agent (exam classic)
  - Basic monitoring: 5 min (free); detailed monitoring: 1 min (paid)
- Custom metrics: `PutMetricData` (standard or high resolution), or **Embedded Metric Format (EMF)** - structured JSON logs automatically extracted into metrics (great for Lambda)
- CloudWatch agent also collects logs, `procstat`, StatsD/collectd, Prometheus metrics, OpenTelemetry/X-Ray traces; config stored in SSM Parameter Store for fleet deployment

![CloudWatch](images/CloudWatch.png)

## CloudWatch - Data

- **Namespace**: container for metrics e.g. AWS/EC2 for EC2 NS, AW/Lambda for lambda NS. Its possible to have same mertic name for different service, NS helps to segregate them.
- **Data point**: timestamp, value, unit of measure (optional)
- **Metric**: time ordered set of data point. Example of builtin metrics: `CPUUtilization`, `NetworkIn`, `DiskWriteBytes` for EC2
- Every metric has a `MetricName` and a namespace e.g. CPUUtilization for AWS/EC2
- **Dimension**: name/value pair, example: a dimension is the way for `CPUUtilization` metric to be separated from one instance to another. Ex: `CPUUtilization (Name: InstandeId, Value: i-190038210)`
- Dimensions can be used to aggregate data, example aggregate data for all instances for an ASG
- **Resolution**: standard (60 second granularity) or high (1 second granularity)
- **Metric resolution**: minimum period that we can get one particular data point for
- Data retention:
  - sub 60s granularity is retained for 3 hours
  - High resolution can be measured but they cost more. Resolution determines the minimum period which ca be specified and get a valid value. Standard (60 Sec) .. High (1 Sec)
  - 60s granularity retained for 15 days
  - 5 min retained for 63 days
  - 1 hour retained for 455 days
- As data ages, its aggregated and stored for longer period of time with less resolution
- Statistics: get data over a period and aggregate it in a certain way
- Percentile: relative standing of a value within the dataset (e.g. p99 latency)
- Metric timestamps accepted: up to 2 weeks in the past and 2 hours in the future
- **Metric Math**: expressions across metrics (e.g. error rate = errors / requests); **Metrics Insights**: SQL-like queries over metrics (`SELECT AVG(CPUUtilization) FROM SCHEMA("AWS/EC2", InstanceId) GROUP BY ...`)
- **Anomaly detection**: ML model of expected band for a metric, alarms on band breach
- **OpenTelemetry metrics (2026)**: native OTLP ingestion of metrics (high cardinality, up to 150 labels) and **PromQL** queries in CloudWatch Query Studio
- **Metric Streams**: continuous near real-time stream of metrics to Kinesis Data Firehose => S3 or third parties (Datadog, Splunk, New Relic...) in JSON or OpenTelemetry format - use instead of polling `GetMetricData`

## CloudWatch Alarms

- Alarm: watches a metric over a period of time
- States: `OK`, `ALARM` or `INSUFFICIENT_DATA` based on the value of a metric against a threshold over time
- Evaluation: period + "datapoints to alarm" M out of N (e.g. 3 out of 5); treat missing data as `missing`, `notBreaching`, `breaching` or `ignore`
- Alarms can be configured with one or more actions, which can initiate actions on our behalf. Actions can be: send notification to an SNS topic, attempt an auto scaling policy modification or use Event Bridge to integrate with other services
  - **EC2 actions**: stop, terminate, reboot, **recover** the instance
  - Lambda function invocation directly, Systems Manager OpsCenter OpsItems / Incident Manager incidents
  - Alarm state changes are always emitted as EventBridge events
- High resolution metrics can have high resolution alarms (10s or 30s periods)
- **Composite alarms**: combine alarms with AND/OR/NOT logic => reduce alarm noise; can suppress actions (e.g. during deployments)
- **Alarm mute rules** (2026): mute notifications during planned maintenance windows
- Alarms can be based on metric math, anomaly detection bands, Metrics Insights queries (fleet-wide alarms) and PromQL
- Alarms are regional; cross-account alarms are possible from a monitoring account (cross-account observability)

![Alarms](images/Alarms.png)

## CloudWatch Logs

- CloudWatch Logs provides two type of functionalities: ingestion and subscription
- CloudWatch Logs is a public service designed to store, monitor and provide access logging data
- Can provide logging ingestion for AWS products natively, but also for on-premises, IOT or any application
- CloudWatch Agent: used to provide ingestion for custom applications
- CloudWatch can also ingest log streams from VPC Flow Logs or CloudTrail (account events and AWS API calls)
- CloudWatch Logs is regional service, certain global services send their logs to `us-east-1`
- Log events consist of 2 parts:
  - **Timestamp**
  - **Raw message**
- Log Events can be collected into Log Streams. Log Streams are sequence of log events sharing the same source
- Log Groups: are collection of Log Streams. We can set retention, permissions and encryption on the log groups. By default log groups store data indefinitely
- Metric Filter: can be defined on the log group and will look for pattern in the log events. Essentially creates a metric from the log streams by looking at occurrences of certain patterns defined by us (example: failed SSH logs in events)
  - Metric filters are **not retroactive** - only new events after creation generate data points
- Retention: 1 day to 10 years, or never expire (default); encryption with KMS at the log group level
- **Log classes**:
  - **Standard**: all features
  - **Infrequent Access** (2023): ~50% lower **ingestion** price (storage and Logs Insights prices are the same), supports Logs Insights, export to S3, KMS, cross-account, data masking - but **NO** subscription filters, metric filters, Live Tail, anomaly detection, EMF, `GetLogEvents`/`FilterLogEvents` - good for ad-hoc/forensic logs
  - **Delivery** class: only for delivering Lambda logs to S3/Firehose (2-day fixed retention, no Insights)
  - Log class is chosen at log group creation and **can't be changed**
- **Logs Insights**: interactive query language (also OpenSearch PPL and SQL), multiple log groups (or by log group tags), saved queries, JOINs/sub-queries, natural-language query generation; pay per GB scanned; **field indexes** reduce scanned data
- **Live Tail**: real-time streaming view of incoming logs (troubleshooting)
- **Log anomaly detection** and pattern analysis (ML-based)
- **Data protection policies**: detect and **mask sensitive data** (PII, credentials) in log events; only users with `logs:Unmask` see the raw data
- **Contributor Insights**: top-N contributors from logs (e.g. top IPs, top error-causing users)
- Ingestion options: CloudWatch agent, SDK `PutLogEvents`, vended logs from AWS services, HTTP-based ingestion endpoints (HTTP Log Collector, OpenTelemetry/OTLP) (2026)

![Logs](images/Logs.png)

Export logs from CloudWatch:

- **S3 Export**: we can create an export task (`CreateExportTask`) which will take up to 12 hours. Its not real time. Bucket encryption SSE-S3 or SSE-KMS; one active export task per account at a time
- **Subscription**: deliver logs in near real time or real time. We should create a subscription filter for the following destination: Kinesis Data Firehose (near real time), OpenSearch (ElasticSearch) using Lambda or custom Lambda, Kinesis Data Streams (any KCL consumer)
  - Max **2 subscription filters per log group**; **account-level subscription filter** applies to all log groups of the account
  - Cross-account: subscription filter in source account => **destination** (`PutDestination` + destination policy) in the central account, backed by Kinesis Data Streams / Firehose
- Subscription filters can be used to create a logging aggregation architecture to aggregate logs from different accounts
- **Logs centralization** (2025+): organization-level rules to copy log data from many accounts **and regions** into a central account (no custom subscription pipelines)
- Direct S3 delivery: many vended logs (VPC Flow Logs, ELB access logs, CloudFront, Route 53 Resolver) can go straight to S3/Firehose without CloudWatch Logs (cheaper)

![Subscription](images/Subscription.png)

## CloudWatch Dashboards

- A great way to setup dashboards for quick access to key metrics
- Dashboards are global
- Dashboards can include graphs from different regions
- We can change the time zone and time range of the dashboards
- We can setup automatic refresh (10s, 1m, 2m, 5m, 15m)
- Pricing:
  - 3 dashboards (up to 50 metrics) for free
  - $3/dashboard/month
- Dashboards can be shared publicly, with specific email addresses (Cognito) or via SSO; cross-account dashboards from a monitoring account
- Automatic dashboards per service; Logs Insights results, alarms and custom widgets (Lambda) can be added

## Cross-Account Observability

- **CloudWatch cross-account observability (OAM)**: a **monitoring account** (sink) links to **source accounts** (links) - share metrics, logs, traces, Application Signals, Internet Monitor data
- Set up per region; works with AWS Organizations (link all accounts of an OU via CloudFormation StackSets / policy) or individual accounts
- A monitoring account can have up to 100,000 source accounts; a source account can share with up to 5 monitoring accounts
- No data copy (data stays in source account) - no extra charge for metrics/logs sharing, cross-account alarms supported
- Older alternative: cross-account cross-region dashboards via IAM roles (`CloudWatch-CrossAccountSharingRole`)
- **Telemetry config / enablement rules**: audit and auto-enable telemetry (VPC Flow Logs, EC2 detailed monitoring, etc.) across the org and regions

## CloudWatch Observability Features

- **Container Insights**: metrics/logs for ECS, EKS, Kubernetes on EC2, Fargate (enhanced observability for EKS/ECS, OpenTelemetry-based for EKS in 2026)
- **Lambda Insights**: Lambda extension (layer) collecting system-level metrics (CPU, memory, cold starts)
- **Database Insights**: fleet-level monitoring for RDS/Aurora (successor of Performance Insights dashboards)
- **Application Signals** (GA 2024): APM - auto-instrumentation with ADOT/OpenTelemetry for EC2, ECS, EKS, Lambda; service maps, golden metrics (latency, errors, volume), **SLOs** (with SLO recommendations and service-level SLOs)
- **Synthetics canaries**: scripted Node.js/Python (Puppeteer/Playwright/Selenium) checks of endpoints, APIs, UI flows on a schedule - detect issues before users do; can run inside a VPC
- **RUM (Real User Monitoring)**: JavaScript snippet collecting real browser performance (page load, JS errors, Core Web Vitals) and sessions
- **Internet Monitor**: internet performance/availability between AWS and your end users (by city/ASN), uses AWS global network data; suggests improvements (e.g. CloudFront, other region)
- **Network Flow Monitor** (Dec 2024): agent-based TCP flow metrics (loss, RTT, retransmissions) between EC2/EKS workloads and AWS services - pinpoints whether network issues are on AWS side
- **Network Synthetic Monitor**: probes from AWS to on-premises IPs over DX/VPN for hybrid network monitoring
- **CloudWatch Investigations** (AI operations): generative-AI assisted root cause analysis from alarms
- **Contributor Insights**, **ServiceLens**/X-Ray trace map integration, Managed Grafana / Managed Prometheus as complementary services

> **Status (2026):** **CloudWatch Evidently** (feature flags / A/B testing) was discontinued on Oct 16, 2025 - use **AWS AppConfig** feature flags instead.

## CloudWatch vs EventBridge vs CloudTrail (exam)

- CloudWatch: performance metrics, logs, alarms => "is it healthy / how is it performing?"
- CloudTrail: API activity => "who did what, when, from where?"
- EventBridge: react to state changes and API calls (CloudTrail events) in near real time => "do something when X happens"
- AWS Config: resource configuration history and compliance => "what did the resource look like, is it compliant?"
