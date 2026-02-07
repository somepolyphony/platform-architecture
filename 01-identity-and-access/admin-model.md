# Admin Model

## Tiered Admin Model

Administrative access is segmented into tiers to limit blast radius and prevent privilege escalation.

This model is derived from Microsoft's Entra ID tiering but adapted for operational reality.

### Tier 0: Identity and Platform

**Scope**: Entra ID, management groups, Azure Policy, central hub network.

**Examples**:
- Global Administrator (Entra ID)
- Privileged Role Administrator (Entra ID)
- User Administrator (Entra ID)
- Owner on Tenant Root Management Group

**Risk**: Tier 0 compromise is full tenant compromise. A Tier 0 identity can:
- Modify Conditional Access policies
- Grant themselves access to any subscription
- Delete audit logs
- Reset passwords for any user

**Controls**:
- No permanent Tier 0 assignments
- PIM with approval workflow required
- Break-glass accounts are the only exception
- Tier 0 identities must authenticate from compliant, managed devices only
- Conditional Access requires phishing-resistant MFA (FIDO2 or Windows Hello)

**Who Holds Tier 0 Access**: Platform lead, security architect (time-limited via PIM only).

### Tier 1: Subscription and Workload

**Scope**: Azure subscriptions, resource groups, networking resources within VNets.

**Examples**:
- Owner on subscription (PIM only)
- Contributor on subscription
- Network Contributor on hub VNet

**Risk**: Tier 1 compromise affects workloads and can pivot to other subscriptions if cross-subscription networking exists.

**Controls**:
- Contributor access is default; Owner is exceptional and PIM-enabled
- No lateral movement to Tier 0 unless explicitly escalated via PIM
- Tier 1 identities must authenticate from compliant devices
- Conditional Access requires MFA (FIDO2, authenticator app, or SMS as fallback)

**Who Holds Tier 1 Access**: Platform engineers, workload owners.

### Tier 2: Application and Data

**Scope**: Application-level access, data within databases or storage accounts.

**Examples**:
- Application Administrator in Entra ID
- SQL Server Administrator
- Storage Blob Data Contributor

**Risk**: Tier 2 compromise affects data but not infrastructure. Cannot pivot to Tier 1 or Tier 0 without credential theft or RBAC escalation.

**Controls**:
- Tier 2 identities have no RBAC on subscriptions or management groups
- Access to data is time-limited where possible
- Conditional Access requires MFA
- Audit logging is mandatory

**Who Holds Tier 2 Access**: Application developers, data engineers.

## Why Least Privilege Breaks Down in Practice

### 1. Azure RBAC is Too Coarse-Grained

RBAC roles are broad. "Contributor" allows modification of nearly all resources except RBAC assignments.

There is no "Deploy a VM but not delete it" role. There is no "Create storage accounts but not disable firewalls" role.

**Reality**: Delivery teams need Contributor to ship workloads. Fine-grained RBAC is not operationally feasible.

**Mitigation**: Azure Policy enforces guardrails that RBAC cannot (e.g., blocking public IP creation, enforcing encryption).

### 2. PIM Adds Latency to Incident Response

Privileged access requires approval. Approval workflows take time. Incidents do not wait for approval.

**Reality**: On-call engineers need immediate access during outages.

**Mitigation**: 
- Define "hot" identities with standing Contributor access for on-call engineers
- Limit standing access to 1-2 identities per team
- Audit standing access quarterly and justify its necessity

### 3. Cross-Functional Teams Require Broad Access

Platform engineers need Tier 1 access to subscriptions. They also need Tier 0 access to management groups. They also need Entra ID access to manage service principals.

**Reality**: Strict tiering requires separate identities for each tier, which is operationally infeasible.

**Mitigation**:
- Use a single identity with multiple PIM-enabled roles
- Activate roles based on task, not as default
- Log all role activations and audit monthly

### 4. Terraform Service Principals are Over-Privileged

Terraform needs to create, modify, and delete resources. This requires Contributor.

Contributor on a subscription can modify networking, delete databases, or reconfigure firewalls.

**Reality**: Terraform service principals are high-value targets. Credential theft enables infrastructure destruction.

**Mitigation**:
- Scope Terraform service principals to resource groups where possible
- Use separate service principals per environment (dev, staging, prod)
- Rotate credentials every 90 days
- Use Terraform Cloud or Azure DevOps with OIDC-based authentication instead of static secrets

## Role Separation

### Platform Team vs Delivery Team

| Responsibility            | Platform Team | Delivery Team |
|---------------------------|---------------|---------------|
| Management Group Policy   | Owner         | None          |
| Subscription Provisioning | Owner         | None          |
| Hub Network Configuration | Owner         | None          |
| Spoke Network (VNet)      | Contributor   | Reader        |
| Workload Resource Groups  | Reader        | Contributor   |
| Azure Policy Exemptions   | Approver      | Requester     |
| RBAC Assignments          | Approver      | Requester     |

**Why**: Platform teams enforce guardrails. Delivery teams operate within them. Conflation creates security gaps.

### Security Team vs Platform Team

| Responsibility                  | Security Team | Platform Team |
|---------------------------------|---------------|---------------|
| Conditional Access Policies     | Owner         | None          |
| Defender for Cloud Configuration| Owner         | None          |
| Azure Policy Definitions        | Contributor   | Contributor   |
| Incident Response               | Owner         | Contributor   |
| Security Exception Approval     | Approver      | Requester     |

**Why**: Security team owns policy and detection. Platform team implements enforcement. Neither should be blocked by the other.

## Break-Glass Accounts

See [break-glass.md](./break-glass.md) for detailed design.

Break-glass accounts have permanent Tier 0 access and bypass PIM and Conditional Access.

**Use cases**:
- Entra ID outage
- Conditional Access misconfiguration locking out all users
- PIM approval workflow failure

**Non-use cases**:
- Routine administrative tasks
- Incident response where PIM is available
- Bypassing policy because it is inconvenient

## Admin Account Hygiene

### 1. Dedicated Admin Identities

Administrators have two identities:
- **Standard user**: Email, Teams, Outlook, general productivity
- **Admin user**: Azure Portal, Entra ID, privileged access only

**Why**: Phishing emails target standard user accounts. Admin accounts should not receive email or browse the internet.

### 2. Phishing-Resistant MFA

Admin identities must use FIDO2 security keys or Windows Hello for Business.

SMS, phone call, and authenticator app push notifications are **not sufficient** for Tier 0 or Tier 1 access.

**Why**: SIM swapping and MFA fatigue attacks bypass SMS and push-based MFA.

### 3. Compliant Device Requirement

Admin identities can only authenticate from Intune-managed, compliant devices.

**Why**: Compromised devices can steal session tokens and bypass MFA.

### 4. No Admin Access from Personal Devices

Admin identities cannot authenticate from personal laptops, phones, or tablets.

**Why**: Personal devices are outside platform control. Malware, outdated OS, or missing EDR undermines security posture.

## Privileged Identity Management (PIM) Configuration

### Activation Duration

- Tier 0 roles: 1 hour maximum
- Tier 1 roles: 4 hours maximum
- Tier 2 roles: 8 hours maximum

**Why**: Shorter activations reduce blast radius of compromised sessions.

### Approval Workflow

- Tier 0 roles: Require approval from two approvers
- Tier 1 roles: Require approval from one approver
- Tier 2 roles: Self-activation (no approval) but audited

### Justification Requirement

All PIM activations require a free-text justification and incident ticket reference.

**Why**: Audit trail must connect privileged actions to business need.

### Activation Alerts

All Tier 0 and Tier 1 activations generate real-time alerts to the security team.

**Why**: Unauthorised or suspicious activations must be detected immediately.

## Known Failure Modes

### 1. PIM Approver Unavailable

**Scenario**: Tier 0 access required during an outage but approvers are offline.

**Mitigation**: Multiple approvers in different time zones. Break-glass accounts as last resort.

### 2. Tier 1 Identity Escalates to Tier 0

**Scenario**: A compromised Contributor account grants itself Owner at management group scope.

**Mitigation**: Azure Policy blocks Contributor accounts from creating RBAC assignments. Audit logs flag unexpected RBAC changes.

### 3. Service Principal Misconfiguration

**Scenario**: A service principal is granted Global Administrator in Entra ID instead of scoped Azure RBAC.

**Mitigation**: Separate service principals for Entra ID and Azure. Block service principals from Tier 0 roles via Conditional Access.

Admin models are imperfect. Detection and response are as important as prevention.