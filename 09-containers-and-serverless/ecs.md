# ECS - Elastic Container Service

- It is a service that accepts containers and orchestrates where and how to run those containers
- It is a managed container based compute service
- It runs on two classic modes: EC2 and Fargate (plus newer **ECS Managed Instances** and **ECS Anywhere/External**, see capacity section below)
- Cluster: is the place where containers run based on how we want them to run
- Containers are located in container registries (ECR, DockerHub)
- ECS uses **container definitions** to locate images in the container registries, which port should the image use, etc. providing information about the container we want to run
- **Task definition**: represents a self-contained application, can have one or many containers defined in it. A task in ECS defines the application as a whole
- Task definitions store the resources to be used (CPU, memory), networking configuration, compatibility (EC2 mode or Fargate) and also they store the task role (IAM role)
- **Task role** is IAM role that the task can assume. It gives permission to ECS containers to access AWS services
- **Task Execution Role**: to set up the container itself and maintain that, some tasks are performed. Container Agent performs those tasks for the container. So, the role required & assumed by Container Agent to set up the container is Task Execution Role
  - Typical permissions: pull image from ECR, write logs to CloudWatch, read secrets from Secrets Manager / SSM Parameter Store to inject as env vars
  - Exam trap: app can't read S3/DynamoDB => fix the **task role**; task fails to start pulling image or secrets => fix the **execution role**
- **Container instance role** (EC2 mode only): instance profile used by the ECS agent to register the instance in the cluster; tasks should NOT rely on it (use task roles)
- A task does not scale by its own and it is not HA
- **ECS service**: it is configured by a **service definition**. In a service we define how we want a task to scale, how many copies we like to run
- ECS services define scalability and HA for tasks

![ECS](images/ECS.png)

## ECS Cluster Modes

- Cluster mode define how much of the admin overhead is required for running containers in ECS (what parts do we manage and what parts does AWS manage)
- Cluster modes are:
  - EC2 Mode
  - Fargate Mode

### EC2 Mode

- Uses EC2 instances which are running inside of a VPC
- Since we are inside of a VPC, we can benefit from using multiple AZs
- When we create the cluster, we specify the initial size of the container instances (EC2 hosts)
- Horizontal scaling of EC2 instances is done by an ASG linked to the cluster via an **ASG capacity provider** with **managed scaling** (scales on `CapacityProviderReservation`); scaling of ECS tasks is done by **Service Auto Scaling** (Application Auto Scaling: target tracking on CPU/memory/ALB requests, step, scheduled)
- **Managed termination protection** + managed draining: instances running tasks are not terminated by scale-in
- With EC2 cluster mode we are paying for EC2 instances independently of what containers and how many of containers are running on them
- Possible to benefit from payment plans such as Reserved Instances, Savings Plans and Spot (Fargate has Compute Savings Plans and Fargate Spot, but **no RIs**)
- Required for: GPUs (also available on Managed Instances), specific instance types, privileged containers, very large tasks, daemon tasks, host/bridge networking

![ECSEC2](images/ECSEC2.png)

### Fargate Mode

- We don't have to manage EC2 instances for use as container host
- With Fargate there are no servers to manage
- AWS maintains a shared Fargate infrastructure platform offered to all users
- We gain access to resources from a shared pool, we don't have visibility for other customers
- A Fargate deployment still uses a cluster with a VPC which operates in AZs
- **ECS tasks are injected into the VPC with an ENI, they are running on the Fargate shared platform**
- With Fargate mode we only pay for the containers we using based on the resources they consume (vCPU + memory per second, 1 min minimum)
- Fargate limits: up to **16 vCPU / 120 GB** memory per task, ephemeral storage 20 GiB default up to **200 GiB**
- Only `awsvpc` networking (each task gets its own ENI, private IP and security group)
- Linux (x86_64 and ARM64/Graviton) and Windows containers
- **Fargate Spot**: up to ~70% discount, can be interrupted with a 2 minute warning (`SIGTERM`) => for fault tolerant/stateless/batch
- No SSH to hosts, no daemon sets, no GPUs, no privileged containers

![ECSFargate](images/ECSFargate.png)

## EC2 vs ECS (EC2) vs Fargate

- Generally never a good idea to run containers on EC2
- If we are already using containers => ECS
- Containers make sense if we want to isolate applications
- We generally pick **ECS - EC2 mode** if we have a large workload and the business is price conscious
- Historically EC2 mode was giving the most value for the price if we were using saving plans. **Nowadays we can have savings plan for Fargate and Lambda, so we should default to Fargate instead of EC2 mode**
- If we are overhead conscious, we should use Fargate
- For small/burst workloads we should use Fargate as well. Same is recommended for batch/periodic workloads

## Capacity Providers

- Capacity providers decide **where** tasks run; a **capacity provider strategy** (base + weight) splits tasks between providers
- Types:
  - `FARGATE` and `FARGATE_SPOT` (e.g. base 2 on FARGATE, then weight 1:3 FARGATE:FARGATE_SPOT)
  - **ASG capacity provider** (EC2 mode, managed scaling)
  - **ECS Managed Instances** (Sept 2025): AWS provisions, patches (Bottlerocket), scales and replaces EC2 instances in our account; we choose instance attributes (incl. GPU, network-optimized) and can use EC2 pricing models. "Fargate-like operations with EC2 choice". Small management fee on top of EC2
- **ECS Anywhere** (`EXTERNAL` launch type): run ECS tasks on on-prem servers/VMs registered via SSM agent, control plane stays in AWS

## Networking Modes

- `awsvpc`: task gets its own ENI (required for Fargate) => security groups per task, best practice. ENI density per instance limited => enable **ENI trunking** on EC2
- `bridge`: Docker virtual network, dynamic host port mapping (ALB can target dynamic ports)
- `host`: container uses the host network directly, no port mapping duplicates
- `none`: no external networking

## Load Balancing and Service-to-Service

- ECS services integrate with **ALB** (path/host routing, dynamic ports), **NLB** (TCP/UDP, static IP, PrivateLink)
- Multiple target groups per service supported
- Service discovery options:
  - **ECS Service Connect**: managed Envoy proxy sidecar injected by ECS, Cloud Map namespace, client-side load balancing, retries, outlier detection, traffic metrics. **Recommended** for ECS-to-ECS communication
  - **Service Discovery**: Cloud Map DNS records (A/SRV) per task, no proxy
  - **VPC Lattice**: cross-VPC/cross-account service networking (ECS services can register directly as Lattice targets)
- > **Status (2026):** AWS App Mesh reached **end of support on September 30, 2026** => migrate to ECS Service Connect (ECS) or VPC Lattice / Istio (EKS)

## Deployments

- **Rolling update** (default): `minimumHealthyPercent` / `maximumPercent`
- **Deployment circuit breaker**: auto rollback when tasks fail to reach steady state; can also roll back on CloudWatch alarms
- **Built-in blue/green** (July 2025) - native ECS, no CodeDeploy needed: bake time, lifecycle hooks (Lambda), instant rollback; works with ALB, NLB and Service Connect. **Linear and canary** strategies added Oct 2025
- **CodeDeploy blue/green** (classic exam answer): ALB/NLB with 2 target groups, traffic shifting `Canary`, `Linear`, `AllAtOnce`, test listener, hooks
- **External** deployment controller: third-party tools

## Storage

- **EFS**: shared, persistent, multi-AZ, works with Fargate and EC2 (access points + IAM authorization)
- **EBS volumes for tasks** (Jan 2024): one EBS volume attached per task at launch, Fargate and EC2, created from snapshot or empty, can be deleted on task stop => high-performance per-task storage (not shared)
- FSx for Windows File Server for Windows containers on EC2
- Bind mounts / ephemeral storage: lifetime of the task

## Operations and Security

- **ECS Exec**: interactive shell / commands into running containers (Fargate and EC2) via SSM Session Manager, no SSH, auditable via CloudTrail/S3/CloudWatch logs
- Secrets: reference Secrets Manager / SSM Parameter Store in the container definition (`secrets`), injected at start by the execution role
- Logging: `awslogs` driver (CloudWatch Logs) or **FireLens** (Fluent Bit sidecar => OpenSearch, S3, Firehose, third-party)
- Observability: Container Insights (enhanced observability), X-Ray / ADOT sidecar
- Images: ECR with image scanning (Inspector), pull-through cache, cross-region/cross-account replication; **SOCI** index for lazy loading to reduce Fargate start time
- Task placement (EC2 only): strategies `binpack`, `spread`, `random`; constraints `distinctInstance`, `memberOf` (cluster query language)

## ECS Express Mode

- Launched Nov 2025, no additional charge
- Give it a container image (+ execution/infrastructure roles) => ECS creates cluster, task definition, Fargate service, ALB (shared up to 25 services), HTTPS URL/domain, auto scaling, logs
- All resources live in our account and can be modified later => "PaaS speed, full ECS escape hatch"
- Replacement recommended by AWS for App Runner

## Exam Tips

- "Run containers, least operational overhead" => ECS on **Fargate** (or Express Mode for simple web apps)
- "Containers on-prem managed from AWS" => **ECS Anywhere** / EKS Anywhere / EKS Hybrid Nodes
- "Reduce cost of stateless containers" => Fargate Spot via capacity provider strategy, Graviton, Compute Savings Plans
- "Each task needs its own security group" => `awsvpc`
- "Batch job containers, queue-based" => AWS Batch (on ECS/Fargate/EKS) or ECS task triggered by EventBridge/Step Functions (`ecs:runTask.sync`)
- "Scheduled container job" => EventBridge Scheduler => ECS `RunTask`
- Scale ECS service on SQS backlog => target tracking on custom metric "backlog per task"
