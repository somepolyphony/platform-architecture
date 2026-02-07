# Exception Process

## How Exceptions Are Requested

Exceptions to platform controls are inevitable. The process must be explicit, auditable, and time-boxed.

This document defines the request, approval, and lifecycle of exceptions.

## Exception Request Workflow

### Step 1: Requester Submits Request

**Method**: ServiceNow ticket (or equivalent ITSM system).

**Required Information**:
- **Requester**: Name and email
- **Resource**: Subscription, resource group, or specific resource ID
- **Control**: Which policy, RBAC, or security control requires exception
- **Justification**: Why exception is necessary (business need, technical limitation, regulatory requirement)
- **Duration**: How long exception is needed (maximum 30 days for most exceptions)
- **Approver**: Who should approve (security team, platform lead, or executive)
- **Risk Acknowledgement**: Requester acknowledges they own the risk during exception period

**Example Request**:
```
Requester: john.doe@company.com
Resource: /subscriptions/abc123/resourceGroups/rg-ecommerce-prod
Control: Azure Policy - Block Public IP on VMs
Justification: Legacy application requires public IP for external partner integration. 
              Cannot migrate to private endpoint due to vendor limitation.
Duration: 30 days (migration to private endpoint in progress)
Approver: security-team@company.com
Risk Acknowledgement: I acknowledge that public IP exposes VM to internet and increases attack surface.
```

### Step 2: Automated Validation

ServiceNow workflow validates request:
- All required fields completed
- Duration does not exceed maximum (30 days for RBAC/policy, 14 days for Conditional Access)
- Approver is authorised for exception type
- Requester is not requesting repeated exception (flags if >2 renewals)

**If Validation Fails**: Request is returned to requester with error message.

### Step 3: Approver Reviews Request

Approver receives notification (email + Teams).

**Approval Criteria**:
1. **Business need is documented**: Vague justifications rejected.
2. **Alternative solutions explored**: Exception is last resort.
3. **Duration is reasonable**: Open-ended exceptions rejected.
4. **Risk is understood**: Requester knows what control is bypassed and why.

**Approver Actions**:
- **Approve**: Proceeds to implementation
- **Reject**: Request closed with rejection reason
- **Request More Info**: Returns to requester for clarification

**SLA**: Approver responds within **2 business days** for non-urgent exceptions, **4 hours** for urgent exceptions.

### Step 4: Implementation

Platform or security team implements exception.

**Implementation Methods**:

1. **Azure Policy Exemption**

   Created via Terraform:
   ```hcl
   resource "azurerm_policy_exemption" "example" {
     name                 = "exception-ecommerce-publicip"
     policy_assignment_id = azurerm_policy_assignment.block_public_ips.id
     exemption_category   = "Waiver"
     expires_on           = "2026-03-07T00:00:00Z"  # 30 days from approval
     metadata = jsonencode({
       ticket_id  = "INC123456"
       requester  = "john.doe@company.com"
       approver   = "jane.smith@company.com"
     })
   }
   ```

2. **RBAC Elevation**

   Created via Azure Portal PIM or Terraform:
   ```hcl
   resource "azurerm_role_assignment" "example" {
     scope                = azurerm_resource_group.example.id
     role_definition_name = "Contributor"
     principal_id         = data.azuread_user.requester.object_id
     
     # Note: Azure RBAC assignments do not support expiry natively.
     # Expiry enforced via automation (see Step 6).
   }
   ```

3. **Conditional Access Exclusion**

   User added to exclusion group in Entra ID:
   ```powershell
   Add-AzureADGroupMember -ObjectId "ca-exclusion-group-id" -RefObjectId "user-object-id"
   ```

4. **Defender for Cloud Exemption**

   Created via Azure Portal or API (no Terraform support in 2026).

**Timeline**: Exception implemented within **4 hours** of approval for urgent requests, **1 business day** for non-urgent.

### Step 5: Notification

Requester receives confirmation:
- Exception has been granted
- Expiry date and time
- Reminder to submit new request if exception still needed after expiry

**Example Email**:
```
Subject: Exception Approved - INC123456

Your exception request has been approved and implemented.

Resource: /subscriptions/abc123/resourceGroups/rg-ecommerce-prod
Control: Azure Policy - Block Public IP on VMs
Duration: 30 days
Expiry: 2026-03-07 00:00:00 UTC

The exception will expire automatically. If you still require the exception after expiry, 
submit a new request via ServiceNow.

Risk: Public IP exposes VM to internet. Ensure VM is patched and Defender for Endpoint is active.
```

### Step 6: Expiry Enforcement

Exceptions expire automatically.

**Automation**:
- Azure Policy exemptions have `expires_on` field. Azure removes exemption automatically.
- RBAC assignments do not support expiry natively. Weekly automation checks for expired exceptions and removes RBAC assignments.
- Conditional Access exclusions removed via automation on expiry.
- Defender exemptions do not support expiry. Manual review monthly.

**Grace Period**: None. Upon expiry, exception is removed immediately.

**User Notification**: Email sent 7 days before expiry and on day of expiry.

## Exception Approval Authority

| Exception Type                  | Approver                          | Maximum Duration |
|---------------------------------|-----------------------------------|------------------|
| Azure Policy (Production)       | Security Team                     | 30 days          |
| Azure Policy (Non-Production)   | Platform Team                     | 90 days          |
| RBAC Elevation (Tier 1)         | Platform Lead                     | 30 days          |
| RBAC Elevation (Tier 0)         | Security Team + Executive Sponsor | 14 days          |
| Conditional Access Bypass       | Security Team                     | 14 days          |
| Defender for Cloud Exemption    | Security Team                     | 30 days          |
| Private Endpoint Exemption      | Security Team                     | 30 days          |
| Firewall Rule (Temporary)       | Platform Team                     | 14 days          |

**Why Different Approvers**: Risk level varies. Tier 0 access and Conditional Access bypass require highest authority.

## Exception Renewal

If exception is still required after expiry, **new request** must be submitted.

**Renewal is not automatic.** Each renewal requires:
1. Updated justification (why exception still needed)
2. Evidence that alternative solutions were re-evaluated
3. Fresh approval from same or different approver

**Escalation**: After **three renewals**, exception escalates to executive leadership for review.

**Why**: Repeated renewals indicate systemic issue. Either control should be adjusted, or workload should be re-architected.

## Exception Audit

### Monthly Exception Report

Platform team generates report of all active exceptions:
- Exception type and resource
- Requester and approver
- Duration and expiry date
- Number of renewals

**Recipients**: Security team, platform lead, executive leadership (if renewals >2).

### Quarterly Deep Dive

Security and platform teams meet to review:
- Exceptions with multiple renewals (candidates for permanent policy change or workload re-architecture)
- Exceptions that expired without renewal (validation that control was temporary)
- Patterns in exception requests (indication of systemic policy misalignment)

**Outcome**: Action items to adjust policies, update modules, or re-train delivery teams.

## Exception Abuse

### Detection

Exception abuse is flagged if:
- Justification is vague or false
- Exception is used for convenience, not necessity
- Requester bypasses exception process (manual Portal change without approval)
- Exception is renewed >3 times without workload re-architecture

### Consequences

**First Offense**: Warning and training on exception process.

**Second Offense**: Temporary revocation of privileged access (PIM, Contributor RBAC).

**Third Offense**: Escalation to HR and management. Potential disciplinary action.

**Why Strict**: Exception process is a safety valve, not a loophole. Abuse undermines platform security.

## Known Issues with Exception Process

### 1. Approval Delays

**Problem**: Approver does not respond within SLA. Requester blocked.

**Mitigation**: Escalation path defined. If approver does not respond within SLA, request escalates to their manager.

### 2. Expiry Not Enforced

**Problem**: Automation fails to remove expired exception.

**Detection**: Weekly audit scans for expired exceptions.

**Remediation**: Manual removal by platform team. Automation fixed.

### 3. Exception Request Bypassed

**Problem**: Requester modifies resource directly without exception request.

**Detection**: Azure Activity Logs and Terraform drift detection flag unauthorised changes.

**Remediation**: Change is reverted. Requester receives warning.

### 4. Justification is Weak

**Problem**: Approver grants exception based on insufficient justification.

**Detection**: Monthly audit of exception justifications.

**Remediation**: Approver receives training. Future exceptions from same approver reviewed more strictly.

## Exception Process vs Break-Glass

**Exception Process**: Planned, approved deviations from policy for specific resources and timeframes.

**Break-Glass**: Emergency access bypassing all controls during outages or incidents.

**Key Differences**:
- Exception process has approval workflow and justification
- Break-glass bypasses approval (immediate access, audited post-incident)
- Exceptions are time-boxed (30 days typical)
- Break-glass is used once and reviewed immediately

**When to Use**:
- **Exception Process**: Non-urgent need for policy deviation
- **Break-Glass**: Immediate access required to prevent or resolve outage

## Exception Philosophy

Exceptions are a **safety valve**, not a failure.

Overly rigid controls that permit no exceptions create:
- Shadow IT (delivery teams bypass platform entirely)
- Manual workarounds (Portal changes without approval)
- Delivery delays (legitimate work blocked indefinitely)

**Acceptable Exception Rate**: 5-10% of policy violations result in approved exceptions.

**Too Many Exceptions (>10%)**: Policy is misaligned with business needs. Policy should be adjusted.

**Too Few Exceptions (<1%)**: Policy may be too permissive or delivery teams are bypassing process.

## Who Owns the Risk

**During Exception Period**: Requester owns the risk. Security incident or audit finding is requester's responsibility.

**After Exception Expiry**: Platform team owns the risk. If exception is not removed, platform team is responsible.

**Example**:
- Exception granted for public IP on VM
- VM compromised during exception period
- Requester is accountable (they acknowledged risk)
- After expiry, if public IP is not removed, platform team is accountable (failed to enforce expiry)

Exceptions shift risk, they do not eliminate it.