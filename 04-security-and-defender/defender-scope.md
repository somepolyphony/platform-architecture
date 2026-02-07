# Defender Scope

## What Defender is Responsible For

Microsoft Defender for Cloud is the centralised security posture management and threat protection platform.

Its scope is clearly defined. It does not do everything.

## Defender for Cloud Core Functions

### 1. Security Posture Assessment

Defender scans Azure subscriptions for security misconfigurations.

**Examples**:
- VMs without endpoint protection
- Storage accounts with public network access
- SQL databases without transparent data encryption
- Key Vaults without soft delete enabled

**Output**: Secure Score (0-100%). Recommendations for remediation.

**Responsibility**: Defender identifies issues. Platform and delivery teams remediate.

### 2. Regulatory Compliance Monitoring

Defender maps Azure resources to compliance frameworks (ISO 27001, PCI-DSS, CIS Azure Benchmarks).

**Output**: Compliance dashboard showing pass/fail for each control.

**Responsibility**: Defender reports compliance status. Security team owns compliance remediation.

### 3. Threat Detection

Defender monitors Azure Activity Logs, VM logs, network traffic, and PaaS service logs for suspicious activity.

**Examples**:
- Unusual RBAC changes (e.g., user grants themselves Owner)
- Suspicious sign-ins (e.g., sign-in from Tor exit node)
- Anomalous resource access (e.g., storage account accessed from unexpected IP)

**Output**: Security alerts sent to Sentinel (SIEM) and email.

**Responsibility**: Defender generates alerts. Security operations team investigates and responds.

### 4. Workload Protection

Defender provides runtime threat protection for VMs, containers, databases, and storage accounts.

**Examples**:
- Defender for Servers: Detects malware, rootkits, brute-force attempts
- Defender for Containers: Scans container images for vulnerabilities
- Defender for SQL: Detects SQL injection, anomalous queries
- Defender for Storage: Detects malware uploads, unusual access patterns

**Responsibility**: Defender detects threats. Workload owners and security team respond.

## Defender Plans (Per Workload Type)

Defender for Cloud has multiple "plans" that must be enabled per subscription.

### Defender for Servers

**Coverage**: Azure VMs and on-premises servers (via Azure Arc).

**Features**:
- Endpoint Detection and Response (EDR) via Microsoft Defender for Endpoint
- Vulnerability scanning without agent installation
- Just-in-time VM access (temporary SSH/RDP access)
- File Integrity Monitoring (FIM)

**Cost**: ~£10/server/month.

**Mandatory For**: All VMs in production subscriptions.

**Optional For**: VMs in sandbox subscriptions.

### Defender for Containers

**Coverage**: Azure Kubernetes Service (AKS), container registries (ACR).

**Features**:
- Container image vulnerability scanning
- Runtime threat detection in Kubernetes pods
- Kubernetes configuration assessment (CIS Kubernetes Benchmark)

**Cost**: ~£5/vCore/month (AKS nodes).

**Mandatory For**: All AKS clusters in production.

### Defender for SQL

**Coverage**: Azure SQL Database, SQL Managed Instance, SQL on VMs.

**Features**:
- SQL injection detection
- Anomalous query patterns (data exfiltration detection)
- Vulnerability assessment (missing patches, weak passwords)

**Cost**: ~£12/SQL server/month.

**Mandatory For**: All SQL databases in production.

### Defender for Storage

**Coverage**: Azure Storage accounts (Blob, Files, Queues).

**Features**:
- Malware scanning on blob uploads
- Anomalous access detection (unexpected IPs, mass downloads)
- Data exfiltration detection

**Cost**: ~£8/storage account/month (varies by activity).

**Mandatory For**: Storage accounts containing sensitive data.

### Defender for Key Vault

**Coverage**: Azure Key Vault.

**Features**:
- Anomalous secret access (unusual frequency, unknown IPs)
- Key Vault configuration assessment

**Cost**: Included in Defender for Cloud (no additional charge).

**Mandatory For**: All Key Vaults.

### Defender for App Service

**Coverage**: Azure App Service and Function Apps.

**Features**:
- Runtime threat detection (malicious HTTP requests, anomalous behaviour)
- App Service configuration assessment

**Cost**: ~£12/App Service plan/month.

**Mandatory For**: Production App Services.

### Defender for Open-Source Relational Databases

**Coverage**: Azure Database for PostgreSQL, MySQL.

**Features**:
- Anomalous query detection
- Brute-force attack detection

**Cost**: ~£12/database server/month.

**Mandatory For**: Production databases.

## What Defender is NOT Responsible For

### 1. Application-Level Security

Defender does not inspect application code for vulnerabilities.

**Examples of What Defender Does NOT Do**:
- Static application security testing (SAST)
- Dynamic application security testing (DAST)
- Dependency vulnerability scanning in application code

**Responsibility**: Delivery teams use tools like Snyk, Dependabot, or SonarQube in CI/CD pipelines.

### 2. Identity and Access Management

Defender flags risky RBAC assignments but does not enforce identity controls.

**Examples of What Defender Does NOT Do**:
- Enforce Conditional Access policies (Entra ID's responsibility)
- Enforce PIM activation (Entra ID's responsibility)
- Block overly permissive RBAC assignments (Azure Policy's responsibility)

**Responsibility**: Identity controls are enforced via Entra ID and Azure Policy.

### 3. Network Security Enforcement

Defender flags misconfigured NSGs and public IPs but does not block traffic.

**Examples of What Defender Does NOT Do**:
- Block inbound RDP from internet (NSG or Azure Firewall does this)
- Enforce private endpoints (Azure Policy does this)

**Responsibility**: Networking controls are enforced via NSGs, Azure Firewall, and Azure Policy.

### 4. Incident Response

Defender generates alerts. It does not respond to incidents.

**Examples of What Defender Does NOT Do**:
- Isolate compromised VMs
- Revoke compromised credentials
- Restore data from backups

**Responsibility**: Security operations team (SOC) responds to Defender alerts.

### 5. Compliance Audits

Defender provides compliance dashboards but does not conduct audits or generate audit reports.

**Examples of What Defender Does NOT Do**:
- Produce ISO 27001 audit reports
- Provide evidence for SOC 2 audits

**Responsibility**: Security team collects Defender data and produces audit reports.

## Defender Integration with Sentinel

Defender alerts are forwarded to Microsoft Sentinel (Azure-native SIEM).

**Why**: Sentinel provides advanced correlation, automation, and incident management beyond Defender's capabilities.

**Example Workflow**:
1. Defender detects suspicious VM access
2. Alert sent to Sentinel
3. Sentinel correlates with Entra ID sign-in logs
4. Sentinel creates incident and assigns to SOC analyst
5. SOC analyst investigates and takes action

**Responsibility Split**:
- Defender: Detection
- Sentinel: Correlation and workflow
- SOC: Investigation and response

## Defender Configuration Ownership

### Central Configuration

The following Defender settings are managed centrally by the security team:

- Defender plans enabled per subscription
- Security contacts (email recipients for alerts)
- Workflow automation (alert forwarding to Sentinel, ServiceNow)
- Secure Score goals

**Why Central**: Prevents delivery teams from disabling Defender to reduce noise or cost.

### Delegated Configuration

The following Defender settings can be managed by delivery teams:

- Just-in-time VM access requests
- Suppression of false-positive alerts (with security team approval)
- Custom assessment policies (organisation-specific security checks)

**Why Delegated**: Workload owners know their applications best and can reduce false positives.

## Defender Limitations and Gaps

### 1. No Zero-Day Detection Guarantee

Defender uses signatures and behavioural analysis. It cannot detect **all** threats.

**Examples of Undetected Threats**:
- Novel malware with no signature
- Low-and-slow attacks designed to evade behavioural detection
- Insider threats using legitimate credentials

**Mitigation**: Assume Defender will miss threats. Defence-in-depth (network segmentation, least privilege) limits blast radius.

### 2. Alert Fatigue

Defender generates high volumes of low-severity alerts.

**Examples**:
- "VM does not have endpoint protection" (VM is being decommissioned)
- "Storage account allows public network access" (required for external integration)

**Impact**: SOC analysts ignore alerts. High-severity threats missed.

**Mitigation**: Tune Defender recommendations. Suppress false positives. Prioritise high-severity alerts.

### 3. Coverage Gaps in Multi-Cloud

Defender for Cloud supports AWS and GCP via Azure Arc. Coverage is not equivalent to native Azure.

**Examples of Gaps**:
- AWS Lambda functions not fully supported
- GCP Cloud Functions not supported

**Mitigation**: Use AWS-native or GCP-native security tools in addition to Defender.

### 4. Defender Does Not Prevent, Only Detects

Defender is a detective control, not a preventative control.

**Example**: Defender detects a compromised VM. It does **not** automatically isolate the VM or revoke credentials.

**Mitigation**: Integrate Defender with Security Orchestration, Automation, and Response (SOAR) tools for automated response.

## Defender Costs

Defender for Cloud is **not** free. Costs scale with resource count and activity.

**Approximate Costs (2026)**:
- Defender for Servers: £10/VM/month
- Defender for Containers: £5/vCore/month
- Defender for SQL: £12/SQL server/month
- Defender for Storage: £8/storage account/month (varies by activity)
- Defender for App Service: £12/App Service plan/month

**Total Cost for Typical Production Subscription**:
- 10 VMs: £100/month
- 2 SQL databases: £24/month
- 5 storage accounts: £40/month
- 1 App Service plan: £12/month
- **Total: ~£176/month per subscription**

**Cost Trade-Off**: Defender costs are non-negotiable for production. Detection and compliance monitoring justify the cost.

**Cost Savings**: Disable Defender in sandbox subscriptions. Risk is acceptable.

## Defender Scope Summary

| Function                     | Defender Provides | Defender Does NOT Provide |
|------------------------------|-------------------|---------------------------|
| Security posture assessment  | ✅                | ❌ Automatic remediation  |
| Threat detection             | ✅                | ❌ Incident response      |
| Compliance monitoring        | ✅                | ❌ Audit reports          |
| Workload protection          | ✅                | ❌ Application security   |
| Alert generation             | ✅                | ❌ Alert investigation    |

Defender is a foundational security tool. It is not sufficient alone. Defence-in-depth requires multiple layers.