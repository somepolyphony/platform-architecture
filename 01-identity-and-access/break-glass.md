# Break-Glass Accounts

## Purpose

Break-glass accounts provide **emergency access** to Entra ID and Azure when normal authentication mechanisms fail.

They are the last-resort recovery path when:

- Entra ID Conditional Access policies block all users
- PIM approval workflow is unavailable
- Identity Protection flags all admin accounts as compromised
- Entra ID itself is experiencing an outage

Break-glass accounts bypass all controls and must be treated as **nuclear options**.

## When to Use Break-Glass

### Legitimate Use Cases

1. **Conditional Access Lockout**

   A misconfigured Conditional Access policy blocks all users, including Global Administrators.

   **Example**: A policy requiring device compliance is applied to all users, but no break-glass exclusion exists. All admin devices are incorrectly flagged as non-compliant.

2. **Entra ID Service Outage**

   Entra ID is partially or fully unavailable. Authentication fails for all users.

   **Example**: Regional Entra ID failure prevents sign-in. Break-glass accounts may have cached credentials or use federated authentication as a fallback (if configured).

3. **PIM Workflow Failure**

   Privileged Identity Management is unavailable or approvers cannot respond.

   **Example**: Critical incident at 2 AM. PIM approvers are offline. Platform engineer needs immediate Global Administrator access to prevent data loss.

4. **Identity Protection False Positive**

   Identity Protection flags all admin accounts as compromised and blocks sign-in.

   **Example**: A misconfigured risk policy auto-blocks high-risk users. All admin accounts are flagged due to anomalous location detection.

### Invalid Use Cases

Break-glass accounts must **never** be used for:

- Routine administrative tasks
- Bypassing approval workflows for convenience
- Testing or troubleshooting
- Granting access when PIM is functioning normally
- Avoiding PIM activation latency during non-critical incidents

**Consequences of Misuse**: Immediate suspension of the user who activated break-glass. Security review of all actions taken. Potential disciplinary action.

## Break-Glass Account Design

### Number of Accounts

Two break-glass accounts exist:

- `breakglass-01@domain.onmicrosoft.com`
- `breakglass-02@domain.onmicrosoft.com`

**Why Two**: Redundancy. If one account is locked, blocked, or compromised, the second remains available.

### Permissions

Both accounts have:

- Permanent Global Administrator role in Entra ID
- Owner at Tenant Root Management Group (Azure)

These are the only identities with permanent Tier 0 access.

### Authentication Method

Break-glass accounts use **long, randomly generated passwords** stored offline.

**Why Not MFA**: If Entra ID is unavailable, MFA enforcement may fail. Password-only authentication is the fallback.

**Why Not FIDO2**: Physical security keys can be lost or unavailable during emergencies.

**Password Storage**:
- Printed on paper and stored in a physical safe
- Encrypted digital copy stored in an offline password manager (not in Azure Key Vault)
- Split knowledge: One executive holds half the password, another holds the other half (optional, depending on organisational paranoia)

### Conditional Access Exclusion

Break-glass accounts are **excluded** from all Conditional Access policies.

This includes:

- Device compliance requirements
- MFA requirements
- Location-based restrictions
- Risk-based sign-in blocks

**Why**: If Conditional Access is the cause of lockout, break-glass must bypass it entirely.

### Identity Protection Exclusion

Break-glass accounts are **excluded** from Identity Protection risk policies.

**Why**: If Identity Protection flags all admins as compromised, break-glass must remain functional.

## Risks and Controls

### Risk 1: Credential Theft

Break-glass passwords are high-value targets. If stolen, an attacker has permanent Global Administrator access.

**Mitigations**:
- Passwords are stored offline (not in cloud Key Vaults or password managers)
- Physical access to password safe is logged and restricted
- Break-glass usage generates immediate alerts to security team
- All actions taken by break-glass accounts are audited and reviewed

**Residual Risk**: Physical security is not perfect. Insider threats or coercion remain risks.

### Risk 2: Misuse

Break-glass accounts bypass all controls. A malicious or negligent user can:

- Delete subscriptions
- Modify Conditional Access policies
- Exfiltrate data
- Grant themselves permanent privileges

**Mitigations**:
- Break-glass usage requires dual-person authorisation (two executives must approve)
- All break-glass sign-ins generate real-time alerts to security team and executive leadership
- Post-incident audit of all actions taken
- User who activated break-glass must justify every action

**Residual Risk**: Dual-person authorisation can be bypassed by collusion or coercion.

### Risk 3: Account Lockout

Break-glass accounts can be locked due to:

- Too many failed password attempts
- Administrative error (accidental disabling of account)
- Entra ID bug or misconfiguration

**Mitigations**:
- Two break-glass accounts for redundancy
- Account lockout thresholds set to maximum (or disabled if possible)
- Quarterly validation that break-glass accounts are functional

**Residual Risk**: Both accounts can be locked simultaneously. No perfect mitigation exists.

### Risk 4: Password Expiry

Entra ID password expiry policies can force break-glass password changes.

**Mitigations**:
- Break-glass accounts have password expiry disabled
- Azure Policy enforces that password expiry cannot be re-enabled on break-glass accounts

**Residual Risk**: An administrative error could re-enable expiry. Quarterly validation catches this.

## Monitoring and Alerting

### Real-Time Alerts

Any sign-in by a break-glass account generates:

1. **Email alert** to security team and executive leadership
2. **SMS alert** to on-call incident responder
3. **Microsoft Teams alert** to security operations channel
4. **SIEM alert** flagged as critical priority

### Alert Content

Alerts must include:

- Which break-glass account was used
- Source IP address and location
- Device details (if available)
- Timestamp of sign-in
- Actions taken (via Entra ID audit logs)

### Post-Incident Review

After every break-glass usage:

1. Security team reviews all actions taken by the break-glass account
2. User who activated break-glass provides written justification
3. Incident is logged with root cause analysis
4. Break-glass password is rotated
5. Executive approval is documented

**Timeline**: Post-incident review must be completed within 24 hours.

## Break-Glass Testing

Break-glass accounts must be tested **quarterly** to ensure functionality.

### Test Procedure

1. Scheduled test announced to security team (not a surprise drill)
2. Authorised user retrieves break-glass password from physical safe
3. User signs in to Entra ID using break-glass account
4. User performs a low-risk action (e.g., reads a document, lists subscriptions)
5. User signs out
6. Security team validates alerts were generated
7. Break-glass password is rotated post-test

**Why Test**: Untested break-glass accounts are assumed non-functional. Testing validates both technical functionality and operational procedures.

### Test Failures

If break-glass test fails:

- Incident is escalated to platform lead and security architect
- Root cause analysis is performed within 48 hours
- Remediation is prioritised above all other work
- Re-test occurs within 1 week

**Examples of Failures**:
- Account is locked or disabled
- Password does not work
- Conditional Access policy is incorrectly applied
- Alerts do not fire

## Alternatives to Break-Glass

Break-glass is a last resort. Other recovery mechanisms should be attempted first:

### 1. PIM Activation with Alternative Approver

If primary PIM approver is unavailable, secondary approver can approve.

### 2. Conditional Access Report-Only Mode

Misconfigured policies in report-only mode do not block users. If testing was done properly, lockouts should be rare.

### 3. Azure Support Ticket

Microsoft Support can assist with Entra ID lockouts in some cases (not guaranteed, not fast).

### 4. Federated Authentication Fallback

If Entra ID is federated with on-premises Active Directory, on-premises credentials may work as a fallback (depends on configuration).

**Reality**: These alternatives are slow or unreliable. Break-glass is necessary.

## Break-Glass in Multi-Tenant Scenarios

Organisations with multiple Entra ID tenants (e.g., dev, staging, production) must have break-glass accounts in **each tenant**.

**Why**: Break-glass in one tenant does not grant access to other tenants.

**Operational Burden**: More accounts to manage, test, and secure. This is unavoidable.

## Known Failure Modes

### 1. Entra ID Global Outage

**Scenario**: Entra ID is completely offline. Break-glass accounts cannot authenticate.

**Impact**: No recovery path exists. All Azure and Microsoft 365 services are inaccessible.

**Mitigation**: None. This is an accepted risk. Microsoft is responsible for Entra ID availability.

### 2. Break-Glass Password Lost

**Scenario**: Physical safe is inaccessible (e.g., office evacuation, lost key).

**Impact**: Break-glass accounts cannot be used.

**Mitigation**: Passwords are stored in multiple physical locations (e.g., primary office safe, offsite backup safe). Digital encrypted backup exists (not in cloud).

### 3. Break-Glass Account Compromised

**Scenario**: Attacker obtains break-glass password and uses it maliciously.

**Impact**: Full tenant compromise. Attacker has Global Administrator access.

**Mitigation**: Real-time alerts enable rapid response. Password rotation and account review occur immediately. Forensic analysis determines extent of compromise.

**Recovery**: Assume full tenant compromise. Rotate all credentials, review all RBAC assignments, audit all Conditional Access changes.

Break-glass accounts are a necessary evil. They must be secured, monitored, and tested rigorously.