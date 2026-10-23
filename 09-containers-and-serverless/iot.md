# AWS IoT

- IoT - Internet of Things

## AWS IoT Core

- AWS IoT Core is a product set in AWS, used for managing millions of small low-power IoT devices
- IoT devices can be temp, wind, water sensors, light sensors, valve control sensors, etc.
- All of these need to be registered into a system to allow secure communication for managing them: provisioning, updates and control
- Communication to or from devices is likely to be unreliable, so AWS provides **device shadows**: virtual representations of actual devices, having the same configuration registered for the actual device. We can read from them the last communicated data, essentially the device communicates with the shadow, the last registered data can be retrieved anytime afterwards. => Help dealing with unreliability of connections with the IoT devices.
- Device messages are sent JSON format, using MQTT protocols (MQTT 3.1.1/5, MQTT over WebSockets, HTTPS, LoRaWAN via IoT Core for LoRaWAN)
- AWS IoT provides rules: event-driven integration with other AWS Services
- IoT Gateway: HA and scalable. Endpoint that all of the IoT devices communicate with.
- IoT Rule: enables integration with other AWS services (DynamoDB, Lambda, S3, Step Functions, etc) by matching rules with incoming data from the IoT devices.
- Security: each device (thing) authenticates with an **X.509 certificate** (mutual TLS), permissions via **IoT policies** attached to certificates; custom authorizers (Lambda) for tokens; Cognito for app users
- Registry: things, thing types, thing groups; **fleet provisioning** for bulk/just-in-time certificate provisioning
- **Rules engine**: SQL-like statements (`SELECT temp FROM 'sensors/+/data' WHERE temp > 50`) => actions to Lambda, DynamoDB, S3, Kinesis Data Streams, Firehose, SQS, SNS, Step Functions, Timestream, OpenSearch, SiteWise, HTTP, Kafka (MSK), CloudWatch, republish; error action for failures
- **Basic Ingest**: publish to `$aws/rules/<rule>` topics to skip the message broker cost
- **Jobs**: push OTA firmware/config updates to fleets; **Device Advisor** for testing; **Secure Tunneling** for remote access to devices behind firewalls
- Exam pattern: millions of devices => IoT Core => rule => Kinesis Data Streams/Firehose => S3 => Athena/QuickSight (analytics), or => Lambda/DynamoDB (real-time state)
- AWS IoT architecture:

![AWS IoT architecture](images/AWSIoT.png)

## AWS IoT Device Management

- Helps us to register, organize, monitor and remotely manage IoT devices at scale

## AWS IoT Device Defender

- Used to audit configurations, authenticate devices, detect anomalies and receive alerts to help us secure our IoT device fleet

## AWS IoT 1-Click

> **Status (2026):** **Discontinued** - end of support on **December 16, 2024** (closed to new customers since Dec 2023). Rebuild on AWS IoT Core.

- Used to launch AWS Lambda functions from IoT devices
- We can also create actions in the cloud or on-premises

## AWS Greengrass

> **Status (2026):** Greengrass **V1 reached end of support on October 7, 2026** (resources no longer accessible). Use **Greengrass V2** (component-based, open-source Nucleus, Nucleus Lite for constrained devices).

- AWS Greengrass is an extension of the services provided by AWS IoT, moving those services closer to the edge
- Greengrass allow some services like compute, messages, data management, sync and ML capabilities to run from edge devices
- Devices with Greengrass Core software can locally run Lambda functions or containers => compute can run locally without leaving the local network
- Greengrass provides local device shadows which are synced back to AWS
- Allows messaging using MQTT
- Allows local hardware access for Lambda functions
- V2: software is deployed as **components** (Lambda functions, Docker containers, native processes) via deployments to thing groups; works offline and syncs when connectivity returns
- Stream Manager: buffer and export local data streams to Kinesis, S3, SiteWise
- Use case: low-latency local decisions, intermittent connectivity, local ML inference (SageMaker models at the edge)

## AWS IoT Analytics

> **Status (2026):** **Discontinued** - closed to new customers July 24, 2024, end of support on **December 15, 2025** (console and resources no longer accessible). Alternatives: IoT Core rules => Kinesis/Firehose => S3 + Glue/Athena/QuickSight, or AWS IoT SiteWise for industrial data.

- Used to run analytics on IoT data and get insights to make better and more accurate decisions
- Supports up to petabyte of data from millions of devices

## AWS IoT Events

> **Status (2026):** **Discontinued** - closed to new customers May 20, 2025, **end of support May 20, 2026**. Alternatives: IoT Core rules + Lambda/Step Functions (state machines), DynamoDB for state, or SiteWise alarms for industrial data.

- Used to detect and respond to events from IoT sensors and applications
- We can ingest data from multiple sources to detect the state of our processes or devices and proactively manage maintenance schedules

## AWS IoT SiteWise

- Simplifies collecting, organizing and analyzing industrial equipment data
- We can organize sensor data streams from multiple production lines and facilities to drive efficiencies across locations
- Asset models/hierarchies, computed metrics/transforms, alarms
- **SiteWise Edge**: collect and process data on-prem (OPC UA, Modbus) on Greengrass-based gateways, then sync to the cloud

## AWS IoT TwinMaker

- Note: TwinMaker is **not** a rename of AWS IoT Things Graph - Things Graph was a separate service which is now **discontinued**

- Used to create digital twins of real-world systems such as buildings, factories, industrial equipment and production lines
- We used this to quickly pinpoint and address equipment and process anomalies from the plant floor to improve worker productivity and efficiency
- Knowledge graph of entities/components connected to data in SiteWise, Kinesis Video Streams, S3 or custom connectors; 3D scenes; dashboards via the Grafana plugin

## AWS IoT Device Management / Defender details

- Device Management: fleet indexing and search, jobs (OTA), remote commands, secure tunneling (the **Fleet Hub** dashboard feature was discontinued on October 18, 2025)
- Device Defender: **Audit** (checks configurations against best practices, e.g. shared certificates, overly permissive policies) and **Detect** (anomalies in device behaviour, ML Detect), mitigation actions (move to quarantine thing group, revoke certificate)
