# Time-Boxed Exceptions

## Expiry-Driven Governance

All exceptions must have an expiry date.

Open-ended exceptions are not permitted. They become permanent bypasses of security controls.

## Why Time-Boxing is Mandatory

### 1. Temporary Problems Require Temporary Solutions

Most exceptions are requested due to temporary constraints:
- Vendor limitation (fix in progress)
- Migration in progress (legacy app being retired)
- Emergency workaround (permanent fix being developed)

**If the problem is temporary, the exception must be temporary.**

### 2. Exceptions Become Forgotten

Without expiry, exceptions accumulate indefinitely.

**Example**:
- 2022: Exception granted for legacy app requiring public IP
- 2023: App migrated to private endpoint but exception not removed
- 2024: Exception still active despite app no longer existing
- 2026: Audit discovers 50+ forgotten exceptions

**Time-boxing forces re-evaluation.** If exception is still needed, renewal is required with fresh justification.

### 3. Risk Changes Over Time

Today's acceptable risk may be unacceptable tomorrow.

**Example**:
- 2024: Weak TLS version accepted for third-party integration (vendor promised fix)
- 2026: TLS vulnerability exploited in the wild (CVE published)
- Risk appetite changes: Weak TLS no longer acceptable

**Time-boxing allows risk re-assessment.** Renewal requires re-approval under current risk posture.

## Maximum Exception Durations

| Exception Type                  | Maximum Duration | Rationale                                      |
|---------------------------------|------------------|------------------------------------------------|
| Azure Policy (Production)       | 30 days          | Security controls must not be bypassed long    |
| Azure Policy (Non-Production)   | 90 days          | Lower risk; longer duration acceptable         |
| RBAC Elevation (Tier 1)         | 30 days          | Privileged access must be time-limited         |
| RBAC Elevation (Tier 0)         | 14 days          | Highest privilege; shortest duration           |
| Conditional Access Bypass       | 14 days          | Identity controls critical                     |
| Defender for Cloud Exemption    | 30 days          | Detective controls; longer duration acceptable |
| Private Endpoint Exemption      | 30 days          | Network exposure risk must be minimised        |
| Firewall Rule (Temporary)       | 14 days          | Network access must be justified continuously  |

**Why Different Durations**: Risk varies by control type. Higher risk = shorter duration.

## Automatic Expiry Mechanisms

Expiry must be **automatic**, not manual.

**Why**: Manual expiry relies on humans remembering to remove exceptions. Humans forget.

### Azure Policy Exemptions

Azure Policy exemptions have native expiry support.

**Implementation**:
```hcl
resource "azurerm_policy_exemption" "example" {
  name                 = "exception-ecommerce-publicip"
  policy_assignment_id = azurerm_policy_assignment.block_public_ips.id
  exemption_category   = "Waiver"
  expires_on           = "2026-03-07T00:00:00Z"  # ISO 8601 format
  metadata = jsonencode({
    ticket_id  = "INC123456"
    requester  = "john.doe@company.com"
    expiry_notified = false
  })
}
```

**Behaviour**: Azure automatically removes exemption on expiry date. No manual intervention required.

### RBAC Assignments

Azure RBAC assignments do **not** support native expiry (as of 2026).

**Workaround**: Weekly automation scans for expired RBAC exceptions and removes them.

**Implementation (PowerShell)**:
```powershell
# Run weekly via Azure Automation
$exceptions = Get-Content -Path "exceptions.json" | ConvertFrom-Json
foreach ($exception in $exceptions) {
    if ($exception.expiry -lt (Get-Date)) {
        Remove-AzRoleAssignment -ObjectId $exception.principalId -Scope $exception.scope
        Send-MailMessage -To $exception.requester -Subject "Exception Expired" -Body "Your RBAC exception has expired and been removed."
    }
}
```

**Why Not PIM**: PIM has native expiry but is limited to specific roles. Not all Contributor access goes through PIM.

### Conditional Access Exclusions

Entra ID Conditional Access policies do **not** support native expiry for group-based exclusions.

**Workaround**: Weekly automation removes users from exclusion groups after expiry.

**Implementation (PowerShell)**:
```powershell
# Run weekly via Azure Automation
$exclusionGroup = Get-AzureADGroup -SearchString "ca-exclusion-temp"
$members = Get-AzureADGroupMember -ObjectId $exclusionGroup.ObjectId

foreach ($member in $members) {
    $exception = Get-ExceptionRecord -UserId $member.ObjectId
    if ($exception.expiry -lt (Get-Date)) {
        Remove-AzureADGroupMember -ObjectId $exclusionGroup.ObjectId -MemberId $member.ObjectId
        Send-MailMessage -To $exception.requester -Subject "Conditional Access Exception Expired"
    }
}
```

### Defender for Cloud Exemptions

Defender exemptions do **not** support native expiry (as of 2026).

**Workaround**: Monthly manual review of all Defender exemptions. Platform team removes expired exemptions.

**Why Manual**: Defender API does not expose exemption metadata (no place to store expiry date programmatically).

**Improvement Request**: Submitted to Microsoft. Awaiting feature support.

## Exception Expiry Notifications

Users are notified before exception expires.

**Notification Schedule**:
- **7 days before expiry**: Email reminder with renewal instructions
- **1 day before expiry**: Email reminder with urgent notice
- **On expiry**: Email confirmation that exception has been removed

**Email Template (7 Days Before)**:
```
Subject: Exception Expiring Soon - INC123456

Your exception will expire in 7 days.

Resource: /subscriptions/abc123/resourceGroups/rg-ecommerce-prod
Control: Azure Policy - Block Public IP on VMs
Expiry: 2026-03-07 00:00:00 UTC

If you still require this exception:
1. Submit a new ServiceNow request
2. Provide updated justification
3. Request will be reviewed and approved/rejected

If you do not submit a renewal request, the exception will be automatically removed on expiry.

Questions? Contact platform-team@company.com
```

## Renewal Limits

Exceptions can be renewed but with escalating scrutiny.

| Renewal Count | Approver Requirement                     | Action Required                          |
|---------------|------------------------------------------|------------------------------------------|
| First         | Same approver as initial request         | Updated justification                    |
| Second        | Same approver + platform lead review     | Evidence of migration plan               |
| Third         | Security team + executive sponsor review | Business case for permanent exception    |
| Fourth+       | Executive leadership + risk committee    | Risk acceptance memo signed by CISO      |

**Why Escalating Approvals**: Repeated renewals indicate systemic issue. Escalation ensures problem is addressed, not deferred indefinitely.

## Permanent Exceptions (Rare)

Some exceptions are effectively permanent due to business or technical constraints.

**Examples**:
- Legacy application cannot be migrated (vendor out of business)
- Regulatory requirement conflicts with platform policy (rare)
- Third-party SaaS integration requires specific configuration

**Process for Permanent Exception**:
1. Exception renewed ≥3 times
2. Business case prepared explaining why permanent exception is necessary
3. Risk acceptance memo signed by CISO and executive sponsor
4. Exception documented in platform architecture repository (this repo) with justification
5. Exception reviewed annually

**Outcome**: Exception moves from time-boxed to permanent. No longer requires renewal.

**Audit Requirement**: Permanent exceptions audited annually. If circumstances change, exception is revoked.

## Exception Expiry Failures

### 1. Automation Fails to Remove Exception

**Scenario**: Weekly automation script fails due to bug or Azure API outage.

**Detection**: Weekly reconciliation report compares active exceptions to exception database. Flags exceptions past expiry date.

**Remediation**: Manual removal by platform team. Automation fixed.

**Frequency**: <1% of exceptions.

### 2. User Circumvents Expiry

**Scenario**: User re-applies exception manually after automation removes it.

**Example**:
- RBAC assignment removed on expiry
- User with Owner access re-assigns RBAC to themselves
- Bypasses exception process

**Detection**: Azure Activity Logs flag unauthorised RBAC changes. Alert sent to security team.

**Remediation**: RBAC assignment revoked. User receives warning. Repeat offense results in loss of privileged access.

### 3. Notification Not Sent

**Scenario**: Expiry notification email fails to send due to SMTP issue.

**Impact**: User unaware exception is expiring. Surprise when access is revoked.

**Detection**: Email delivery monitoring. Alert if notification send fails.

**Remediation**: Retry email send. If still fails, manual notification via Teams or phone.

### 4. Exception Renewed Indefinitely

**Scenario**: Approver grants renewals without scrutiny. Exception renewed 10+ times.

**Impact**: Exception becomes de facto permanent. Risk not re-evaluated.

**Detection**: Monthly audit flags exceptions with >3 renewals.

**Remediation**: Escalation to executive leadership. Forced decision: Permanent exception (with risk acceptance) or workload re-architecture.

## Time-Boxing Best Practices

### 1. Start with Shortest Duration

Request shortest duration necessary. Extend if needed via renewal.

**Why**: Easier to extend than to revoke early. Short duration forces frequent re-evaluation.

### 2. Plan for Expiry

When requesting exception, plan for permanent fix.

**Example**:
- Request 30-day exception for public IP
- Use 30 days to implement private endpoint migration
- Exception expires naturally when migration completes

**Anti-Pattern**: Request exception with no migration plan. Renewal becomes routine.

### 3. Document Expiry Actions

When exception expires, document what happens.

**Examples**:
- Public IP removed → Workload switches to private endpoint
- RBAC revoked → User loses access to subscription
- Firewall rule removed → External integration breaks (acceptable, migration complete)

**Why**: Ensures expiry is intentional, not accidental.

## Time-Boxed Exceptions Summary

- **All exceptions have expiry date**: No open-ended exceptions
- **Expiry is automatic**: Automation removes exceptions, not humans
- **Notifications before expiry**: 7 days, 1 day, and on expiry
- **Renewal requires fresh justification**: Each renewal is a new approval
- **Escalating scrutiny for repeated renewals**: ≥3 renewals escalate to executive leadership
- **Permanent exceptions are rare**: Require CISO approval and annual review

**Time-boxing is not bureaucracy. It is governance.**

Exceptions without expiry become permanent. Permanent exceptions become policy. Policy bypasses undermine platform security.