# Policy Authority

## Central Policy Ownership

Azure Policy is owned centrally by the platform and security teams.

Delivery teams do **not** create or modify policies.

## Why Centralised Policy Ownership

### 1. Consistency

All subscriptions must adhere to the same security baseline.

**Problem with Decentralised Policy**: Each delivery team creates their own policies. Security posture fragments.

**Example**:
- Team A blocks public IPs on VMs
- Team B allows public IPs
- Security team cannot guarantee consistent controls

**Solution**: Platform team creates policies. All subscriptions inherit.

### 2. Expertise

Policy creation requires deep knowledge of Azure Resource Manager, policy effects, and compliance frameworks.

**Reality**: Delivery teams do not have this expertise. Centralising policy ownership ensures correctness.

### 3. Auditability

External auditors require evidence of consistent controls across the tenant.

**Problem with Decentralised Policy**: Auditors must review policies in every subscription. Does not scale.

**Solution**: Policies assigned at management group level. Single source of truth.

## Policy Ownership Model

| Policy Scope              | Owner          | Approver       | Requester        |
|---------------------------|----------------|----------------|------------------|
| Tenant Root (All Subs)    | Security Team  | CISO           | Platform/Security|
| Management Group          | Platform Team  | Platform Lead  | Platform/Delivery|
| Subscription              | Platform Team  | Platform Lead  | Delivery Team    |
| Resource Group            | Delivery Team  | Platform Team  | Delivery Team    |

**In Practice**: Most policies are assigned at management group or tenant root. Resource group policies are rare.

## Policy Categories

### 1. Security Baseline Policies (Mandatory)

These policies enforce minimum security posture. No exceptions without executive approval.

**Examples**:
- **Encryption-at-rest**: All storage accounts, managed disks, and databases must have encryption enabled.
- **Private endpoints**: PaaS services in production must use private endpoints.
- **No public IPs**: VMs cannot have public IP addresses.
- **Defender for Cloud**: All subscriptions must have Defender plans enabled.
- **Tagging**: All resources must have tags (owner, cost centre, environment).

**Policy Effect**: **Deny** (blocks non-compliant resource creation).

**Ownership**: Security team.

### 2. Compliance Policies (Mandatory for Regulated Workloads)

These policies enforce regulatory requirements (ISO 27001, PCI-DSS, HIPAA).

**Examples**:
- **Audit logging**: All resources must forward logs to central Log Analytics workspace (365-day retention).
- **Geo-replication**: Production databases must have geo-redundant backups.
- **Immutable storage**: Compliance data must use immutable blob storage.

**Policy Effect**: **Audit** or **Deny** (depends on regulation).

**Ownership**: Security team with input from compliance team.

### 3. Cost Control Policies (Recommended)

These policies prevent excessive costs.

**Examples**:
- **VM SKU restrictions**: Non-production subscriptions can only deploy Standard_D4s_v3 or smaller.
- **Auto-shutdown**: VMs in dev/test subscriptions automatically shut down at 19:00.
- **Unused resources**: Storage accounts with no activity for 90 days are flagged for deletion.

**Policy Effect**: **Deny** (for SKU restrictions) or **Audit** (for unused resources).

**Ownership**: Platform team with input from FinOps.

### 4. Operational Policies (Recommended)

These policies enforce operational best practices.

**Examples**:
- **Naming conventions**: Resources must follow `<type>-<environment>-<workload>-<region>` format.
- **Resource locks**: Critical resources must have CanNotDelete locks.
- **Diagnostic settings**: All resources must send logs to Log Analytics.

**Policy Effect**: **Audit** (alert on non-compliance, but do not block).

**Ownership**: Platform team.

## Policy Effects

Azure Policy supports multiple effects. Choosing the right effect is critical.

### 1. Deny

**Behaviour**: Blocks resource creation or modification if it does not comply with policy.

**Use Case**: Security baseline policies where non-compliance is unacceptable.

**Example**: Policy blocks creation of storage accounts without encryption-at-rest.

**Risk**: Overly aggressive Deny policies break legitimate workloads. Test in Audit mode first.

### 2. Audit

**Behaviour**: Allows resource creation but logs non-compliance.

**Use Case**: Soft enforcement. Delivery teams are notified of non-compliance but not blocked.

**Example**: Policy logs VMs without backup configured. VM is created, but alert is generated.

**Risk**: Non-compliance is ignored if alerts are not actioned.

### 3. Append

**Behaviour**: Adds properties to resources during creation.

**Use Case**: Enforcing tags or diagnostic settings.

**Example**: Policy appends "environment: production" tag to all resources in production subscriptions.

**Risk**: Cannot override existing properties. Limited use cases.

### 4. Modify

**Behaviour**: Changes resource properties during creation or update.

**Use Case**: Enforcing encryption or enabling features.

**Example**: Policy enables soft delete on Key Vaults automatically.

**Risk**: Unexpected behaviour if delivery teams are not aware of automatic modifications.

### 5. DeployIfNotExists (DINE)

**Behaviour**: Deploys additional resources if they do not exist.

**Use Case**: Ensuring monitoring or security tools are deployed.

**Example**: Policy deploys Defender for Endpoint agent to VMs if not present.

**Risk**: Requires managed identity for policy. Complex to configure.

### 6. AuditIfNotExists

**Behaviour**: Audits whether a related resource exists.

**Use Case**: Checking for dependent resources (e.g., backup policies, diagnostic settings).

**Example**: Policy audits whether VMs have Azure Backup configured.

**Risk**: Does not enforce. Audit logs must be actioned.

## Policy Testing and Rollout

Policies must be tested before enforcement.

### 1. Test in Sandbox

All new policies are created in a sandbox management group first.

**Process**:
1. Platform team writes policy definition
2. Policy assigned to "Sandbox" management group in Audit mode
3. Delivery teams deploy workloads in sandbox
4. Platform team reviews audit logs for false positives
5. Policy is refined based on feedback

**Timeline**: 2-4 weeks of testing.

### 2. Pilot in Non-Production

Once tested, policies are rolled out to non-production subscriptions in Audit mode.

**Process**:
1. Policy assigned to "Non-Production" management group in Audit mode
2. Delivery teams validate workloads remain functional
3. False positives are documented and exemptions created
4. After 2 weeks, policy is switched to Deny mode (if appropriate)

### 3. Rollout to Production

Finally, policies are rolled out to production.

**Process**:
1. Policy assigned to "Production" management group in Audit mode
2. Existing non-compliant resources are flagged
3. Delivery teams remediate non-compliant resources
4. After all resources are compliant, policy is switched to Deny mode

**Timeline**: 4-8 weeks from sandbox to production enforcement.

## Policy Exemptions

See [07-exceptions-and-risk-acceptance/exception-handling.md](../07-exceptions-and-risk-acceptance/exception-handling.md) for detailed process.

**Summary**:
- Exemptions are granted by security team
- Exemptions are time-boxed (maximum 30 days)
- Exemptions require justification and risk acceptance
- Exemptions are audited monthly

## Override Risks

### 1. Subscription-Level Policy Overrides Management Group Policy

**Scenario**: Delivery team has Owner on subscription. Creates policy that conflicts with management group policy.

**Impact**: Security baseline is undermined.

**Mitigation**: Azure RBAC prevents Contributor-level users from creating policies. Owner access is PIM-only.

### 2. Policy Exemption Abuse

**Scenario**: Delivery teams request excessive exemptions. Exemptions become de facto policy.

**Impact**: Security posture degrades incrementally.

**Mitigation**: Exemptions escalate to executive leadership after three renewals. Repeated exemptions trigger policy review.

### 3. Policy Effect Changed Without Approval

**Scenario**: Policy changed from Deny to Audit without security team approval.

**Impact**: Non-compliant resources are created.

**Mitigation**: Azure Activity Logs flag policy changes. Alerts sent to security team. Unauthorised changes are reverted.

## Policy Governance

### 1. Policy Review Cadence

All policies are reviewed quarterly.

**Review Includes**:
- Are policies still relevant?
- Have Azure features changed, requiring policy updates?
- Are exemptions still justified?
- Are there new policies required based on threat landscape?

### 2. Policy Change Control

All policy changes require:
- Pull request in Terraform repository
- Peer review by platform engineer
- Approval by security team (for security policies)
- Testing in sandbox and non-production

**No emergency policy changes.** Policy changes are planned and tested.

### 3. Policy Documentation

All policies must have:
- **Why**: Business or security justification
- **What**: Resources affected and policy effect
- **Trade-Off**: What breaks or becomes harder due to this policy
- **Exemption Process**: How to request an exception

Documentation is stored alongside policy definitions in Terraform repository.

## Known Failure Modes

### 1. Policy Denies Legitimate Resource

**Scenario**: Policy is too restrictive. Delivery team cannot deploy necessary resource.

**Impact**: Delivery blocked. Incident requires emergency exemption.

**Mitigation**: Policies tested in Audit mode first. Exemption process provides escape valve.

### 2. Policy Conflicts with Azure Service

**Scenario**: Azure service behaviour changes. Policy becomes incompatible.

**Example**: Azure introduces new VM SKU. Policy blocks it because it is not in approved list.

**Mitigation**: Monitor Azure service updates. Update policies proactively.

### 3. Policy Performance Impact

**Scenario**: Complex policy definitions slow resource creation.

**Impact**: Terraform deployments take longer.

**Mitigation**: Simplify policies where possible. Test performance in non-production.

### 4. Policy Scope Misconfiguration

**Scenario**: Policy accidentally assigned at tenant root instead of specific management group.

**Impact**: Policy affects all subscriptions, including sandboxes.

**Mitigation**: Terraform code review catches scope errors. Policies tested in sandbox first.

## Policy Authority Summary

- **Centralised ownership**: Platform and security teams own policies
- **Testing mandatory**: Policies tested in Audit mode before Deny mode
- **Exemptions time-boxed**: No permanent exemptions
- **Quarterly reviews**: Policies reviewed and updated
- **Documentation required**: All policies must explain why, what, and trade-offs

Policy is the enforcement layer above RBAC and networking. Without policy, controls are optional.