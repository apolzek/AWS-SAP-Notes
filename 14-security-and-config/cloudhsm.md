# CloudHSM

- Similar to KSM, it creates, manages and secures cryptographic material or keys
- KMS is a shared service. AWS has a certain level of access to the product, they manage the hardware and the software of the system
- KMS uses behind the scene HSM devices
- CloudHSM is true single tenant HSM(Hardware Security Module) hosted by AWS <span style="background-color: Red"><-- REMEMBER THIS FOR EXAM</span>
- AWS provisions the hardware for CloudHSM but they do not have access to it. In case of losing access to a HSM device there is no easy way to re-gain the access to it
- CloudHSM is fully compliant with FIPS 140-2 Level 3 (KMS is L2 compliant overall) <span style="background-color: Red"><-- REMEMBER THIS FOR EXAM</span>
    - **Update (2026):** current HSM type `hsm2m.medium` is **FIPS 140-3 Level 3** validated; the old `hsm1.medium` (FIPS 140-2 L3) reached end of support Dec 1, 2025 (clusters auto-migrated to hsm2m). KMS HSMs are now also validated at **FIPS 140-3 Level 3** overall, so "KMS = Level 2" is outdated. The exam differentiator is now **single-tenant, customer-exclusive control of keys** (AWS has no access), not the FIPS level
- CloudHSM is accessed with industry standards APIs: PKCS#11, Java Cryptography Extensions (JCE), Microsoft CryptoNG (CNG) libraries. It is not that integrated with other AWS services by design (in comparison, KMS integrates with basically every service) <span style="background-color: Red"><-- REMEMBER THIS FOR EXAM</span>
- KMS can use CloudHSM as a **custom key store**, CloudHSM integration with KMS
    - KMS keys are created in KMS but key material lives in our CloudHSM cluster - AWS services that integrate with KMS (S3 SSE-KMS, EBS, RDS) can then use HSM-backed keys we exclusively control <span style="color: red;">EXAM</span>
    - Cluster needs at least 2 active HSMs in different AZs; if the cluster is unavailable, KMS operations with those keys fail
    - Alternative: **External key store (XKS)** - keys stay in an HSM outside AWS (on-prem/other provider) via an XKS proxy

## CloudHSM Architecture

- CloudHSM devices are deployed into a VPC managed by AWS, on which we don't have visibility
- They are injected into customer managed VPCs using ENIs (Elastic Network Interfaces)
- For HA we need to deploy multiple HSM devices and configure them as a cluster (at least 2 HSMs in different AZs; keys are synchronized automatically between HSMs in the cluster; client load-balances across them)
- Cluster backups are taken automatically (at least every 24h) to S3 in AWS's account, encrypted - can be copied to another region to create a cluster there (cross-region DR)
- Quotas: up to 28 HSMs per cluster (default limits lower); pricing per HSM-hour (no upfront)
- A client needs to be installed on the EC2 instances in order to be able to access HSM modules (CloudHSM Client SDK 5: PKCS#11, OpenSSL Dynamic Engine, JCE, KSP/CNG)
- Users: Crypto Officer (CO - manages users), Crypto User (CU - creates/uses keys), Appliance User (AWS - can sync/clone only, can't use keys)
- Separate IAM (control plane: create cluster/HSM) from HSM users (data plane: keys) - IAM can't access keys
- While AWS do provision the HSM devices, we as customers are responsible for the management of the customer keys
- AWS can provide software updates on the HSM devices, but these should not affect the encryption storage part

## CloudHSM Use Cases/Limitations

- There is no native integration with AWS services (except KMS) , this means CloudHSM can not be used for S3 SSE
- CloudHSM can be used for client-side encryption before uploading data to S3
- CloudHSM can be used to offload SSL/TLS processing for web servers. It's economical and efficient to use cloud HSM.
- Oracle Databases from RDS can perform Transparent Data Encryption (TDE) using CloudHSM
- CloudHSM can be used to protect private keys for an Issuing Certificate Authority (CA)
- Exam: "SSL offload with private keys in an HSM" (NGINX/Apache/IIS), "Oracle/SQL Server TDE", "Windows Server CA private key", "keys must be in a single-tenant HSM under exclusive customer control", "PKCS#11/JCE/CNG APIs" -> CloudHSM <span style="color: red;">EXAM</span>
- KMS vs CloudHSM: KMS = multi-tenant, fully managed, native integration with ~all services, IAM + key policies; CloudHSM = single-tenant, we manage users/keys/HA, standard crypto APIs, no native S3 SSE (except via KMS custom key store)
