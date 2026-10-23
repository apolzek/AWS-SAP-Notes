# AWS Certificate Manager - ACM

- HTTPS (SSL/TSL) was designed to address security problems occurred with HTTP
- HTTPS provides data encryption in-transit and certificates to prove the identity
- ACM can function as a public certificate authority or a private certificate authority (CA) - private CA is the separate AWS Private CA service
- In case of a private CA applications need to be configured to trust the private CA
- With ACM we can generate or import certificates
- If ACM generates the certificate, it can renew it automatically. If imported, the user is responsible for renewal <span style="color: red;">EXAM</span>
- ACM can only deploy certificates to supported services (services in AWS which are integrated with ACM)
- Not all services all supported. Services which integrate with ACM are the following: load balancers (ALB, NLB, CLB), CloudFront, Cognito, Elastic Beanstalk, App Runner, API Gateway, AWS Nitro Enclaves (ACM for Nitro Enclaves on EC2), OpenSearch, AWS Network Firewall (TLS inspection), Amplify, Verified Access. EC2, for example, is not supported (unless using an exportable certificate or Nitro Enclaves) <span style="color: red;">EXAM</span>
- ACM is a regional service
- Certificates cannot leave the region they are generated or imported in, to use a certificate within an ALB in ap-southeast-2, the certificate needs to be in ACM in ap-southeast-2 (ACM is a regional service!!!) <span style="color: red;">EXAM</span>
- For global Services such as CloudFront, certificates should be stored in **us-east-1** !

## Public Certificates

- Free for public certificates used with integrated services (non-exportable)
- Validation: **DNS validation** (CNAME record, recommended - enables automatic renewal as long as the record stays in place) or email validation (manual renewal approval) <span style="color: red;">EXAM</span>
- Validity: since Feb 18, 2026 ACM issues public certificates with **198 days** validity (was 395 days) to comply with the CA/Browser Forum 200-day maximum (Mar 15, 2026). Managed renewal starts 45 days before expiry
- Imported certificates: ACM does not renew them; it emits `ACM Certificate Approaching Expiration` events to EventBridge (default 45 days before, configurable) and Config has the `acm-certificate-expiration-check` rule
- **Exportable public certificates** (June 2025): paid option where we can export the certificate + private key + chain and install it anywhere (EC2, containers, on-premises). ACM still renews it, we must re-export/redeploy (automate via EventBridge). Priced per FQDN/wildcard
- Before exportable certs existed, "EC2 needs a public cert" meant: put an ALB/NLB/CloudFront in front, or use a third-party/Let's Encrypt certificate - still a common exam answer
- Wildcard (`*.example.com`) and SAN (multiple names) supported; CAA DNS records must allow `amazon.com` (or related) if present

## AWS Private CA (formerly ACM Private CA)

- Managed private certificate authority hierarchy (root + subordinate CAs), keys stored in FIPS-validated HSMs
- Use cases: internal TLS (service mesh, internal ALBs), mTLS, IoT devices, code signing, Windows/AD (smart card), Kubernetes (cert-manager), IAM Roles Anywhere trust anchor <span style="color: red;">EXAM</span>
- Private certificates issued through ACM are **exportable** (can be installed on EC2/on-prem) and can be renewed by ACM
- Modes: general purpose (any validity) or **short-lived certificate mode** (validity <= 7 days, much cheaper, no revocation needed)
- Revocation: CRL (in S3) or OCSP
- Can be shared cross-account with AWS RAM - central CA in a security account, workloads in other accounts request certificates
- Costly: ~$400/month per general-purpose CA (+ per certificate) - prefer one shared CA hierarchy

## Exam Tips

- ALB/NLB/API Gateway regional -> certificate in the **same region**; CloudFront and API Gateway edge-optimized -> **us-east-1**
- Multi-region app -> request a certificate in each region (certificates can't be copied)
- ALB supports multiple certificates per listener with SNI; NLB supports TLS listeners with ACM certificates too
- End-to-end encryption to EC2: ALB terminates with ACM cert, re-encrypts to targets using a self-signed or exportable/private cert on the instance
