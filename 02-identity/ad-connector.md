# Directory Service - AD Connector

- AD Connector provides a pair of directory endpoints in a VPC
- It injects ENIs into to subnets in a VPC
- Once injected AD connector appears as a native directory to other AWS instances capable of using a directory service
- Redirects requests to an existing on-premise directory server, which means no directory data is stored in AWS
- AD connector allows us to use AWS services which do require an AD directory (such as Workspaces) and use this with an on-premises directory service => we don't need to deploy additional AD directory in AWS
- There are 2 sizes of directory services _small_ and _large_, while there are no explicit user limits, the chosen size does impact the amount of compute allocated by AWS for the connector
    - Start with small; large for very large user counts (AWS guidance: 10,000+ active users of integrated apps) or deeply nested groups
    - Small can be upgraded to large, **large can not be downgraded**
- We can use multiple connector to distribute the load
- AD directory is placed in 2 subnets in a VPCs in different availability zones => resilient to AZ failure
- The connector should be configured to point to at least one on-premise directory service => we need to provide account information for the connector to be able to authenticate itself
    - Uses a single **service account** in the on-prem AD with delegated rights (read users/groups, join computers to the domain)
- Requires a PRIVATE working network to on-premise service, otherwise wont work (private network via Direct Connect or VPN) => no caching or storing of data on AWS !
    - If the network link or the on-prem DCs are down, authentication to AWS apps (WorkSpaces logins, console) **fails**
    - Required ports to on-prem DCs: DNS 53, Kerberos 88, LDAP 389 (TCP/UDP); security groups/NACLs/firewalls must allow them

## AD Connector Architecture

![AD Connector](images/DirectoryServiceADConnector.png)

## Features

- **Seamless domain join** of EC2 Windows (and Linux) instances to the on-prem domain
- **AWS Management Console** sign-in with AD credentials (mapping AD groups to IAM roles)
- **IAM Identity Center** identity source (users/groups read from on-prem AD)
- **RADIUS-based MFA** using the existing on-prem RADIUS/MFA infrastructure
- **Smart card (CAC/PIV) authentication** for WorkSpaces (mTLS / certificate-based)
- Works with WorkSpaces, WorkDocs, Amazon Quick (QuickSight), WorkMail, Chime, Amazon Connect
- Users, passwords and policies (expiration, history, lockout) stay managed in the on-prem AD with the existing tools

## Limitations (exam traps)

- No directory data in AWS: **no trusts**, no schema extensions, no Group Policy hosted in AWS
- **Not compatible with RDS for SQL Server** (and Amazon FSx for Windows needs a real AD: Managed Microsoft AD or self-managed AD, not AD Connector)
- Can not be **shared** across AWS accounts (directory sharing is a Managed Microsoft AD feature) - deploy one AD Connector per account/VPC or use Managed Microsoft AD
- Each AD Connector connects to **one** AD domain; for multiple domains use multiple connectors
- Latency-sensitive: every authentication travels to on-prem

## Use cases for AD Connector

- Prof of concept projects, we don't want to move our Active Directory to AWS for it
- We have a small infrastructure in AWS and we don't want to move the Active Directory to AWS
- Legal/compliance reasons - we don't want to store user info in AWS
- For larger requirements use Microsoft AD mode

## AD Connector vs alternatives

- **AD Connector**: proxy only, no data in AWS, depends on the network link to on-prem
- **AWS Managed Microsoft AD**: real AD in AWS, works during link failures, trusts with on-prem, supports RDS SQL Server / FSx / sharing / multi-region
- **AWS Managed Microsoft AD (Hybrid Edition)**: extends an existing self-managed AD into AWS (AWS-managed domain controllers for the existing domain), no trust and no identity sync needed
- **Self-managed AD on EC2**: full control (e.g. needs Domain/Enterprise Admin rights, custom replication) at the cost of operational overhead
- **Simple AD**: Samba-based standalone directory, no trusts/MFA
    > **Status (2026):** Simple AD is no longer open to new customers; existing customers keep full functionality. AWS recommends Managed Microsoft AD or AD Connector instead
