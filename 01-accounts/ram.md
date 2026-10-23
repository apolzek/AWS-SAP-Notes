# AWS Resource Access Manager - RAM

- Allows sharing resources between AWS accounts
- Some services may allow sharing between any AWS accounts, some allow sharing only between accounts from the same organization (e.g., subnets)
- Services needs to support RAM in order to be shared (not everything can be shared)
- Services can be shared with principals: accounts, OU's and ORG
    - Some resource types can also be shared with **IAM roles/users** or **AWS service principals**
- Shared resources can be accessed natively
- There is no cost by using RAM, only the service cost may apply
- RAM is regional: a resource share lives in one region and shares resources of that region (global resources such as some Route 53 / Network Firewall items are shared from the home region)
- AWS RAM for sharing resources in an organization can be enabled with `enable-sharing-with-aws-organizations` CLI command. This operation creates a service-linked role called `AWSServiceRoleForResourceAccessManager` that has the IAM managed policy named `AWSResourceAccessManagerServiceRolePolicy` attached. This role permits RAM to retrieve information about the organization and its structure. This lets us share resources with all of the accounts
- Sharing with an OU/org is dynamic: accounts added to the OU/org later get access automatically; accounts leaving lose it
- RAM vs resource-based policies: use RAM when the service supports it (central view of shares, org/OU targets, invitations); some services (e.g. Private CA, Glue) let RAM *promote* a resource-policy share into a RAM share

## Commonly shared resources (exam)

- **Networking**: VPC subnets, Transit Gateways, prefix lists, Route 53 Resolver rules / query log configs / DNS Firewall rule groups, IPAM pools, VPC Lattice service networks and services, Network Firewall policies, Cloud WAN core networks, Verified Access groups
- **Compute**: EC2 Capacity Reservations, Dedicated Hosts, placement groups, Outposts, EC2 Image Builder components/images
- **Security / Governance**: AWS Private CA, License Manager configurations, Systems Manager documents/parameters (advanced), AWS Glue Data Catalog / Lake Formation resources, Resource Groups
- **Databases**: Aurora DB clusters (for cross-account **cloning**)
- Exam patterns:
    - Centralized egress/inspection: share the **Transit Gateway** with the org instead of peering VPCs
    - Hybrid DNS: create Route 53 Resolver forwarding rules in the network account and share them with all accounts
    - Centralized IP management: share **IPAM pools** to OUs

## Managed permissions

- Each resource share has a **managed permission** defining what principals can do with the shared resource
- **AWS managed permissions**: default per resource type (some types have several, e.g. read-only vs full)
- **Customer managed permissions** (2023): author our own least-privilege permission for a resource type; versioned (new version becomes default for new shares, existing shares keep their version until updated with `ReplacePermissionAssociations`); region-scoped; single statement only

## Availability Zone IDs

- A region in AWS has multiple availability zones, example: `us-east-1a`, `us-east-1b`, etc.
- AWS rotates the name of the AZs depending on the AWS account, meaning that `us-east-1a` may not be the same AZ if we compare 2 accounts
- If a failure happens on the hardware level, two accounts may see the issue being in different AZ, this may introduce a challenge in troubleshooting
- AWS provides AZ IDs to overcome this challenge. Example of IDs: `use1-az1`, `use1-az2`
- AZ IDs are consistent across multiple accounts
- Shared subnets are identified by AZ ID; use AZ IDs when coordinating multi-account deployments, Capacity Reservations or latency-sensitive placement

## RAM Concepts

- **Owner account**:
  - Owns the resource, creates a share, provides the name
  - Retains full permission over the resource shared
  - Defines the principal (AWS account, OU, entire AWS organization) with whom they share a specific resource
- **Principle**:
  - It can be an AWS account, OU, entire AWS organization
  - Resources are shared with a principle
- If the participant is inside an ORG that has sharing enabled, sharing is accepted automatically
- For non ORG accounts, or sharing with AWS Organizations is not enabled, we have to accept an invite
- Admins can block external sharing org-wide with an SCP denying `ram:CreateResourceShare` / `ram:AssociateResourceShare` when `ram:RequestedAllowsExternalPrincipals` is `true`

## Shared Services VPC

- It is a VPC which provides infrastructure which can be used by other services
- In AWS this has been traditionally architected using separate networks connected using VPC peering or Transit Gateways. With AWS RAM and AWS Organizations we can create something which is more effective:
  ![Shared Services VPC](images/RAM.png)
- VPC owner can create and manage the VPC and subnets which are shared with participants of the same AWS Organization
- Participants can provision services into the shared subnets, can read an reference network objects but can not modify or delete the subnets
- Resources created by a participant account will not be visible for other participants or by the VPC owner account
- Resources created by a participant account can be accessed from other resources created by other participant accounts because they are on the same network
- VPC owners cannot delete or modify resources created by participant VPCs
- More details on VPC sharing:
    - Subnets can only be shared **within the same organization**; the **default VPC** subnets can not be shared
    - The owner manages VPC-level objects: route tables, NACLs, IGW, NAT gateways, VPC endpoints, Transit Gateway attachments, Flow Logs (at VPC level)
    - Participants manage their own resources and **security groups**; they can reference security groups of other participants in the same VPC by ID; the owner can also share its security groups with participants
    - Billing: participants pay for their resources (EC2, RDS, ...) and data transfer; owner pays for NAT GW hourly, VPC endpoints, TGW attachments, etc.
    - Benefits: fewer VPCs (less IP waste, fewer TGW attachments/peerings), central network control by a network team, workload isolation still at account level (billing, IAM, quotas)
    - Account-level isolation vs network-level isolation: security groups + NACLs are the only network boundary between participants
