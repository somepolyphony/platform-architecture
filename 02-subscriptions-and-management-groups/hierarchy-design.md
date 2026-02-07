# Hierarchy Design

## Management Group Rationale

Azure management groups provide hierarchical policy and RBAC inheritance across subscriptions.

Every Azure subscription must belong to a management group. Subscriptions outside the hierarchy are **ungoverned** and represent security debt.

## Why Management Groups Exist

### 1. Centralised Policy Enforcement

Azure Policy assigned at management group scope applies to all child subscriptions automatically.

**Why This Matters**: Manual policy application per subscription does not scale. Delivery teams can create subscriptions faster than platform teams can secure them.

**Example**: A policy requiring encryption-at-rest on all storage accounts is assigned at the root management group. Every new subscription inherits this policy without manual intervention.

### 2. Blast Radius Containment

Management groups segment organisational risk.

A subscription in the "Production" management group cannot accidentally inherit policies or RBAC from the "Sandbox" management group.

**Why This Matters**: Overly permissive policies in development or sandbox environments must not leak into production.

### 3. RBAC Delegation

Platform teams can grant Contributor access at a management group scope without granting Tenant Root access.

**Example**: A delivery team owns all subscriptions under the "Workload-A" management group. They have Contributor at that scope but cannot affect "Workload-B" subscriptions.

## Management Group Structure

### Root Management Group (Tenant Root)

**Scope**: Entire Azure tenant. All subscriptions inherit policies from here unless explicitly excluded.

**Policies Assigned**:
- Require tags (environment, cost centre, owner)
- Block public IP creation on VMs
- Enforce Defender for Cloud on all subscriptions
- Require encryption-at-rest for storage accounts and managed disks

**RBAC**: Platform lead and security architect have Owner (via PIM only). No standing access.

### Platform Management Group

**Scope**: Platform services (hub network, central logging, Defender for Cloud management).

**Subscriptions**:
- `Platform-Hub-Network`
- `Platform-Logging`
- `Platform-Security`

**Policies Assigned**:
- Stricter than root (e.g., no internet egress without explicit approval)
- Mandatory private endpoints for all PaaS services

**RBAC**: Platform team has Contributor. Delivery teams have no access.

### Production Management Group

**Scope**: Production workloads. Highest security posture.

**Policies Assigned**:
- No public endpoints permitted (private endpoints mandatory)
- Change control via Terraform only (Azure Portal changes blocked where possible)
- Mandatory audit logging with 365-day retention
- No deletion of resources without approval workflow

**RBAC**: Delivery teams have Reader access only. Changes via CI/CD pipelines with service principal authentication.

### Non-Production Management Group

**Scope**: Development, staging, and UAT workloads.

**Policies Assigned**:
- Less restrictive than production (e.g., public endpoints permitted for dev convenience)
- Shorter log retention (90 days)
- Cost controls (VM sizes capped, auto-shutdown enabled)

**RBAC**: Delivery teams have Contributor access.

### Sandbox Management Group

**Scope**: Experimentation, proof-of-concept, learning.

**Policies Assigned**:
- Minimal enforcement (primarily cost and time-boxing)
- Mandatory expiry date (subscriptions auto-deleted after 90 days)
- Network isolation (no connectivity to hub or production VNets)

**RBAC**: Users have Owner access within their sandbox subscriptions.

**Why Sandbox Exists**: Innovation and experimentation require freedom. Constraining sandboxes to production-level controls stifles learning and drives shadow IT.

## Management Group Anti-Patterns

### 1. Flat Hierarchy (No Management Groups)

**Problem**: All subscriptions at root level. No segmentation. Policies cannot be tailored per environment.

**Impact**: Over-permissive policies in production. Overly restrictive policies in development.

### 2. Too Many Layers

**Problem**: Management group hierarchy 5+ levels deep. Complex inheritance chains. Policies conflict.

**Impact**: Debugging policy application becomes impossible. Delivery teams cannot predict behaviour.

**Recommended Maximum**: 3 levels (root → environment → workload).

### 3. Per-Team Management Groups

**Problem**: Each delivery team gets its own management group.

**Impact**: Policy fragmentation. Inconsistent security posture. RBAC becomes unmanageable.

**Alternative**: Teams own subscriptions within shared management groups (e.g., all production workloads under "Production").

### 4. Management Groups as Cost Centres

**Problem**: Management groups aligned to billing cost centres rather than security boundaries.

**Impact**: Security and billing are different concerns. Conflating them creates governance gaps.

**Alternative**: Use Azure tags for cost allocation. Use management groups for security segmentation.

## Organisational Blast Radius

Management groups define **blast radius boundaries**.

A compromised subscription or identity should not cascade to unrelated workloads.

### Scenario: Compromised Production Subscription

**Blast Radius Without Management Groups**:
- Attacker pivots to other subscriptions via shared networking or RBAC inheritance
- All subscriptions in tenant are at risk

**Blast Radius With Management Groups**:
- Attacker is contained within the "Production" management group
- Cannot pivot to "Sandbox" or "Non-Production" management groups unless RBAC explicitly grants access

### Scenario: Misconfigured Policy

**Blast Radius Without Management Groups**:
- Policy applied at tenant root affects all subscriptions
- Breaking change impacts production and development simultaneously

**Blast Radius With Management Groups**:
- Policy applied at "Sandbox" management group affects only sandbox subscriptions
- Production workloads unaffected

### Scenario: Insider Threat

**Blast Radius Without Management Groups**:
- Malicious admin with Owner at tenant root can delete or modify all subscriptions

**Blast Radius With Management Groups**:
- Admin scoped to "Non-Production" management group cannot affect production
- Separation of duties limits damage

## Management Group Governance

### 1. Subscription Placement

All subscriptions must be placed in the correct management group at creation.

**Process**:
1. Delivery team requests subscription via ServiceNow
2. Platform team provisions subscription via Terraform
3. Subscription is placed in appropriate management group based on environment (production, non-production, sandbox)
4. RBAC is assigned based on subscription type

**Validation**: Quarterly audit scans for subscriptions outside management group hierarchy. Orphaned subscriptions are flagged for remediation.

### 2. Policy Inheritance

Policies assigned at parent management groups cannot be overridden at child subscriptions **unless explicitly exempted**.

**Example**: Root-level policy requires encryption-at-rest. A subscription cannot disable encryption without a documented exemption approved by security team.

### 3. RBAC Inheritance

RBAC assigned at management group scope inherits to all child subscriptions.

**Risk**: Over-privileged RBAC at management group level creates excessive blast radius.

**Control**: RBAC at management group scope is limited to Reader or Contributor. Owner is granted at subscription scope only.

### 4. Management Group Deletion

Management groups cannot be deleted if they contain subscriptions or child management groups.

**Why**: Prevents accidental deletion of governance structure.

**Process for Deletion**:
1. Move all subscriptions to different management group
2. Delete child management groups first (bottom-up)
3. Delete parent management group last

## Policy Scope and Precedence

Azure Policy can be assigned at:
1. Management group
2. Subscription
3. Resource group

**Precedence**: More specific assignments override less specific.

**Example**:
- Management group policy: "Block all public IPs"
- Subscription policy exemption: "Allow public IP on this specific VM"

Result: Public IP is allowed on the specific VM. All other VMs are blocked.

**Recommendation**: Assign policies at management group level for consistency. Use exemptions sparingly and with time-boxing.

## Known Failure Modes

### 1. Subscription Created Outside Management Group

**Scenario**: User with sufficient permissions creates subscription via Azure Portal without specifying management group.

**Impact**: Subscription inherits only tenant root policies. Missing environment-specific controls.

**Mitigation**: Azure Policy blocks subscription creation by users. All subscriptions provisioned via Terraform under platform team control.

### 2. Policy Conflict Between Layers

**Scenario**: Management group policy conflicts with subscription policy.

**Impact**: Policy evaluation fails or behaves unpredictably.

**Mitigation**: Policy assignments are tested in non-production before applying to production. Conflicts are resolved via exemptions, not overrides.

### 3. RBAC Creep

**Scenario**: Users accumulate RBAC assignments at multiple scopes (management group, subscription, resource group).

**Impact**: Effective permissions are unclear. Principle of least privilege is violated.

**Mitigation**: Quarterly RBAC audit. Remove redundant assignments. Prefer assignment at lowest necessary scope.

### 4. Management Group Misconfiguration

**Scenario**: Subscriptions are placed in wrong management group (e.g., production workload in "Sandbox").

**Impact**: Wrong policies applied. Security posture weakened or unnecessarily restrictive.

**Mitigation**: Subscription provisioning is automated via Terraform. Manual subscription creation is blocked.

Management groups are the **foundation** of Azure governance. Without them, policy enforcement is manual, inconsistent, and unscalable.