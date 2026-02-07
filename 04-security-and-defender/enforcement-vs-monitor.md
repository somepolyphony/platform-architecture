# Enforcement vs Monitor

## When to Enforce, When to Observe

Security controls can be **preventative** (enforcement) or **detective** (monitoring).

Both are necessary. Choosing incorrectly creates either security gaps or operational paralysis.

## Enforcement (Preventative Controls)

Enforcement **blocks** non-compliant actions before they occur.

**Implementation**: Azure Policy with **Deny** effect, Conditional Access with **Block** action, NSG rules.

### When to Enforce

Enforcement is appropriate when:

1. **Non-compliance is unacceptable**

   Security requirements where violation creates immediate, unacceptable risk.

   **Examples**:
   - Encryption-at-rest (data breach risk)
   - No public IPs on VMs (direct internet exposure)
   - MFA for all identities (credential theft mitigation)

2. **Remediation is expensive or impossible**

   Once a resource is deployed incorrectly, fixing it is difficult or requires downtime.

   **Examples**:
   - Storage account without encryption (cannot enable encryption retroactively without data migration)
   - VM deployed in wrong region (requires redeployment)

3. **Compliance is regulatory**

   Regulatory requirements (PCI-DSS, HIPAA, GDPR) do not permit non-compliance, even temporarily.

   **Examples**:
   - PCI-DSS requires encryption-in-transit for cardholder data
   - HIPAA requires access logging for protected health information (PHI)

### Enforcement Trade-Offs

**Benefit**: Guarantees compliance. No human error or negligence.

**Cost**:
- Delivery teams are blocked when enforcement is too restrictive
- False positives stop legitimate work
- Incident response may be delayed if break-glass process is slow
- Innovation is constrained

**Example Failure**:
- Policy blocks all public IPs on VMs
- Delivery team needs to deploy a bastion host (requires public IP)
- Delivery blocked until exemption is granted
- Incident: Public-facing workload cannot be deployed in time for launch

**Mitigation**: Enforcement must have an exception process (see [exception-handling.md](../01-identity-and-access/exception-handling.md)).

## Monitoring (Detective Controls)

Monitoring **observes** and **alerts** on non-compliant actions after they occur.

**Implementation**: Azure Policy with **Audit** effect, Defender for Cloud alerts, Sentinel SIEM.

### When to Monitor Only

Monitoring is appropriate when:

1. **Non-compliance is tolerable temporarily**

   Violation creates risk but can be remediated after detection.

   **Examples**:
   - VM without backup configured (can be fixed within days)
   - Resource missing tags (can be added retroactively)
   - Diagnostic settings not forwarding logs (can be enabled post-deployment)

2. **False positives are common**

   Enforcement would block too many legitimate actions.

   **Examples**:
   - Anomalous sign-ins (legitimate user travelling may trigger alert)
   - Unusual resource access (legitimate workload scaling may appear anomalous)

3. **Enforcement is technically impossible**

   Azure does not provide a mechanism to block the action.

   **Examples**:
   - Insider data exfiltration (cannot block all data downloads; monitoring detects anomalous patterns)
   - Privilege escalation via application vulnerability (cannot prevent; monitoring detects unusual RBAC changes)

### Monitoring Trade-Offs

**Benefit**: Does not block delivery teams. Allows experimentation. Reduces false positive impact.

**Cost**:
- Non-compliance persists until detected and remediated
- Relies on humans actioning alerts (alert fatigue is real)
- Incident response is reactive, not proactive

**Example Failure**:
- Policy audits storage accounts without private endpoints
- Delivery team deploys storage account with public endpoint
- Alert is generated but ignored (alert fatigue)
- Storage account is compromised via public endpoint
- Breach detected weeks later during incident response

**Mitigation**: Monitoring must be paired with **response SLAs** and **escalation** for ignored alerts.

## Hybrid Approach (Audit, Then Enforce)

The safest rollout strategy is:

1. **Phase 1: Audit Only**

   Deploy policy in Audit mode. Observe compliance rate. Identify false positives.

   **Duration**: 2-4 weeks in non-production, 4-8 weeks in production.

2. **Phase 2: Remediation**

   Delivery teams fix non-compliant resources. Platform team refines policy to reduce false positives.

3. **Phase 3: Enforcement**

   Once compliance is high (>95%), switch policy to Deny mode.

**Why This Works**: Delivery teams are not surprised. False positives are ironed out before enforcement.

## Enforcement vs Monitoring by Control Type

| Control Type                  | Enforcement (Deny)       | Monitoring (Audit)       | Rationale                                      |
|-------------------------------|--------------------------|--------------------------|------------------------------------------------|
| Encryption-at-rest            | ✅ Enforce               |                          | Non-compliance is unacceptable                 |
| Private endpoints (production)| ✅ Enforce               |                          | Public exposure is unacceptable                |
| Private endpoints (non-prod)  |                          | ✅ Monitor               | Non-production risk is lower                   |
| Tagging                       |                          | ✅ Monitor               | Missing tags do not create security risk       |
| VM backup configuration       |                          | ✅ Monitor               | Can be remediated post-deployment              |
| Public IPs on VMs             | ✅ Enforce               |                          | Direct internet exposure is unacceptable       |
| MFA for all identities        | ✅ Enforce               |                          | Password-only authentication is unacceptable   |
| Anomalous sign-ins            |                          | ✅ Monitor               | False positives are high                       |
| Unusual RBAC changes          |                          | ✅ Monitor               | Some changes are legitimate                    |
| Defender for Cloud enabled    | ✅ Enforce               |                          | Detection capability is mandatory              |
| Soft delete on Key Vault      | ✅ Enforce (Modify)      |                          | Prevents accidental secret deletion            |
| Diagnostic settings           |                          | ✅ Monitor               | Can be enabled retroactively                   |
| VM SKU restrictions (non-prod)|                          | ✅ Monitor               | Cost control, not security                     |

## Enforcement Maturity Model

Organisations progress through enforcement maturity over time.

### Level 1: Manual Enforcement

**Characteristics**:
- Security controls are documented but not enforced
- Delivery teams self-attest compliance
- Audits identify non-compliance retroactively

**Risk**: High. Compliance depends on human behaviour.

### Level 2: Monitoring Only

**Characteristics**:
- Azure Policy in Audit mode
- Alerts generated for non-compliance
- Security team follows up manually

**Risk**: Moderate. Non-compliance is detected but remediation is slow.

### Level 3: Selective Enforcement

**Characteristics**:
- Critical controls enforced (encryption, private endpoints)
- Non-critical controls monitored (tagging, backup)
- Exception process exists

**Risk**: Low. Balance between security and delivery velocity.

### Level 4: Full Enforcement

**Characteristics**:
- All security controls enforced
- Exceptions are rare and time-boxed
- Automation remediates non-compliance (e.g., DeployIfNotExists policies)

**Risk**: Very low. Non-compliance is impossible.

**Trade-Off**: Innovation is constrained. Break-glass process is critical.

**Recommendation**: Most organisations should target **Level 3**. Level 4 is appropriate only for highly regulated industries (finance, healthcare).

## Known Failure Modes

### 1. Enforcement Without Exception Process

**Scenario**: Policy enforced with Deny effect. No exception process exists.

**Impact**: Delivery teams are blocked. Shadow IT increases. Legitimate workloads cannot be deployed.

**Mitigation**: All enforcement policies must have a documented exception process.

### 2. Monitoring Without Response SLA

**Scenario**: Policies in Audit mode generate alerts. No one actions them.

**Impact**: Non-compliance persists indefinitely. Monitoring is useless.

**Mitigation**: Define response SLAs for alerts. Escalate unactioned alerts to leadership.

### 3. Enforcement is Too Aggressive

**Scenario**: Policies enforce controls that are incompatible with some workloads.

**Impact**: Delivery teams cannot deploy. Business objectives missed.

**Mitigation**: Test policies in Audit mode first. Collect feedback. Refine before enforcement.

### 4. Monitoring is Ignored (Alert Fatigue)

**Scenario**: High volume of low-severity alerts. SOC ignores all alerts.

**Impact**: High-severity alerts missed.

**Mitigation**: Tune alerts. Suppress false positives. Prioritise high-severity alerts.

## Response SLAs for Monitoring

Monitoring is only effective if alerts are actioned.

### High-Severity Alerts

**Examples**:
- VM without endpoint protection in production
- Storage account with public network access containing PII
- Unusual RBAC changes (potential privilege escalation)

**SLA**: Investigate within **4 hours**. Remediate within **24 hours**.

### Medium-Severity Alerts

**Examples**:
- VM without backup configuration
- Resource missing tags
- Diagnostic settings not configured

**SLA**: Investigate within **2 business days**. Remediate within **1 week**.

### Low-Severity Alerts

**Examples**:
- VM SKU exceeds recommendation
- Unused resource flagged for deletion

**SLA**: Investigate within **1 week**. Remediate within **1 month** or accept risk.

## Enforcement vs Monitoring Checklist

- [ ] Critical controls enforced (encryption, private endpoints, MFA)
- [ ] Non-critical controls monitored (tagging, backup, diagnostic settings)
- [ ] Exception process exists for all enforced controls
- [ ] Monitoring alerts have defined response SLAs
- [ ] Policies tested in Audit mode before Deny mode
- [ ] Quarterly review of enforcement vs monitoring balance

Enforcement and monitoring are not mutually exclusive. Defence-in-depth requires both.