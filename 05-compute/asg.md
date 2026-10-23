# ASG - Auto Scaling Groups

- Provide the ability to implement a self-healing architecture
- ASGs make use of configurations defined in launch templates or launch configurations
- ASGs are using one version of a launch template/configuration (can point to `$Default` or `$Latest`)
- ASG have 3 important values defined: _Minimum_, _Desired_ and _Maximum_ size. Desired size has to be between the minimum and the maximum size (min <= desired <= max).
- ASG provides on foundational job: keeps the size of running instances at the desired size
- ASG define where the EC2 instances are launched. They are linked to a VPC, and can be configured with subnets to use to deploy instances into.
- **Scaling Policies**: update the desired capacity based on some metric (CPU usage, number of connections, etc.)
  - They are rules defined by us which can adjust the values of an ASG
  - Scaling policies are used with ASG.
  - Scaling types:
    - **Manual Scaling** : Manually adjusts the desired capacity.
    - **Scheduled Scaling**: Scheduling based on known time window
    - **Dynamic Scaling**: 3 subtypes
      - **Simple Scaling**: Based on Metric. Example "CPU above 50% +1", "CPU Below 50% -1"
      - **Step Scaling**: scaling based on difference, allowing to react quicker (preferable to Simple except when priority is simplicity)
      - **Target Tracking**: example desired aggregate CPU = 40%. Not all metrics are supported by target tracking scaling
    - **Predictive Scaling**: ML forecast of load (from up to 14 days of history) to scale out **ahead** of cyclical demand
- **Cooldown Period**: a value in seconds, controls how long to wait after a scaling action happened before starting another action
- ASG monitor the health of instances, by default using the EC2 health checks
- ASG can integrate with load balancers: ASG can add/remove instances from a LB target group
- ASG can use the LB health checks in case of EC2 health checks

## Launch Templates vs Launch Configurations

> **Status (2026):** Launch Configurations are deprecated. No new EC2 instance types/features since Dec 31, 2022, and accounts created on or after **Oct 1, 2024** can't create launch configurations at all. Migrate to Launch Templates.

- Launch Configurations: immutable (no versioning, must be recreated to change), single instance type, no Spot/On-Demand mix, no newer EC2 features or instance types
- **Launch Templates**: versioned, can be used for standalone EC2 launches, ASG, EC2 Fleet; support mixed instances policy, multiple network interfaces, Capacity Reservations / Capacity Blocks targeting, dedicated hosts, placement groups, IMDSv2 options, all newer features
- Exam tip: any answer requiring Spot + On-Demand mix, multiple instance types or new instance families requires a **Launch Template**

## Mixed Instances Policy and Attribute-based Selection

- One ASG can run multiple instance types and purchase options (On-Demand + Spot)
- `OnDemandBaseCapacity` + `OnDemandPercentageAboveBaseCapacity` => e.g. base of 2 On-Demand, then 20% On-Demand / 80% Spot
- Spot allocation strategy: `price-capacity-optimized` (recommended), `capacity-optimized`, `lowest-price`; On-Demand strategy: `lowest-price` or `prioritized` (order of the overrides - use with RIs/SPs)
- **Attribute-based instance type selection (ABIS)**: specify vCPU, memory, architecture, etc. instead of a list of types => new instance generations are picked up automatically
- Instance **weights**: each type contributes a number of capacity units (e.g. by vCPU)
- **Capacity Rebalancing**: proactively replace Spot instances that receive a rebalance recommendation
- Can launch into **On-Demand Capacity Reservations** / Capacity Blocks (capacity reservation preference in the launch template); AZ distribution strategy can fill reservations first, then balance across AZs (2026)

## Scaling Processes

- `Launch` and `Terminate`: if Launch is suspended, the ASG won't scale out / if Terminate is suspended the ASG won't scale in
- `AddToLoadBalancer`: add instance to LB
- `AlarmNotification`: control is the ASG reacts to CloudWatch alarms
- `AZRebalance`: balances instances evenly across all of AZs
- `HealthCheck`: controls if instance health checks are on/off
- `ReplaceUnhealthy`: controls if instances are replaced in case there are unhealthy
- `ScheduledActions`: controls if scheduled actions are on/off
- `Standby`: suspend any activities of ASG in a specific instance
- `InstanceRefresh`: controls instance refresh operations
- Suspending `ReplaceUnhealthy` / `HealthCheck` / `Terminate` is the classic trick to troubleshoot an instance without the ASG replacing it (or put it in `Standby`)
- **Instance scale-in protection**: protect individual instances from being terminated on scale-in
- **Termination policies**: `Default` (AZ balance first, then oldest launch template/config, then closest to next billing hour), `OldestInstance`, `NewestInstance`, `OldestLaunchTemplate`, `AllocationStrategy`, custom Lambda policy
- **Deletion protection** (2026): prevent deletion of the group; IAM condition key `autoscaling:ForceDelete`

## ASG Considerations

- ASG are free, we pay only for the instances provisioned
- We should use cool downs to avoid rapid scaling
- We should use smaller instances for granularity
- ASG integrates with ALBs
- ASG defines when and where, Launch Templates defines what
- ASG spans multiple AZs of one region (single region only) - multi-region = one ASG per region + Route 53 / Global Accelerator
- **AZ rebalancing**: when AZs become unbalanced, ASG launches new instances before terminating old ones (can temporarily exceed max by 10% or 1 instance)
- **AZ distribution** strategies: `balanced-best-effort` (default, launches into other AZs if one fails) or `balanced-only`
- **Zonal shift** (Application Recovery Controller) integration: ASG stops launching into an impaired AZ and shifts capacity away
- Scale on SQS: use custom metric **backlog per instance** (`ApproximateNumberOfMessagesVisible` / running instances) with target tracking - AWS recommended pattern

## Lifecycle Hooks

- Allow to configure custom actions which can occur during ASG actions
- Lifecycle hooks are run AFTER instance launch/termination but BEFORE InService/Terminated
- When an ASG scales out/in => instances may pause within the flow to allow execution of lifecycle hooks (WAIT)
- We can specify a timeout (3600s by default) for the lifecycle action, after the pause => the system can decide if the ASG process continues or is abandoned
- We can resume the ASG process by calling `CompleteLifecycleAction`
- Default result on timeout: `ABANDON` (launch => instance terminated) or `CONTINUE`; extend with `RecordLifecycleActionHeartbeat` (max 48h or 100x heartbeat timeout, whichever is smaller)
- Lifecycle event hooks can be integrated with EventBridge, SNS or SQS notifications (and Lambda via EventBridge)
- Use cases: install software/pull config before `InService`, register with external systems, **copy logs/flush data before termination**, deregister from DNS/licensing

![ASG Lifecycle Hooks](images/ASGArchitecture3.png)

## Scaling Policies

- ASGs don't need scaling policies, they can work just fine with none
- When created without a scaling policy, an ASG has static values for `MinSize`, `MaxSize` and `Desired` capacity
- Manual scaling: we manually adjust the values listed before; useful for testing or urgent situations or when we need to hold capacity at fixed number of instances
- In addition to manual scaling, we had different types of dynamic scaling policies. Each of these adjusts the desired capacity of an ASG based on a certain criteria
- Dynamic scaling policies:
  - **Simple Scaling**:
    - We define action which occur when a alarm goes to ALARM state. For example: add one instance if `CPUUtilization` is above 40%
    - Helps infrastructure scale out/in based on demand
    - This scaling is **NOT FLEXIBLE**, add/remove static number of instances based on the status of an alarm
  - **Step Scaling**:
    - Adjust number of instances based on a number of step adjustments that vary based on the size of the alarm range
    - Example:
      - If the CPU usage is between 50-60%, do nothing
      - If the CPU usage is between 60 and 70%, add one instance
      - If the CPU usage is between 70 and 80%, add two instances
      - Finally, add 3 instances if the CPU usage is above 80%
    - It is generally better compared to Simple Scaling, allows us to adjust better to changing load patterns
  - **Target Tracking**:
    - Comes with a predefined set of metrics: `CPUUtilization`, `AvgNetworkIn`, `AvgNetworkOut`, `AlbRequestCountPerTarget`
    - We define an ideal value, a target we want to track against for a supported metric
    - The ASG calculates the scaling adjustment based on the metric and the target value
    - The ASG keeps the metric at the target value we specified and adjusts the capacity as required
  - **Scaling based on SQS**:
    - `ApproximateNumberOfMessagesVisible`: scaling based on the number of messages currently in the SQS queue.
- **AWS recommends using Step Scaling instead of Simple Scaling policy** (and target tracking as the default choice)
  - Simple scaling waits for the cooldown to finish before reacting again; step and target tracking keep reacting while scaling activities are in progress
  - **Default instance warmup** (recommended to enable): new instances don't contribute to aggregated metrics until warmed up - replaces per-policy warmup/cooldowns
- **Predictive Scaling**:
  - Forecasts capacity for the next 48h from historical CloudWatch data (min 24h history, uses up to 14 days), scales out **before** the load arrives
  - Modes: `ForecastOnly` (evaluate) or `ForecastAndScale`; can pre-launch with `SchedulingBufferTime`
  - Best for cyclical patterns (business hours, batch windows) and apps with long initialization; combine with target tracking for unexpected spikes
- **Scheduled actions**: set min/max/desired at specific times (cron, time zone aware) - for known events
- Multiple policies at once: ASG picks the policy that provides the **largest capacity** (scale-out wins)
- Custom metric math in target tracking is supported (e.g. backlog per instance from SQS)

## ASG Health Checks

- ASGs assess the health of instances within their group using health checks
- If an instance fails health checks, it is replaced within the group => automatic healing
- There 3 different types of health checks that can be used by ASGs:
  - EC2 (default):
    - Any of those is viewed as UNHEALTHY: Stopping, Stopped, Terminated, Shutting Down, Impaired (not 2/2 status checks)
  - ELB (can be enabled)
    - An instance is viewed as HEALTHY if it is running and it is passing the ELB health checks
    - ELB health checks can be more application aware (Layer 7)
  - Custom
    - Instances can be marked HEALTHY/UNHEALTHY by an external system
  - **EBS** health checks (impaired attached volumes) and **VPC Lattice** health checks can also be enabled
- Health check grace period:
  - It is configurable value, by default is 300s in the console (0s when created via CLI/SDK)
  - It is a delay before health checks starting to check on a specific instance
  - Allows system launch, bootstrapping and application start
  - Prefer lifecycle hooks / default instance warmup over a very long grace period
- Exam trap: ASG with ELB but only EC2 health checks => app crashes are NOT detected (instance still "running") => enable ELB health checks

## Warm Pools

- Pool of **pre-initialized** instances kept alongside the ASG to reduce scale-out latency for apps with long boot/initialization
- Instance states in the pool: `Stopped` (pay only EBS), `Hibernated` (RAM preserved, pay EBS) or `Running` (pay full)
- Sizing: by default `max size - desired`, or `MinSize` / `MaxGroupPreparedCapacity`; instance reuse policy can return instances to the pool on scale-in
- Lifecycle hooks run on the way into the pool (initialize once) and on the way out (finalize)
- Exam tip: "app takes 10+ min to initialize and scale-out is too slow" => warm pool (or AMI baking + predictive scaling)

## Instance Refresh

- Rolling replacement of instances to roll out a new launch template version / AMI / instance types
- Settings: `MinHealthyPercentage` (default 90%), `MaxHealthyPercentage` (up to 200% => launch before terminate), instance warmup, **checkpoints** (pause at % for validation)
- **Skip matching**: only replace instances that don't match the desired configuration
- **Auto rollback** to the previous configuration on failure or when CloudWatch alarms go into `ALARM`; can also be rolled back manually
- **Instance maintenance policy**: launch-before-terminate vs terminate-before-launch for health replacements, refreshes and rebalancing
- Alternative deployment patterns: blue/green with two ASGs + weighted target groups / CodeDeploy; CloudFormation `UpdatePolicy` (`AutoScalingRollingUpdate` / `AutoScalingReplacingUpdate`)

## Other ASG Features

- **Notifications** to SNS on launch/terminate/fail events; all events in EventBridge
- `LaunchInstances` synchronous API (2025): launch into a specific AZ/subnet with immediate capacity feedback
- ASG metrics in CloudWatch (group metrics, 1-minute granularity) - must be enabled
- Instance lifetime: `MaxInstanceLifetime` (min 1 day) forces periodic replacement (patching, compliance)
- Compute Optimizer gives rightsizing recommendations for ASGs
