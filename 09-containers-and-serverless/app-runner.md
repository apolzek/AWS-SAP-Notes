## AWS App Runner

> **Status (2026):** App Runner is **closed to new customers since April 30, 2026**. Existing customers can keep using it (including creating new services), AWS keeps security/availability investment but **no new features**. AWS recommends **Amazon ECS Express Mode** for new workloads and as the migration target (blue/green with Route 53 weighted DNS).

- Fully managed service for deploying a containerized apps and APIs
- It is a PaaS solution with all components managed by AWS - we just bring our code and container image
- Integrates with GitHub and ECR for getting code or already built containers
- Provides the following features:
    - Auto scaling
    - Load balancing
    - Health checks
    - Observability
- App Runner applications can run inside in a VPC
- Networking details:
    - Inbound: public by default; private services via an `apprunner.requests` interface VPC endpoint
    - Outbound: public by default; **VPC connector** to reach private resources (RDS, ElastiCache) in our VPC
- Scales on **concurrent requests per instance** (configurable), can scale to a provisioned minimum (memory billed while idle, CPU only when serving)
- Custom domains with managed TLS certificates, AWS WAF association supported
- Services can be paused/resumed

## App Runner vs Alternatives (Exam)

- "Simplest way to run a web container, least operational overhead, new project" => **ECS Express Mode** (ECS + Fargate + ALB + autoscaling + URL provisioned from one image)
- Need full control of networking/sidecars/deployments => ECS on Fargate
- Kubernetes required => EKS
- Event-driven / short requests, scale to zero => Lambda (or Lambda Web Functions for plain HTTP servers)
- Legacy PaaS with platform runtimes (Java, .NET, PHP, etc.) => Elastic Beanstalk
