# Exception Handling

## Temporary Elevation

Exceptions to platform controls are inevitable but must be time-boxed and risk-accepted.

This document defines when exceptions are granted, who owns the risk, and how exceptions expire.

## Types of Exceptions

### 1. Temporary RBAC Elevation

**Scenario**: A user needs privileged access beyond their normal role for a limited time.

**Examples**:
- Incident response requires Contributor access to production subscription
- Third-party auditor needs Reader access across all subscriptions for 2 weeks
- Delivery team needs temporary Network Contributor to configure VNet peering

**Process**:
1. User submits exception request via ServiceNow (or equivalent ticketing system)
2. Request includes justification, scope (subscription or resource group), duration (maximum 30 days), and approver
3. Approver reviews and approves or rejects
4. If approved, RBAC assignment is created with expiry date
5. Upon expiry, assignment is automatically removed

**Approver**: Platform team for Tier 1 access. Security team for Tier 0 access.

**Audit**: All temporary elevations are logged and reviewed monthly.

### 2. Azure Policy Exemption

**Scenario**: A workload cannot comply with an Azure Policy due to technical limitations or business requirements.

**Examples**:
- Legacy application requires public IP on VM (policy blocks this by default)
- Regulatory requirement prohibits encryption-at-rest for specific dataset (policy enforces encryption)
- Third-party SaaS integration requires storage account firewall to be disabled (policy enforces firewall)

**Process**:
1. Delivery team submits policy exemption request
2. Request includes policy name, resource, justification, and duration
3. Security team reviews risk and approves or rejects
4. If approved, exemption is created in Terraform with expiry date
5. Upon expiry, exemption is removed and resource must comply or be deleted

**Approver**: Security team.

**Audit**: All policy exemptions are reviewed quarterly. Non-compliant resources flagged for remediation or re-approval.

### 3. Conditional Access Bypass

**Scenario**: A user or application cannot meet Conditional Access requirements.

**Examples**:
- External partner needs access but does not have a compliant device
- Service account requires authentication but cannot use MFA
- Emergency access during network outage where compliant devices are unavailable

**Process**:
1. User or application owner submits Conditional Access exception request
2. Request includes user/service principal, policy name, justification, and duration (maximum 14 days)
3. Security team reviews and approves or rejects
4. If approved, user is added to policy exclusion group with expiry date
5. Upon expiry, exclusion is removed

**Approver**: Security team. No delegation allowed.

**Audit**: All Conditional Access exclusions are reviewed weekly. Expired exclusions are removed automatically.

### 4. Defender for Cloud Exemption

**Scenario**: A Defender for Cloud recommendation or alert is a false positive or cannot be remediated.

**Examples**:
- VM does not have endpoint protection (VM is a legacy system pending decommissioning)
- Storage account has public network access (required for external partner integration)
- SQL database has transparent data encryption disabled (regulatory requirement)

**Process**:
1. Resource owner submits Defender exemption request
2. Request includes recommendation ID, resource, justification, and duration
3. Security team reviews and approves or rejects
4. If approved, exemption is created in Defender for Cloud
5. Upon expiry, recommendation resurfaces and must be remediated or re-approved

**Approver**: Security team.

**Audit**: All Defender exemptions reviewed monthly.

## Exception Request Template

All exception requests must include:

| Field           | Description                                                                 |
|-----------------|-----------------------------------------------------------------------------|
| Requester       | Name and email of person requesting exception                               |
| Resource        | Subscription, resource group, or resource ID affected                       |
| Exception Type  | RBAC elevation, policy exemption, Conditional Access bypass, Defender exemption |
| Justification   | Why exception is required (business need, technical limitation, regulatory) |
| Duration        | How long exception is needed (maximum 30 days for most exceptions)          |
| Approver        | Who is authorised to approve this exception                                 |
| Risk Acceptance | Acknowledgement that requester owns the risk during exception period        |

## Exception Approval Criteria

### When to Approve

Exceptions are approved if:

1. **Business need is documented**: Vague justifications are rejected.
2. **Alternative solutions have been explored**: Exception is last resort, not first choice.
3. **Duration is time-boxed**: Open-ended exceptions are not permitted.
4. **Risk is understood**: Requester acknowledges what control is being bypassed and why.

### When to Reject

Exceptions are rejected if:

1. **Workaround exists**: Exception is requested for convenience, not necessity.
2. **Control can be implemented**: Exception is requested due to lack of effort, not technical impossibility.
3. **Justification is weak**: "We've always done it this way" is not sufficient.
4. **Risk is unacceptable**: Exception would violate compliance requirements or create excessive blast radius.

## Exception Expiry and Renewal

### Automatic Expiry

All exceptions expire automatically after the approved duration.

**No grace period.** Upon expiry:
- RBAC assignments are removed
- Azure Policy exemptions are deleted
- Conditional Access exclusions are removed
- Defender for Cloud exemptions resurface as active recommendations

**Why**: Manual expiry processes are unreliable. Automation is mandatory.

### Renewal Process

If an exception is still required after expiry, a **new exception request** must be submitted.

Renewals are **not automatic**. Each renewal requires:

1. Updated justification explaining why exception is still required
2. Evidence that alternative solutions were re-evaluated
3. Fresh approval from the same or different approver

**Why**: Exceptions should be increasingly rare. Repeated renewals indicate systemic issues that must be addressed.

### Exception Escalation

If an exception is renewed more than **three times**, it escalates to executive leadership for review.

**Why**: Repeated exceptions indicate that platform controls are misaligned with business needs. Either the control should be adjusted, or the workload should be re-architected.

## Audit Expectations

### Monthly Exception Report

Platform team generates a report of all active exceptions including:

- Exception type and resource
- Requester and approver
- Duration and expiry date
- Number of times exception has been renewed

Report is reviewed by security team and platform lead.

### Quarterly Exception Deep Dive

Security team and platform team meet quarterly to review:

- Exceptions with multiple renewals (candidates for permanent policy changes or workload re-architecture)
- Exceptions that expired without renewal (validation that control was temporary)
- Patterns in exception requests (indication of systemic policy misalignment)

### Annual Exception Audit

External or internal audit reviews all exceptions granted in the past year.

Audit validates:

- Exception approval process was followed
- Risk acceptance was documented
- Exceptions expired as intended
- No unauthorised bypasses occurred

## Known Failure Modes

### 1. Exception Expiry Not Enforced

**Scenario**: Automatic expiry fails due to misconfiguration or tooling bug.

**Impact**: Exceptions persist indefinitely, undermining controls.

**Mitigation**: Weekly automated scan for expired exceptions. Manual remediation if automation fails.

### 2. Exception Request Bypassed

**Scenario**: User modifies Azure Policy or RBAC directly without exception request.

**Impact**: Unauthorised bypasses create security gaps.

**Mitigation**: Azure Policy blocks unauthorised exemptions. Audit logs flag unexpected changes. Offending user loses privileged access.

### 3. Exception Justification is Insufficient

**Scenario**: Approver grants exception based on weak or false justification.

**Impact**: Controls are bypassed unnecessarily.

**Mitigation**: Monthly audit of exception justifications. Approvers are trained on approval criteria.

### 4. Exception Renewal Becomes Default

**Scenario**: Exceptions are renewed automatically without re-evaluation.

**Impact**: Temporary exceptions become permanent, undermining platform philosophy.

**Mitigation**: Escalation to executive leadership after three renewals. Forced re-architecture or policy change.

## Exceptions are Not Failures

Exceptions are a safety valve.

Overly rigid controls that permit no exceptions create shadow IT, manual bypasses, and delivery delays.

Exceptions are acceptable **if**:

- They are time-boxed
- Risk is explicitly accepted
- Justification is documented
- Expiry is enforced

What is **not acceptable**:

- Permanent exceptions
- Silent bypasses
- Exceptions granted for convenience
- Exceptions without audit trail

Exception handling is a balance between security and operational reality. This process ensures that balance is maintained.