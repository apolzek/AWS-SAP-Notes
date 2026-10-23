# AWS Transfer Family

- Fully managed file transfer service TO or FROM **S3** and **EFS** (not FSx, not EBS)
- Clients keep using their existing tools (WinSCP, FileZilla, OpenSSH, partner systems) => no client changes when migrating legacy FTP/SFTP servers
- Supported protocols:
  - FTP (File Transfer Protocol) - unencrypted
  - FTPS (FTP Secure) - file transfer with TLS encryption
  - SFTP (SSH FTP) - file transfer over SSH
  - AS2 (Applicability Statement 2) - structured B2B data
- Transfer Family supports a wide range of identities: service managed (SSH keys, SFTP only), AWS Directory Service (Microsoft AD, incl. on-prem AD via AD Connector/trust), custom IdP (Lambda or API Gateway => Okta, LDAP, Secrets Manager, etc.)
- Each user maps to an IAM role (access to bucket/file system) + optional session policy; **logical directories** hide real bucket/paths and chroot users to a home directory
- Managed File Transfer Workflows (MFTW): serverless file workflow engine:
  - Can be used when file are uploaded, we can define workflows as to what happens to the file as it gets uploaded
  - Predefined steps: copy, tag, delete, **PGP decrypt**; custom steps via Lambda (e.g. virus scan, validation, ETL)
  - Exception handling steps if a workflow step fails
  - Alternative: S3 Event Notifications / EventBridge => Step Functions for more complex processing

![Transfer Family](images/TransferFamily.png)

## Transfer Family Endpoint Types

- Within Transfer Family we create servers which we can think of as the front-end accesspoint to our storage
- They present S3 and EFS via one or more supported protocol
- How we access these servers depends on how configure the service's endpoints:
  - Public: runs on the AWS public zone, accessible to the public internet
    - No networking components to configure
    - Only supported protocol is SFTP
    - The endpoint has a dynamic IP which can change, we should use DNS to access it
    - We can't control who will access it using features such as NACLs or security groups
  - VPC - Internet Access
    - Runs in a VPC
    - We can use SFTP/FTPS and AS2 protocols
    - Anything that has connectivity to the VPC (DX/VPN) can access it as it was running inside the VPC
    - Transfer Family provides a static IP for it
    - SG/NACLs are supported
    - It is allocated an Elastic IP for it which is static, which allows it to be accessed over the public internet
    - Partners can allow-list the EIPs in their firewalls; we can bring our own IPs (BYOIP)
  - VPC - Internal
    - Runs inside a VPC
    - We can use SFTP/FTPS/FTP and AS2 protocols
    - Anything that has connectivity to the VPC (DX/VPN) can access it as it was running inside the VPC
    - Transfer Family provides a static IP for it
    - SG/NACLs are supported

![Transfer Family Endpoints](images/EndpointsTransfer.png)

## Other Features

- It is multi-AZ => resilient and scalable
- Cost is based for provisioned server per hour + data transfer
- With FTP/FTPS only Directory Service and Custom IDP is supported
- FTP can only be used internally within a VPC
- AS2 has to be VPC Internet/Internal only, we cannot use the public endpoint type
- Public endpoint: can't be locked down by SG and has changing IPs => for IP allow-listing use VPC Internet-facing endpoint with EIPs
- Legacy `VPC_ENDPOINT` endpoint type is deprecated => use `VPC` endpoint type
- Security policies select TLS/SSH ciphers (including FIPS and post-quantum hybrid key exchange for SFTP)
- Logging to CloudWatch Logs (structured JSON), dual-stack IPv6 support
- Use cases for Transfer Family:
  - In case we need access to S3/EFS but using the supported protocols
  - Integration with existing workflows
  - Using MFTW to create new workflows
  - B2B data exchange with partners (EDI over AS2, payments, healthcare, supply chain)
  - Replace self-managed SFTP servers on EC2 (no patching, HA by default)

## Connectors (outbound)

- Transfer Family can also act as a **client** to external servers:
  - **SFTP connectors**: send/retrieve files between S3 and remote (partner) SFTP servers, can list/delete/rename/move remote files; credentials in Secrets Manager
    - Egress via service-managed static IPs or through our VPC (VPC Lattice resource gateway) to reach private/on-prem servers with our EIPs/BYOIP
  - **AS2 connectors**: send AS2 messages (with signing, encryption, MDN receipts) to partners
- Triggered via `StartFileTransfer` API, typically orchestrated with EventBridge Scheduler / Step Functions
- Exam: "periodically pull files from a partner's SFTP server into S3 without managing servers" => SFTP connector

## Transfer Family Web Apps

- Fully managed, branded **browser-based** interface for end users to upload/download/browse S3 data (no SFTP client needed)
- Authentication via **IAM Identity Center**, authorization via **S3 Access Grants**
- Can be hosted with a VPC endpoint for private access; requires CORS on the bucket
- Exam: "non-technical users need a simple web portal to exchange files in S3" => Transfer Family web apps

## Transfer Family vs Other Options

- Transfer Family: protocol-based (SFTP/FTPS/FTP/AS2) access for users/partners to S3/EFS
- DataSync: bulk/online migration and scheduled sync between on-prem (NFS/SMB/HDFS/object) or other clouds and AWS storage
- Storage Gateway (File Gateway): on-prem NFS/SMB mount backed by S3 with local cache
- Offline transfer when the network is too slow: Snowball Edge (> **Status (2026):** closed to new customers since Nov 2025; new customers => AWS Data Transfer Terminal or partner solutions)
- Presigned URLs / Transfer Acceleration: app-driven uploads over HTTPS
