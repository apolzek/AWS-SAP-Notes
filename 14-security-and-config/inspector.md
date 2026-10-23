# AWS Inspector

> **Status (2026):** The content below the "Inspector Classic" header describes **Amazon Inspector Classic**, which reached **end of support on May 20, 2026** (closed to new customers since May 20, 2025). The current service ("Amazon Inspector", sometimes called Inspector v2, launched Nov 2021) is a completely re-architected product - see the first section. On the exam assume the new Inspector unless "assessment templates/rules packages" are mentioned.

## Amazon Inspector (current)

- Automated, continuous vulnerability management - no assessment runs to schedule; rescans automatically when a new CVE is published or the resource changes
- Enabled per account/region in one click; multi-account via AWS Organizations **delegated administrator** (auto-enable for new members)
- Resource coverage:
    - **EC2**: package vulnerabilities (CVE) + network reachability findings. Scan modes: **agent-based** (uses the SSM Agent - no separate Inspector agent) or **agentless** (EBS snapshots) - default *hybrid* mode uses SSM where available and falls back to agentless <span style="color: red;">EXAM</span>
    - **ECR container images**: on push and continuously (configurable re-scan duration, e.g. lifetime/180/30 days); also maps images to running ECS tasks/EKS pods. Replaces ECR "enhanced scanning" engine (ECR basic scanning is the free alternative)
    - **Lambda functions**: standard scanning (package dependencies) and **Lambda code scanning** (custom code - injection flaws, hard-coded secrets, weak crypto)
    - **Code security** (GA June 2025): SAST, SCA (dependencies) and IaC scanning of GitHub / GitLab repositories (on push/PR, CI/CD, scheduled)
    - **CIS benchmark** assessments for EC2 OS (on demand / scheduled)
    - CI/CD integration: scan container images in Jenkins/TeamCity/CodeBuild via the Inspector SBOM generator before pushing
- **Inspector risk score**: CVSS adjusted with network reachability and exploitability (e.g. known exploited) - prioritize
- **SBOM export** (CycloneDX / SPDX) to S3
- Findings: Inspector console, EventBridge, Security Hub (CSPM and new Security Hub), suppression rules
- Pricing: per EC2 instance per month, per image scanned (initial + rescans), per Lambda function per month
- Exam patterns:
    - "Continuously scan EC2 and container images for CVEs across the org" -> Inspector with delegated admin
    - "Instances must not be internet reachable on port 22" -> Inspector network reachability findings (or Config rule)
    - Finding -> EventBridge -> SSM Patch Manager (`AWS-RunPatchBaseline`) for auto-remediation
    - Inspector = vulnerabilities in software; GuardDuty = threats/malicious activity; Macie = sensitive data in S3; Config = resource configuration compliance

## Inspector Classic (legacy - end of support May 20, 2026)

- Is a product designed to check EC2 instances and the operating systems running on those instances, as well as container workflows, for any vulnerabilities or deviations against best practice
- Inspector can be run for a certain period of time (15 min, 1 hour, 1 day, etc.) to identify any unusual traffic and configurations which can put instances to risk
- At the end of this process, Inspector provides a report of findings ordered by severity
- Inspector can work with 2 main type of assessments:
    - Network Assessment: can be conducted agentless, but adding an agent can provide a more richer assessment
    - Network and Host Assessment: requires an agent to be installed. The host assessment looks for OS level vulnerabilities, so it requires the presence of an agent
- Rules packages: determine what is checked on an instance
- Examples of rule packages:
    - **Network Reachability**: 
        - Can be done with no agent or with an agent providing OS visibility
        - Checks reachability end to end
        - Returns the following findings:
            - `RecognizedPortWithListener`
            - `RecognizedPortNoListener`
            - `RecognizedPortNoAgent`
            - `UnrecognizedPortWithListener`
    - **Host Assessment**:
        - Agent is required
        - Checks for Common vulnerabilities and exposures (CVE)
        - Center for Internet Security (CIS) Benchmarks
        - Security best practices for Amazon Inspector
