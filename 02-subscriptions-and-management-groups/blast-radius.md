# Blast Radius

## Failure Containment Strategies

The platform must contain failures and prevent cascading compromises.

A security incident, misconfiguration, or outage in one subscription must not affect others.

## What is Blast Radius?

Blast radius is the **scope of impact** when something goes wrong.

Examples:

- A compromised VM in Subscription A cannot pivot to Subscription B
- A misconfigured Azure Policy in the "Sandbox" management group does not affect production
- A deleted VNet in one workload does not disrupt networking for others

**Goal**: Minimise blast radius at every layer (identity, networking, policy, RBAC).

## Identity Blast Radius

### Service Principal Scope

Service principals (automation identities) must be scoped to the **minimum necessary** scope.

**Bad**: Service principal with Contributor at Tenant Root.

**Impact**: Compromise grants access to all subscriptions. Full tenant compromise.

**Good**: Service principal with Contributor scoped to a single resource group.

**Impact**: Compromise affects only resources in that resource group.

**Example**:
```
Terraform service principal for "ecommerce" workload:
- Contributor on resource group "rg-ecommerce-production"
- No access to other resource groups or subscriptions
```

### User RBAC Scope

Users must have access only to subscriptions they need.

**Bad**: User with Reader across all subscriptions.

**Impact**: Credential compromise allows reconnaissance of entire tenant.

**Good**: User with Reader on their specific workload subscriptions only.

**Impact**: Credential compromise limited to their workloads.

### PIM Blast Radius

Privileged Identity Management (PIM) limits blast radius by time-boxing access.

**Example**: User activates Owner on a subscription for 4 hours to perform emergency change. After 4 hours, access is revoked automatically.

**Impact of Compromise**: Attacker has limited time window. Blast radius is temporal, not just spatial.

## Network Blast Radius

### VNet Peering as Trust Relationships

VNet peering creates bidirectional connectivity between VNets.

**Default**: No VNet peering between subscriptions. Workloads are isolated by default.

**Exception**: Workload VNets peer to hub VNet. All inter-workload traffic routes through Azure Firewall in hub.

**Why**: Firewall enforces allow/deny rules. Direct peering bypasses inspection.

**Example**:
```
Subscription A (VNet-A) <--peer--> Hub VNet <--peer--> Subscription B (VNet-B)
```

Traffic from VNet-A to VNet-B is routed through hub firewall. Firewall blocks or allows based on rules.

### Network Security Groups (NSGs)

NSGs limit inbound and outbound traffic per subnet.

**Default-Deny**: All inbound traffic is blocked unless explicitly allowed.

**Example**:
- Web tier subnet: Allow inbound HTTPS (443) from internet. Block all other inbound.
- Database tier subnet: Allow inbound traffic from app tier only. Block all other inbound.

**Blast Radius**: Compromised VM in web tier cannot directly access database tier unless NSG rules permit it.

### Private Endpoints

Private endpoints remove public internet exposure of PaaS services.

**Without Private Endpoint**:
- Storage account has public endpoint (e.g., `storageaccount.blob.core.windows.net`)
- Accessible from anywhere with credentials
- Compromise of credentials = data exfiltration

**With Private Endpoint**:
- Storage account accessible only via private IP in VNet
- No public internet access
- Attacker must first compromise a VM in the VNet, then access storage account

**Blast Radius**: Private endpoints limit access to internal network only.

## Subscription Blast Radius

### Subscription as Failure Domain

Subscriptions are the **primary blast radius boundary** in Azure.

**Why**: Azure Policy, RBAC, and networking are scoped to subscriptions (or management groups).

**Example**:
- Subscription A has misconfigured Azure Firewall. Internet access is blocked.
- Subscription B is unaffected.

### Cross-Subscription Dependencies

Workloads should minimise dependencies on other subscriptions.

**Bad**: Workload in Subscription A depends on storage account in Subscription B.

**Impact**: Failure or misconfiguration in Subscription B breaks Workload A.

**Good**: Workload in Subscription A contains all dependencies within the same subscription.

**Impact**: Failure is contained to Subscription A.

**Exception**: Shared platform services (hub network, central logging) are intentionally cross-subscription dependencies. These are treated as critical infrastructure.

## Management Group Blast Radius

### Policy Assignment Scope

Azure Policy assigned at management group affects all child subscriptions.

**Example**:
- Policy assigned at "Production" management group: "Block all public IPs".
- All subscriptions in "Production" inherit this policy.
- Subscriptions in "Sandbox" management group are unaffected.

**Blast Radius of Misconfigured Policy**:
- Policy at root management group affects entire tenant.
- Policy at lower-level management group affects only that branch.

**Recommendation**: Test policies in non-production management group before applying to production.

### RBAC Assignment Scope

RBAC at management group scope grants access to all child subscriptions.

**Example**: User with Contributor at "Non-Production" management group has Contributor on all non-prod subscriptions.

**Blast Radius of Compromised Identity**:
- Compromise affects all subscriptions in that management group.
- Does not affect subscriptions in other management groups.

**Recommendation**: Prefer RBAC at subscription or resource group scope. Use management group scope only for platform-wide roles.

## Resource-Level Blast Radius

### Resource Locks

Resource locks prevent accidental deletion or modification.

**Types**:
1. **CanNotDelete**: Resource cannot be deleted but can be modified.
2. **ReadOnly**: Resource cannot be deleted or modified.

**Use Cases**:
- Hub VNet: Apply CanNotDelete lock. Prevents accidental deletion of critical infrastructure.
- Production databases: Apply ReadOnly lock during maintenance freeze. Prevents unintended changes.

**Blast Radius**: Deletion of unlocked resource affects only that resource. Deletion of critical resource (e.g., hub VNet) affects entire tenant.

**Recommendation**: Apply locks to critical resources in platform subscriptions. Locks are optional in workload subscriptions (delivery teams own the risk).

### Soft Delete and Backup

Soft delete retains deleted resources for a recovery period.

**Examples**:
- Key Vault: Soft delete enabled. Deleted secrets recoverable for 90 days.
- Storage Account: Soft delete enabled. Deleted blobs recoverable for 7 days.

**Blast Radius**: Accidental deletion is recoverable. Malicious deletion with purge still possible (requires elevated permissions).

**Recommendation**: Enable soft delete on all production resources. Backups provide secondary recovery path.

## Blast Radius During Incidents

### Incident Response Priorities

When an incident occurs, containment priorities are:

1. **Isolate compromised identity**: Disable user account or service principal. Prevents lateral movement.
2. **Isolate compromised subscription**: Block network egress via Azure Firewall. Prevents data exfiltration.
3. **Isolate compromised workload**: Shut down VMs or App Services. Prevents further attacker activity.

**Blast Radius Trade-Off**: Aggressive isolation may break legitimate dependencies. Incident response must balance containment with business continuity.

### Post-Incident Blast Radius Assessment

After an incident, assess:

- Which subscriptions were affected?
- Which identities were compromised?
- Which data was accessed or exfiltrated?
- Could the incident have been contained faster with better segmentation?

**Outcome**: Incident post-mortem drives architectural improvements (e.g., additional network segmentation, tighter RBAC scoping).

## Known Failure Modes

### 1. Shared Service Compromised

**Scenario**: Hub VNet or central logging subscription is compromised.

**Blast Radius**: All workloads are affected. Hub network is single point of failure.

**Mitigation**: Treat platform subscriptions as Tier 0 assets. PIM-only access. Immutable audit logging. Real-time alerting on changes.

### 2. Cross-Subscription Networking Escalation

**Scenario**: Attacker compromises Subscription A, pivots via VNet peering to Subscription B.

**Blast Radius**: Multiple subscriptions affected.

**Mitigation**: Azure Firewall in hub inspects all inter-VNet traffic. NSGs enforce least-privilege traffic rules.

### 3. Service Principal Over-Privileged

**Scenario**: Terraform service principal with Contributor at management group scope is compromised.

**Blast Radius**: All subscriptions in management group are at risk.

**Mitigation**: Scope service principals to resource groups or subscriptions, not management groups.

### 4. Policy Exemption Cascade

**Scenario**: Policy exemptions granted liberally. Subscriptions accumulate uncontrolled exemptions.

**Blast Radius**: Security posture degrades incrementally. Exemptions become de facto policy.

**Mitigation**: Time-box exemptions. Quarterly audit. Escalate repeated exemptions to executive leadership.

## Blast Radius Design Checklist

- [ ] Service principals scoped to resource groups, not subscriptions
- [ ] VNet peering explicit and minimised
- [ ] Azure Firewall inspects inter-VNet traffic
- [ ] NSGs enforce default-deny on all subnets
- [ ] Private endpoints used for all PaaS services
- [ ] Subscriptions contain related workloads only (no "junk drawer" subscriptions)
- [ ] Platform subscriptions have PIM-only access
- [ ] Resource locks on critical infrastructure
- [ ] Soft delete enabled on production resources
- [ ] Policy exemptions time-boxed and audited

Blast radius containment is not optional. It is the difference between a localised incident and a tenant-wide compromise.