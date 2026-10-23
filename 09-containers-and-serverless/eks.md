# EKS - Elastic Kubernetes Service

## Kubernetes 101

- It is an open source container orchestration system
- We used to automate the deployment, scaling and management of containerized applications
- It is a cloud-agnostic product, can be used on-premises as well
- Cluster Structure:
  - A Kubernetes cluster is a HA cluster of compute resources organized to work as one unit
  - Cluster starts with the Cluster Control Plane: manages the cluster, scheduling, applications, scaling and deploying
  - Compute in Kubernetes is provided via Cluster Nodes: VM or physical servers which function as a worker in the cluster. These run the containerized applications
  - Software that will run on each node:
    - On each of the nodes runs a container runtime: `containerd` or other software used to handle container operations
    - `kubelet`: the agent which interacts with the control plane is. This uses the Kubernetes API to communicate with the control plane
- Cluster details:
  - Pods:
    - The smallest units of computing in Kubernetes
    - Pods can have multiple containers and provide shared storage and networking for them
    - It is common to have one container per pod, although we can have multiple container in a pod
    - Pods are not permanent: they can be deleted when finished with a job, evicted because of lack of resources or when a node fails
  - Control plane runs the following software:
    - `kube-apiserver`:
      - The front-end for the Kubernetes control
      - It is what nodes and other cluster elements interact with
      - Can be horizontally scaled for HA and performance
    - `etcd`:
      - HA key/value store
      - Backing store for the cluster
    - `kube-scheduler`:
      - Identifies any pods within a cluster that does not have a node assigned
      - Assigned pods based on resources, deadlines, affinity, data locality and any other constraints
    - `cloud-controller-manager`:
      - Optional component
      - Provides cloud-specific control logic
      - It allows us to link Kubernetes with cloud providers APIs such as AWS
    - `kube-controller-manager`:
      - It is a collection of processes:
        - Node Controller: monitoring and responding to node outages
        - Job Controller: one-of tasks (jobs)
        - Endpoint Controller: populates endpoints
        - Service Account and Token Controllers: accounts/API tokens
  - kube-proxy:
    - Every node runs it
    - It is a networking proxy
    - It coordinates networking with the control plane
    - It helps implement "services" and configures rules allowing communications with pods from inside or outside of the cluster
- Ingress: exposes a way into a service from outside to the cluster
- Ingress Controller: a piece of software with arranges the underlying hardware to allow ingres (example: AWS LB Controller which use ALB/NLB)
- Persistent Storage (PV): they are volumes whose lifecycle lives beyond any one single pod that is using it. "Normal" long running storage

![Kubernetes 101](images/Kubernetes101.png)

## Elastic Kubernetes Service 101

- It is an AWS managed implementation of Kubernetes as a service
- EKS can be run in different ways:
  - On AWS itself as a product
  - On Outposts: racks and servers operating in on-premises locations, but controlled and managed by AWS (extended cluster = control plane in region; local cluster = control plane on the Outpost, survives disconnects)
  - EKS Anywhere: based on EKS Distro (open source Kubernetes distribution used by AWS) to deploy EKS clusters running on on-premises or anywhere else. Possible to pay for Enterprise Subscription and have 24/7 support. Control plane runs on-prem (we manage it)
  - **EKS Hybrid Nodes** (Dec 2024): control plane stays managed in the AWS region, our **on-prem/edge servers join as worker nodes** (needs private connectivity via DX/VPN, routable pod/node CIDRs). Billed per vCPU-hour
- The EKS control plane is managed by AWS and scales based on load. It runs across multiple AZs
- Integrates with other AWS services such as ECR, ELB, IAM, VPC
- EKS Cluster = EKS Control Plane and EKS Nodes
- etcd is also managed by AWS and distributed across multiple AZs
- Nodes can be the following types:
  - Self managed: EC2 instances managed by us
  - Managed node groups: still EC2 instances, but EKS manages the provisioning and lifecycle managing
  - Fargate: nodes are managed by EKS, we don't have to worry about scaling and optimizing clusters (similar to ECS Fargate)
    - Fargate profiles select pods by namespace + labels; one pod per micro-VM
    - Limitations: no DaemonSets, no GPUs, no privileged pods, no EBS (EFS is supported), no Spot, only private subnets
  - **EKS Auto Mode** (Dec 2024): AWS manages nodes end to end - Karpenter-based provisioning, Bottlerocket AMIs, patching/upgrades (nodes max 21 days lifetime), core add-ons (VPC CNI, CoreDNS, kube-proxy, EBS CSI, AWS Load Balancer Controller) run off-cluster. Extra management fee per instance on top of EC2. Default answer for "Kubernetes with least operational overhead" while keeping EC2 features (GPU, Spot, RI/SP)
- Choosing between node types depends on for what we want to use the cluster. If we want Windows nodes, GPU, Inferentia, Bottlerocket, Outposts, Local zones, etc. we need to check which types of nodes support the feature we require
- Persistent storage on EKS: it can use EBS, EFS, FSx for Lustre, FSx for NetApp ONTAP, FSx for OpenZFS, S3 (Mountpoint for S3 CSI driver) as storage providers via CSI drivers
  - EBS = single AZ, ReadWriteOnce; EFS = multi-AZ, ReadWriteMany

![EKS](images/EKS.png)

## Node Scaling

- **Cluster Autoscaler**: scales ASGs of node groups based on pending pods (slower, tied to node group instance types)
- **Karpenter**: open source, provisions right-sized EC2 instances directly (no ASGs), mixes instance types, Spot + On-Demand, consolidation (bin-packing to cut cost), drift/expiry for upgrades. Preferred today; core of Auto Mode
- Pods scaling: HPA (metrics), VPA, **KEDA** (event-driven, e.g. SQS queue depth)
- Spot on EKS: diversify instance types, handle interruptions (Karpenter or Node Termination Handler)

## Security and Access

- **Cluster access (authentication to the K8s API)**:
  - **Access entries** (EKS API): map IAM principals to Kubernetes permissions via EKS API + access policies (e.g. `AmazonEKSClusterAdminPolicy`). Recommended, manageable via IaC, cluster creator admin can be removed
  - `aws-auth` ConfigMap: legacy, deprecated
- **Pod permissions to AWS services**:
  - **EKS Pod Identity** (Nov 2023): Pod Identity Agent add-on + association (namespace/service account => IAM role) via EKS API; no OIDC provider needed, same role reusable across clusters, supports session tags (ABAC). Recommended
  - **IRSA** (IAM Roles for Service Accounts): OIDC provider per cluster + role trust policy per cluster. Still needed for Fargate pods and some cases
  - Exam trap: never put credentials in pods or rely on the node instance role for app permissions
- Endpoint access: public, public + private, or **private only** (API server reachable only from VPC/peered/DX networks)
- Secrets: envelope encryption of K8s secrets with KMS; or Secrets Store CSI driver (Secrets Manager / Parameter Store)
- Network policies (VPC CNI native support), **security groups for pods**
- GuardDuty EKS Protection / Runtime Monitoring, Inspector for images in ECR

## Networking

- **Amazon VPC CNI**: pods get real VPC IPs => can exhaust subnet IPs
  - Fixes: **prefix delegation** (/28 per ENI slot), **custom networking** (pods in secondary CIDR e.g. `100.64.0.0/10`), **IPv6** clusters
- Ingress: **AWS Load Balancer Controller** => ALB for `Ingress`, NLB for `Service type: LoadBalancer`; IP target mode works with Fargate
- Service-to-service across VPC/accounts: **VPC Lattice** (Gateway API controller)
- > **Status (2026):** AWS App Mesh reached end of support on **September 30, 2026** - use VPC Lattice, Istio or other meshes

## Versions and Pricing

- Control plane: **$0.10/hour per cluster** in standard support
- Each Kubernetes version: **14 months standard support** + **12 months extended support** (**$0.60/hour** per cluster). Clusters are auto-upgraded after extended support ends; extended support can be disabled (upgrade policy)
- Upgrades: control plane first (one minor version at a time), then nodes/add-ons; EKS **Upgrade Insights** flags deprecated APIs
- **Provisioned Control Plane** (late 2025): pre-provisioned API server capacity tiers (XL, 2XL, 4XL, 8XL) for predictable high-scale workloads, extra hourly fee; 99.99% SLA for these clusters (2026)
- EKS supports **ultra-scale** clusters up to 100,000 nodes (AI/ML training)

## Add-ons and EKS Capabilities

- **EKS add-ons**: AWS-managed lifecycle of VPC CNI, CoreDNS, kube-proxy, EBS/EFS CSI, Pod Identity Agent, ADOT, GuardDuty agent, etc.
- **EKS Capabilities** (re:Invent 2025): fully managed **Argo CD** (GitOps), **ACK** (AWS Controllers for Kubernetes - manage AWS resources like S3/RDS via K8s manifests) and **kro** (compose resources into custom APIs), running in AWS-owned infrastructure
- Observability: CloudWatch Container Insights / Observability add-on, Amazon Managed Service for Prometheus + Managed Grafana, control plane logs to CloudWatch

## ECS vs EKS (Exam)

- Already on Kubernetes, need portability/multi-cloud/open-source ecosystem (Helm, operators) => **EKS**
- AWS-native, simpler, less operational overhead, no K8s skills => **ECS** (Fargate)
- Migrate on-prem Kubernetes with minimal changes => EKS (Auto Mode for least ops); keep workers on-prem with central control => Hybrid Nodes; fully disconnected on-prem => EKS Anywhere
- Cost: ECS control plane is free, EKS charges per cluster-hour (watch extended support fees on old versions)
