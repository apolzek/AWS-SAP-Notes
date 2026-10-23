# Directory Service - Microsoft AD

- Built using real Microsoft Active Directory, powered by **Windows Server 2019** (originally launched on 2012 R2; AWS upgraded the domain controllers and functional level)
- Can be managed using the standard active directory tools
- Supports Group Policy and SSO
- Supports Schema extension - MS AD Aware Apps => such as Sharepoint, SQL, DFS (Distributed File System)
- Two sizes: Standard (~30,000 objects, up to ~5,000 users) VS Enterprise (~500,000 objects)
- Native AD, used for AD authentication/authorisation of products and services within AWS
- HA by default => >=2AZs
    - Deploys 2 domain controllers in 2 subnets/AZs; we can **add more domain controllers** to scale reads/authentications
- Includes monitoring, recovery, replication, snapshots, and maintenance => configurable but managed by AWS
    - We do NOT get Domain Admin / Enterprise Admin; we get a delegated *AWS Delegated Administrators* OU/group

- Supports 1-way or 2-way external and forest trusts with on-prem active directory (but are SEPARATE directories, they can only be configured to trust each other !)
- Directory in AWS => can operate through a network link failure to any connected on-prem systems (not the case wih AD Connector !)
- Supports RADIUS-based MFA
- Best choice for >5'000 users and if trust relationships between AWS and on-prem directories are needed

![alt text](images/microsoft-ad.png)

## Editions

- **Standard Edition**: primary directory for SMBs, up to ~5,000 employees / ~30,000 directory objects
- **Enterprise Edition**: up to ~500,000 objects; required for **multi-region replication**
- **Hybrid Edition** (newer): extends an existing self-managed AD into AWS with AWS-managed domain controllers, without a trust relationship and without identity sync; AWS runs periodic directory assessments
- Edition upgrade Standard -> Enterprise is possible; downgrade is not

## Key features

- **Multi-region replication** (Enterprise only): replicate the directory to additional regions, AWS manages DCs and replication; apps in each region use local DCs (low latency, regional DR)
- **Directory sharing**: share one directory with other AWS accounts (same org or via handshake) for **seamless EC2 domain join** and RDS/FSx integration - avoids one directory per account (number of share targets depends on edition)
- **Trusts**: external or forest trusts, one-way or two-way, with on-prem or self-managed AD (requires network connectivity + DNS conditional forwarders)
    - One-way trust (AWS trusts on-prem): on-prem users can access AWS resources, identities stay on-prem - typical *resource forest* pattern
- **Integrations**: WorkSpaces, Amazon Quick (QuickSight), WorkMail, Chime, Amazon Connect, **RDS for SQL Server / Oracle / PostgreSQL (Kerberos)**, **FSx for Windows File Server**, FSx for NetApp ONTAP (SMB), IAM Identity Center, console sign-in, Microsoft Entra Connect (Office 365)
- **Secure LDAP** (LDAPS, client and server side), **client certificate (smart card) auth** for WorkSpaces, Kerberos SSO
- **Password policies** (fine-grained), AD recycle bin, gMSA, schema extensions
- **Log forwarding** of security events to CloudWatch Logs; **snapshots** (daily automatic + manual) for restore
- **Compliance**: can be enabled for PCI DSS / HIPAA (disables older protocols)
- AWS Private CA Connector for AD: issue certificates to domain-joined machines/users without running AD CS

## Microsoft AD vs AD Connector vs Simple AD

- **Microsoft AD**: real AD in AWS, trusts, sharing, multi-region, RDS SQL Server/FSx, works if on-prem link fails
- **AD Connector**: proxy to on-prem AD, no data in AWS, fails if the link fails, no RDS SQL Server, no sharing
- **Simple AD**: Samba 4, small (500 users) / large (5,000 users), no trusts, no MFA, no RDS SQL Server, no IAM Identity Center
    > **Status (2026):** Simple AD is no longer open to new customers; existing customers are not affected
- Exam rules of thumb:
    - "Users must stay on-prem, minimal cost, no data in AWS" -> AD Connector
    - "AD-aware apps in AWS (SharePoint, SQL Server, FSx), must survive link failure, trust with on-prem" -> Managed Microsoft AD (+ trust)
    - "Many accounts must domain-join to one directory" -> Managed Microsoft AD + directory sharing
    - "Global users, low-latency auth in several regions" -> Managed Microsoft AD Enterprise + multi-region replication
    - "Need Domain Admin rights / unsupported AD configuration" -> self-managed AD on EC2
