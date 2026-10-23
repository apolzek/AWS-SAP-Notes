# Amazon Workspaces

- Managed desktop as a service product (DAAS) - dedicated virtual Windows/Linux desktop delivered as a managed service
- Ideal for home working
- Similar to Citrix/Remote Desktop - hosted within AWS
- It provides a consistent desktop accessible from anywhere, we can log off and log back in from different places, applications maintaining their state
- We can have Windows and Linux workspaces in various sizes
    - Windows (Windows Server-based desktops, Windows 10/11 with BYOL), Ubuntu, Rocky Linux, Red Hat Enterprise Linux; GPU bundles for graphics workloads
- We can use commercial applications on the workspaces
- Workspaces can be charged on monthly or hourly basis. There is an additional monthly infrastructure cost, which is applied even in case when we are billed hourly
    - Running modes: **AlwaysOn** (monthly) or **AutoStop** (hourly, stops after inactivity - cheaper for part-time users)
- Workspaces use Directory Service (Simple, AD, AD Connector) for authentication and user management
    - Also **Microsoft Entra ID** (dedicated directory, Windows 10/11 BYOL, Intune enrollment)
    > **Status (2026):** Simple AD is no longer open to new customers - new deployments use Managed Microsoft AD or AD Connector
- Each workspace uses an ENI (Elastic Network Interface) injected in a VPC
- Workspaces are accessed using client software from desktop/laptop, bandwidth is included free of charge. For any other internet access from the workspaces normal VPC infrastructure is used and charged accordingly
    - Clients: Windows, macOS, Linux, iPad/Android, **web browser** access, third-party thin clients
- Windows workspaces can access FSx and EC2 windows resources
- Any existing hybrid network can be also utilized for accessing on-premise resources
- Workspaces provide a system volume and an user volume, both can be encrypted
    - Encryption with KMS keys (must be chosen at launch); root volume `C:` / user volume `D:`
    - Automatic snapshots of volumes; **Restore** (to last snapshot) vs **Rebuild** (recreate from the bundle image, keep user volume)

## Workspaces Architecture

![Workspaces Architecture](images/AmazonWorkspaces.png)

- Workspaces inject an ENI into customer managed VPCs, they run into AWS managed VPCs
- Directory services also inject ENIs into customer managed VPCs for file access, etc.
- Workspaces are not highly available by design
- We can distribute workspaces in different AZs, but a single workspace in particular can be affected by an AZ failure

## WorkSpaces family

- **WorkSpaces Personal**: persistent desktop assigned to one user (classic WorkSpaces)
- **WorkSpaces Pools**: non-persistent desktops from a shared pool, ephemeral infrastructure, user settings persisted via application settings persistence/home folders; authentication via **SAML 2.0 IdP**, AD domain join optional - good for task workers, students, contact centers
- **WorkSpaces Core**: managed VDI infrastructure APIs for third-party VDI vendors (Citrix, Omnissa/VMware, Leostream...)
- **WorkSpaces Applications** (formerly **AppStream 2.0**, renamed Nov 2025): stream individual applications (not full desktops) to a browser
- **WorkSpaces Secure Browser** (formerly **WorkSpaces Web**): managed, isolated browser to access internal web apps/SaaS from unmanaged devices, no data left on the device
- **WorkSpaces Thin Client**: low-cost AWS hardware device for WorkSpaces/Secure Browser/Applications
    > **Status (2026):** WorkSpaces Thin Client is closed to new customers (spring 2026) and reaches end of support on March 31, 2027. Use third-party thin clients instead

## Networking and security

- **Protocols**: **DCV** (formerly WSP - WorkSpaces Streaming Protocol, AWS-built, supports web access, smart cards, webcams, better on high latency) and **PCoIP**
- Client to WorkSpace streaming goes over the internet to AWS-managed gateways (TCP/UDP 4172 for PCoIP, 4195 for DCV), or privately through Direct Connect public VIF / VPN; **interface VPC endpoints** for the streaming gateway are supported for DCV
- Two ENIs per WorkSpace: management ENI (AWS managed) and primary ENI in our VPC (internet access via NAT GW or Elastic IP, access to on-prem via TGW/VPN/DX)
- **IP access control groups**: restrict which public IP ranges can connect (e.g. corporate offices only)
- **Trusted devices**: certificate-based restriction so only corporate devices can connect
- **MFA** via RADIUS (Managed Microsoft AD, AD Connector), **smart card (CAC/PIV)** with DCV, **certificate-based authentication** with SAML for SSO
- **SAML 2.0 integration**: users authenticate with their IdP (Okta, Entra ID, ...) before the directory login
- Patching/AV management is the customer's responsibility (WSUS, SSM, Intune, etc.)

## Resilience and DR

- **Multi-Region Resilience**: **standby WorkSpaces** in a secondary region + **cross-region redirection** (connection aliases / FQDN with Route 53 failover) - users reconnect to the standby desktop if the primary region fails
- Standby WorkSpaces have data replication option (periodic snapshot copy of user volume) - RPO in hours, not zero
- Directory must be available in the DR region (Managed Microsoft AD multi-region replication or a second directory/AD Connector)
- Exam pattern: "remote employees need desktops quickly, data must not leave AWS, access only from corporate IPs" -> WorkSpaces + IP access control groups; "contractors on unmanaged devices only need internal web apps" -> WorkSpaces Secure Browser; "stream one legacy app to many users" -> WorkSpaces Applications
