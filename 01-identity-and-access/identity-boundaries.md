# Identity Boundaries

## Identity as the Primary Control Plane

In Azure and Microsoft 365, **identity is the perimeter**.

Network segmentation, firewalls, and encryption are defence-in-depth. Identity is the primary control.

A compromised identity can:

- Pivot across subscriptions within the same tenant
- Access PaaS services regardless of network isolation
- Exfiltrate data via legitimate application access
- Modify Azure Policy and undermine platform controls

**Platform design must assume identities will be compromised.**

## Entra ID Trust Assumptions

### What Entra ID Provides

- Centralised authentication and authorisation for Azure, Microsoft 365, and federated SaaS applications
- Conditional Access policies enforcing device compliance, location, and risk-based controls
- Privileged Identity Management (PIM) for just-in-time access to administrative roles
- Identity Protection with real-time risk detection and remediation

### What Entra ID Does NOT Provide

- Protection against credential theft if MFA is bypassed (e.g., via consent phishing)
- Prevention of privilege escalation if RBAC is misconfigured
- Immutable audit logs (Entra ID audit logs can be modified by Global Administrators)
- Guaranteed availability (Entra ID outages have occurred and will occur again)

### Trust Model

We **trust** Entra ID to:

- Authenticate users with strong cryptographic guarantees
- Enforce Conditional Access policies consistently
- Detect anomalous sign-ins and flag risky users

We **do not trust** Entra ID to:

- Prevent all sophisticated phishing attacks
- Guarantee audit log integrity against privileged attackers
- Remain available during regional failures

**Mitigation**: Break-glass accounts, offline backups of privileged role assignments, and defensive RBAC design.

## Azure RBAC as an Identity Boundary

Azure RBAC (Role-Based Access Control) defines **what identities can do** on Azure resources.

### RBAC Principles

1. **Least Privilege by Default**

   Identities start with no access. Access is granted explicitly, not inherited.

   **Why**: Overly permissive RBAC is the most common misconfiguration in Azure. Default-deny is the only safe posture.

2. **Explicit Deny Over Implicit Allow**

   Azure Policy can block actions even if RBAC permits them.

   **Why**: RBAC is a coarse-grained control. Azure Policy enforces fine-grained compliance (e.g., blocking public IP creation, enforcing encryption).

3. **Separation of Platform and Workload Permissions**

   Platform teams have Owner on management groups. Delivery teams have Contributor on resource groups.

   **Why**: Platform teams control infrastructure. Delivery teams control workloads. Conflation creates security gaps.

4. **No User-Assigned Privileged Roles in Production**

   Contributor and Owner on production subscriptions are granted via PIM only.

   **Why**: Permanent access enables unaudited changes and increases blast radius of credential compromise.

### RBAC Anti-Patterns

The following RBAC patterns are **prohibited**:

1. **Owner at Subscription Scope for Delivery Teams**

   Delivery teams do not need to create RBAC assignments or modify Azure Policy. Contributor suffices.

2. **Reader Access for Auditors Across All Subscriptions**

   Auditors should not have standing access. PIM-enabled Reader access with justification is required.

3. **Service Principals with Owner at Tenant Root**

   Service principals should be scoped to resource groups. Tenant root access enables cross-subscription privilege escalation.

## Managed Identity vs Service Principal

### Managed Identities (Preferred)

Managed identities are Azure-native identities with no credential material.

**Advantages**:
- Credentials are managed by Azure and rotated automatically
- Cannot be exfiltrated or reused outside the resource
- No secrets to store in Key Vault or CI/CD variables

**Disadvantages**:
- Only work within Azure (cannot authenticate to external systems)
- Limited to a single resource (system-assigned) or shared across resources (user-assigned)

**When to Use**: Always, for Azure-to-Azure authentication.

### Service Principals (When Required)

Service principals are Entra ID identities with explicit credentials (certificate or secret).

**Advantages**:
- Work across Azure, Microsoft 365, and external SaaS systems
- Credentials can be rotated manually

**Disadvantages**:
- Credentials are exposed and can be stolen
- Long-lived credentials are high-value targets

**When to Use**: Only when managed identities are not supported (e.g., GitHub Actions, Terraform, external APIs).

**Controls**:
- Certificate-based credentials preferred over secrets
- Secrets expire after 90 days maximum
- Service principals scoped to resource groups, not subscriptions
- Azure Policy blocks creation of secrets with >90 day expiry

## Cross-Tenant Identity Boundaries

Azure supports B2B (guest users) and cross-tenant service principal access.

**Threat**: Guest users and external service principals can pivot across tenants if not explicitly blocked.

### Guest User Controls

- Guest users must be explicitly invited and approved
- Guest users have restricted directory permissions by default
- Conditional Access policies apply to guest users (device compliance, MFA)
- Guest user access is reviewed quarterly

### Cross-Tenant Service Principal Controls

- External service principals must be registered in the tenant's Enterprise Applications
- Admin consent is required for all external applications
- Permissions granted to external apps are scoped to specific resources
- Third-party SaaS integrations are audited monthly

**Hard Stop**: Cross-tenant access to production subscriptions is prohibited without executive approval.

## Known Failure Modes

### 1. Entra ID Outage

**Scenario**: Entra ID becomes unavailable. No authentication or authorisation possible.

**Impact**: All Azure and Microsoft 365 services are inaccessible.

**Mitigation**: Break-glass accounts with offline authentication. On-premises AD DS with federation is NOT a mitigation (it relies on Entra ID for cloud access).

### 2. Conditional Access Policy Misconfiguration

**Scenario**: A Conditional Access policy blocks all users, including Global Administrators.

**Impact**: Tenant lockout. No one can sign in.

**Mitigation**: Break-glass accounts excluded from Conditional Access. Test policies in report-only mode before enforcement.

### 3. PIM Approval Workflow Failure

**Scenario**: PIM approvers are unavailable or PIM is misconfigured.

**Impact**: Privileged access requests are blocked, preventing incident response.

**Mitigation**: Multiple PIM approvers. Break-glass accounts do not require PIM activation.

### 4. Managed Identity Deletion

**Scenario**: A managed identity is deleted while workloads depend on it.

**Impact**: Workload authentication fails. Applications cannot access storage, databases, or Key Vaults.

**Mitigation**: Terraform prevents accidental deletion. Managed identity lifecycle is tied to resource lifecycle.

### 5. Service Principal Secret Expiry

**Scenario**: Service principal secret expires without rotation.

**Impact**: CI/CD pipelines fail. Automation stops.

**Mitigation**: Azure Policy enforces short-lived secrets. Monitoring alerts on expiry 30 days in advance.

## Identity Boundaries are Not Perfect

No amount of RBAC, Conditional Access, or PIM prevents **all** identity compromise.

Defence-in-depth is mandatory:

- Network segmentation limits lateral movement
- Private endpoints reduce attack surface
- Audit logging enables detection
- Threat intelligence identifies known malicious IPs and actors

Identity is the **primary** control plane, not the **only** control plane.