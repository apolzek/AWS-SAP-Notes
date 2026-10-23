# AWS Systems Manager (SSM)

- Is a product which lets us manage and control AWS and on-premise infrastructure
- SSM is agent based, which means an agent needs to be installed on Windows and Linux based AMIs (the SSM Agent is **preinstalled** on Amazon Linux 2/2023, Ubuntu, Windows Server and macOS AMIs provided by AWS)
- SSM manages inventory (what application are installed, files, network config, hw details, services, etc.) and can patch assets
- It can also run commands and manage desired state of instances (example: block certain ports)
- It provides a parameters store for configurations and secrets
- Finally it provides session manager used to securely connect to EC2 instances even in private VPCs
- **Unified Systems Manager console** (Nov 2024): one console across all accounts/regions of an AWS Organization (delegated admin), shows managed and *unmanaged* EC2 nodes and diagnoses why a node is not managed (missing agent/role/endpoint) <span style="color: red;">EXAM</span>
- Capabilities grouped as: Node tools (Fleet Manager, Session Manager, Run Command, State Manager, Patch Manager, Distributor, Inventory, Hybrid Activations, Compliance, Just-in-time node access), Change management (Automation, Maintenance Windows, Change Calendar, Change Manager), Operations (OpsCenter, Explorer, Incident Manager), Application tools (AppConfig, Application Manager, Parameter Store)

## Agent Architecture

- An instances needs the SSM agent to be installed in order to be able to be managed by the service
- It also needs an EC2 instance role attached to it which allows communication with the service (managed policy `AmazonSSMManagedInstanceCore`)
    - Alternative: **Default Host Management Configuration (DHMC)** - account/region setting that lets SSM manage all EC2 instances using a service role, no instance profile needed (requires IMDSv2)
- Instances require connectivity to the SSM service. This can be done via IGW or VPCE
    - Private subnets without NAT: interface endpoints `com.amazonaws.<region>.ssm`, `ssmmessages` (Session Manager) and `ec2messages`; plus S3 gateway endpoint for patches/logs/agent updates and `kms`/`logs` endpoints if session encryption/logging is used <span style="color: red;">EXAM</span>
- On the on-premises side we need to create managed instance activations (hybrid activations) - IAM service role + activation code and ID, nodes get `mi-` IDs
- For each activation we will receive an activation code and an activation ID
- Hybrid nodes: standard tier (free, up to 1,000 per account/region) or advanced tier (paid, >1,000 nodes, required for Session Manager to on-prem nodes)
- **Quick Setup**: one-click/org-wide configurations (host management, DHMC, patch policies, Config recording, DevOps Guru, etc.) deployed via CloudFormation StackSets

## SSM Run Command

- It allows us to run commands on managed instances
- It takes a Command Document and executes it using the agent installed on the instance
- It does this without using SSH/RDP protocol
- Command documents can be executed on individual instances, multiple instances based on tags or resource groups
- Command documents can be reused and they can have input parameters
- Rate Control: if we are running commands on lot of instances, we can control it by using rate control. It can be based on:
    - **Concurrency**: on how many instances must run the command at a time
    - **Error Threshold**: defines how many individual commands can fail
- Output of commands can be sent to S3 or we can send SNS notifications
- Commands can be integrated with EventBridge
- Output displayed in console is truncated at 48,000 characters - send to S3/CloudWatch Logs for full output
- No inbound ports, no bastion, no SSH keys needed - all actions logged in CloudTrail <span style="color: red;">EXAM</span>

## SSM Session Manager

- Browser/CLI shell (and RDP via Fleet Manager) to EC2, on-prem and edge nodes without opening inbound ports, bastion hosts or SSH keys
- Access controlled with IAM (`ssm:StartSession`, condition on resource tags); session data encrypted with KMS
- Session logging to S3 and/or CloudWatch Logs; session activity in CloudTrail; can run as specific OS user (Run As)
- Port forwarding (incl. to remote hosts, e.g. RDS through an instance) and SSH/SCP tunnelling over Session Manager
- **Just-in-time node access** (Apr 2025): zero standing privileges - operators request time-bound access, approval policies (auto-approve, manual approval, deny). To enforce it, remove `ssm:StartSession` from users' policies
- Exam: "replace bastion hosts, audit every command, no inbound SG rules" -> Session Manager

## SSM State Manager and Automation

- **State Manager**: associations that keep nodes in a defined state on a schedule (install agent, join domain, apply Chef recipes/Ansible playbooks, enforce config). Re-applies on drift
- **Automation**: runbooks (`AWS-*` and custom documents) for multi-step operational tasks across AWS resources - e.g. `AWS-StopEC2Instance`, golden AMI creation (`AWS-UpdateLinuxAmi`), remediation for Config rules, approval steps, rate control, multi-account/multi-region execution
- **Maintenance Windows**: schedule Run Command, Automation, Lambda, Step Functions tasks against targets
- **Change Calendar**: block/allow change windows (e.g. no changes during Black Friday) - Automation and State Manager respect it
- **Distributor**: package and distribute software (e.g. CloudWatch agent, third-party agents) to nodes
- **Inventory**: collect software/OS/network metadata; **Resource Data Sync** aggregates inventory from many accounts/regions into one S3 bucket (query with Athena/QuickSight) <span style="color: red;">EXAM</span>
- **Fleet Manager**: GUI to manage nodes (file system, users, performance, registry, RDP)

## SSM Operations tooling

- **OpsCenter**: central place for operational issues (OpsItems) created from CloudWatch alarms, EventBridge, Config, Security Hub; link runbooks for remediation
- **Explorer**: aggregated operations dashboard (OpsData) across accounts/regions
- **Application Manager**: view/operate resources grouped as an application (CloudFormation stack, tags, AppRegistry, EKS)
- **AppConfig**: feature flags and dynamic configuration with validators and gradual deployment + automatic rollback on CloudWatch alarm
- **Change Manager**: change requests with approvals and templates
    > **Status (2026):** Change Manager is no longer open to new customers (since Nov 7, 2025). Existing customers can continue; AWS recommends partner solutions.
- **Incident Manager**: response plans, on-call schedules, escalation, runbooks, chat channels
    > **Status (2026):** Incident Manager is no longer open to new customers (since Nov 7, 2025), no new features. AWS recommends OpsCenter for operational issues and partner tools (e.g. PagerDuty) for paging/on-call.

## SSM Patch Manager

- Allows to patch Linux and Windows instances running in EC2 or on-premises
- Concepts:
    - **Patch Baseline**: we can have many of these defined. Defines what should be installed (what patches, what hot-fixes)
    - **Patch Groups**: what resources we want to patch
    - **Maintenance Windows**: time slots when patches can take place
    - **Run Command**: base level functionality to perform the patching process. The command used for patching is `AWS-RunPatchBaseline` (operation `Scan` or `Install`)
    - **Concurrency** and **Error Threshold**: (see above)
    - **Compliance**: after patches are applied, system manager can check success of compliance compared to what is expected
    - **Patch policies** (via Quick Setup): recommended modern way - one policy to patch across all accounts/regions of an Organization, replaces manually wiring patch groups + maintenance windows
    - Patch groups: tag key `Patch Group` (or `PatchGroup`) associates instances with a baseline; an instance can be in only one patch group
    - Auto-approval rules: approve patches N days after release (e.g. 7 days) by classification/severity
- Patch Baselines:
    - For Linux: `AWS-[OS]DefaultPatchBaseline` - explicitly defines patches, example: `AWS-AmazonLinux2DefaultPatchBaseline`, `AWS-UbuntuDefaultPatchBaseline` - contain security updates and any critical update
    - For Windows: `AWS-DefaultPatchBaseline`, `AWS-WindowsPredefinedPatchBaseline-OS`, `AWS-WindowsPredefinedPatchBaseline-OS-Application` - Contains patches for MS applications as well.
    - Predefined baselines can't be modified - create a custom baseline to change approval rules, add approved/rejected patches or include non-security updates
- Exam: "patch hundreds of EC2 + on-prem servers in a defined window with compliance reporting" -> Patch Manager + Maintenance Windows (or patch policy), compliance view in SSM / Security Hub


