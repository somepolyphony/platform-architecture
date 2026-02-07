# Threat Model

## Assumed Threat Actors

Platform security must defend against the following threat actors in order of likelihood:

### 1. Opportunistic External Attackers

**Capability**: Automated scanning, credential stuffing, exploitation of known CVEs.

**Motivation**: Financial gain via ransomware, cryptomining, or data exfiltration.

**Assumed Access**: None initially. Targets publicly exposed endpoints, leaked credentials, or unpatched services.

**Mitigations**:
- Multi-factor authentication for all identities
- Private endpoints for PaaS services
- Azure Firewall with threat intelligence
- Defender for Cloud with automatic remediation
- Regular patching via Windows Update for Business

### 2. Malicious Insiders

**Capability**: Legitimate access to subscriptions, workloads, or endpoints. Understands internal architecture.

**Motivation**: Financial gain, revenge, espionage.

**Assumed Access**: Standard user account with limited Azure RBAC. May have elevated access to specific workloads.

**Mitigations**:
- Principle of least privilege enforced via Azure RBAC and PIM
- Immutable audit logging to external SIEM
- Anomaly detection via Defender for Cloud Apps
- Separation of duties between platform and workload teams
- Break-glass accounts monitored with immediate alerting

### 3. Compromised Service Accounts

**Capability**: Automated access to APIs, subscriptions, or PaaS services. Often has broad permissions due to historical over-provisioning.

**Motivation**: Lateral movement, persistence, privilege escalation.

**Assumed Access**: Service principal or managed identity with Contributor or Owner on one or more subscriptions.

**Mitigations**:
- Managed identities preferred over service principals
- Service principals scoped to specific resource groups, not subscriptions
- Credential rotation enforced every 90 days
- Azure Policy blocks creation of long-lived secrets
- Conditional Access enforced for service principal authentication where possible

### 4. Supply Chain Compromise

**Capability**: Malicious code in third-party dependencies, container images, or SaaS integrations.

**Motivation**: Persistent access, data exfiltration, downstream customer compromise.

**Assumed Access**: Application-level access via trusted code execution path.

**Mitigations**:
- Container image scanning via Defender for Containers
- Dependency vulnerability scanning in CI/CD pipelines
- Network egress restrictions to prevent data exfiltration
- Microsoft 365 app consent policies restrict third-party integrations
- Zero trust assumption: even trusted apps are sandboxed

### 5. Nation-State Actors

**Capability**: Advanced persistent threats, zero-day exploits, social engineering.

**Motivation**: Espionage, intellectual property theft, disruption.

**Assumed Access**: Potentially unlimited given sufficient time and resources.

**Mitigations**:
- Accept that perfect defence is not achievable
- Focus on detection and response over prevention
- Segment workloads to limit impact
- Assume breach mentality: credential theft is inevitable, blast radius containment is critical
- Regular red team exercises

## Trust Boundaries

### Entra ID Tenant Boundary

The Entra ID tenant is the **primary trust boundary**.

A compromise of Global Administrator or privileged roles in Entra ID is a **full tenant compromise**.

**Controls**:
- No permanent Global Administrator assignments
- PIM enforced with approval workflow for all Tier 0 roles
- Break-glass accounts stored offline with alerting on use
- Conditional Access requires compliant devices for privileged access

**Residual Risk**: A determined attacker with sufficient time can escalate to Global Administrator. Detection and response speed is critical.

### Subscription Boundary

Azure subscriptions are **blast radius containers**.

A compromise of Owner or Contributor on a subscription must not allow lateral movement to other subscriptions or Entra ID.

**Controls**:
- Management group policies enforce guardrails at subscription creation
- Cross-subscription networking is explicit and audited
- Service principals scoped to single subscriptions or resource groups
- Azure RBAC assignments reviewed quarterly

**Residual Risk**: Subscriptions share the same Entra ID tenant. A privileged identity compromise can pivot.

### Network Boundary

Private networks (VNets, subnets) are **not** trust boundaries.

Network segmentation is defence-in-depth, not a primary control.

**Controls**:
- Network Security Groups enforce inbound/outbound rules
- Azure Firewall inspects inter-VNet traffic
- Private endpoints remove public internet exposure
- Assume breach: even internal networks are untrusted

**Residual Risk**: NSG misconfigurations, Azure Firewall bypasses, and stolen credentials undermine network isolation.

### Device Boundary

Corporate-managed devices are **more trusted** than personal devices, but neither is fully trusted.

**Controls**:
- Intune compliance enforced via Conditional Access
- Defender for Endpoint with EDR telemetry
- BitLocker full-disk encryption
- Windows Update for Business enforces patching

**Residual Risk**: User-initiated malware installation, phishing, and local admin abuse remain risks.

## High-Level Risk Posture

### Accepted Risks

The following risks are **accepted** because mitigations are not cost-effective or technically feasible:

1. **Zero-day exploits**: We cannot defend against unknown vulnerabilities. Patching lag is inevitable.
2. **Determined nation-state actors**: Perfect defence against APTs is not achievable. Focus is on detection and containment.
3. **Third-party SaaS provider compromise**: We trust Microsoft and key vendors. Their compromise is outside our control.
4. **Operational errors by privileged users**: Automation reduces human error but cannot eliminate it.

### Unacceptable Risks

The following risks are **unacceptable** and must be mitigated:

1. **Permanent privileged access**: All Tier 0 and Tier 1 access must be time-limited.
2. **Unencrypted data at rest**: All storage accounts, databases, and managed disks must use encryption.
3. **Public internet exposure of PaaS services**: Private endpoints or service endpoints are mandatory.
4. **Credential storage in code or configuration**: Secrets must be in Key Vault or managed identities only.
5. **Unrestricted lateral movement**: Network segmentation and identity controls must limit pivoting.

### Risk Appetite

The organisation's risk appetite is **low**.

We prefer operational friction and delivery delays over security compromises.

Where a control conflicts with delivery velocity, the control wins unless an executive-approved exception is granted.

This is non-negotiable.