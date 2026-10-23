# AWS Batch

- Managed compute service commonly used for large scale data analytics and processing
- It is a managed batch processing product
- Batch Processing: jobs that can run without end-user interaction, or can be scheduled to run as resources permit
- AWS Batch lets us worry about defining jobs, it will handle the underlying compute and orchestration
- No additional charge for Batch itself => we pay for the underlying EC2/Fargate/EKS resources
- AWS Batch core components:
  - Job: script, executable, docker container submitted to batch. Jobs are executed using containers in AWS. The job defines the work. Jobs can depend on other jobs
  - Job Definition: metadata for a job, including IAM permissions, resource configurations, mount points, etc.
  - Job Queue: jobs are submitted to a queue, where they wait for 1 or + compute environments. Queues can have a priority
  - Compute Environment: the compute resources which do the actual work. Can be managed by AWS or by ourselves. We can configure the instance type, vCPU amount, spot price, or provide details on a compute environment we manage (ECS)

![BatchJob](images/BatchJob.png)

- Integration with other services:

![AWS Batch Integration](images/AWSBatch2.png)

## Job Types and Scheduling

- **Array jobs**: one submission => up to **10,000 child jobs** (e.g. process 10,000 S3 files in parallel), each gets `AWS_BATCH_JOB_ARRAY_INDEX`
- **Multi-node parallel jobs**: one job spanning multiple EC2 instances (HPC, MPI, distributed ML training). EC2 only, can use **EFA** and cluster placement groups
- **Job dependencies**: `SEQUENTIAL`/`N_TO_N` for arrays, or depend on other job IDs
- **Fair-share scheduling**: share identifiers + weights => many teams/users share one queue fairly (instead of FIFO)
- Retry strategies (per exit code), job timeouts, priority across queues
- Triggers: **EventBridge** schedules/rules (e.g. S3 object created => submit job), **Step Functions** (`.sync` integration) for workflows, Lambda, MWAA
- Logs to CloudWatch Logs, job state changes emitted to EventBridge

## Compute Environment Options

- **EC2** On-Demand or **Spot** (allocation strategies: `BEST_FIT_PROGRESSIVE`, `SPOT_CAPACITY_OPTIMIZED`, `SPOT_PRICE_CAPACITY_OPTIMIZED` recommended for Spot)
- **Fargate / Fargate Spot**: serverless, no instances to manage, fast start, per-job isolation. Limits: no GPUs, max 16 vCPU / 120 GB per job, no multi-node parallel
- **Amazon EKS**: Batch schedules jobs as pods on our existing EKS cluster nodes (EC2)
- GPU jobs supported on EC2 (P/G instances), also Trainium/Inferentia
- Exam: "cheapest for fault-tolerant batch" => Batch + Spot (with retries). "No infra management for container batch jobs" => Batch on Fargate

## AWS Batch vs Lambda

- Lambda has a 15 minutes execution limit, for longer workflows we should use Batch
- Lambda has limited disk space in the environment, we can fix this by using EFS => but this means VPC Lambda
  - Lambda `/tmp` is configurable up to **10 GB** ephemeral storage, max 10 GB memory / 6 vCPU
- Lambda is fully serverless with limited runtime selection (container images up to 10 GB and custom runtimes are supported)
- Batch is **NOT** serverless, it uses Docker with any runtime
  - Exception: Batch on **Fargate** is serverless (no EC2 to manage)
- Batch does not have a time limit for execution (optional job timeout can be set)
- Lambda **durable functions** (Dec 2025) can checkpoint and suspend for up to 1 year (waits are not billed), but each invocation still has the 15 min limit => for long continuous compute still use Batch/ECS

![BatchLambda](images/BatchLambda.png)

## AWS Batch vs Others

- Batch vs **ECS/Fargate tasks directly**: Batch adds queues, priorities, retries, array jobs, dependencies, fair-share
- Batch vs **EMR**: EMR for Spark/Hadoop frameworks on big data; Batch for generic containerized jobs
- Batch vs **Glue**: Glue for serverless Spark/Python ETL with Data Catalog integration
- Batch vs **Step Functions**: Step Functions orchestrates; Batch executes the heavy compute (often combined)
- Batch vs **AWS ParallelCluster**: ParallelCluster = traditional HPC cluster with Slurm scheduler

# Managed vs Unmanaged AWS Batch

- Managed:
  - AWS Batch manages capacity based on our workload needs
  - We decide the instance type/size and if we want to use on-demand or spot instances
  - We can determine our own max spot price
  - We need to create VPC gateways for access to the resources if using **private subnets**
  - Using the managed compute environment we allow AWS the manage all the infra on our behalf, we just need to tweak some high level values
  - Instances scale between min/max/desired vCPUs; set min vCPU = 0 to pay nothing when idle
  - Custom AMIs via launch templates are supported
- Unmanaged:
  - We manage everything, we create the environment, we direct AWS to use that environment
  - Generally used if we have a full compute environment ready to go and we want to control the costs
  - Requires specific AMI for the EC2 instances (must meet Amazon ECS container instance AMI requirements)
