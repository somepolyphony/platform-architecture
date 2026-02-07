# RBAC and Separation

## Separation of Duties

No single person should have end-to-end control over platform or workloads.

Separation of duties prevents insider threats, errors, and unauthorised changes.

## Why Separation Matters

### 1. Prevents Single Point of Compromise

If one identity has both deployment and approval authority, a compromised account can bypass all controls.

**Example**:
- User has Contributor (can deploy resources)
- User also has Owner (can modify RBAC)
- Compromised account can grant itself additional privileges and bypass policy

**Mitigation**: Separate deployment from RBAC management. Contributor cannot modify RBAC.

### 2. Enforces Peer Review

Changes require multiple people: one to propose, one to approve.

**Example**:
- Engineer writes Terraform code (proposal)
- Platform lead reviews and approves (approval)
- CI/CD pipeline deploys (automation)

No single engineer can deploy without review.

### 3. Reduces Accidental Damage

Humans make mistakes. Separation creates checkpoints that catch errors.

**Example**:
- Engineer accidentally deletes production resource group in Terraform
- Peer reviewer catches error during PR review
- Deployment blocked

Without peer review, error reaches production.

## RBAC Separation Model

### Development/Staging Subscriptions

| Role                | Platform Team | Delivery Team | Security Team |
|---------------------|---------------|---------------|---------------|
| Reader              | ✅            | ✅            | ✅            |
| Contributor         | ✅            | ✅            | ❌            |
| Owner               | ✅ (PIM)      | ❌            | ❌            |
| Defender Management | ❌            | ❌            | ✅            |

**Why**:
- Delivery teams need Contributor to deploy workloads
- Platform team needs Owner (via PIM) for RBAC and policy changes
- Security team has read-only access plus Defender management

### Production Subscriptions

| Role                | Platform Team | Delivery Team | Security Team |
|---------------------|---------------|---------------|---------------|
| Reader              | ✅            | ✅            | ✅            |
| Contributor         | ✅ (PIM)      | ❌            | ❌            |
| Owner               | ✅ (PIM)      | ❌            | ❌            |
| Defender Management | ❌            | ❌            | ✅            |

**Why**:
- Production changes via CI/CD only
- Manual Contributor access via PIM for emergencies
- Delivery teams have Reader access (view workloads, debug issues)

### Platform Subscriptions (Hub Network, Logging)

| Role                | Platform Team | Delivery Team | Security Team |
|---------------------|---------------|---------------|---------------|
| Reader              | ✅            | ❌            | ✅            |
| Contributor         | ✅ (PIM)      | ❌            | ❌            |
| Owner               | ✅ (PIM)      | ❌            | ❌            |
| Network Contributor | ✅ (PIM)      | ❌            | ❌            |

**Why**:
- Platform subscriptions are Tier 0 assets
- Delivery teams have no access (prevents accidental or malicious changes)
- All changes via Terraform with peer review

## Separation of Duties by Function

### Terraform Deployment

| Function               | Who Performs                     | Approval Required |
|------------------------|----------------------------------|-------------------|
| Write Terraform code   | Platform engineer                | No                |
| Submit PR              | Platform engineer                | No                |
| Review PR              | Different platform engineer      | Yes               |
| Approve PR             | Platform lead                    | Yes               |
| Merge PR               | Platform engineer (after approval) | No              |
| Run terraform apply    | CI/CD pipeline (service principal) | Manual gate     |

**Key Points**:
- Author cannot approve their own PR
- At least two people involved (author + reviewer)
- CI/CD pipeline uses service principal (not human identity)

### Azure Policy Assignment

| Function               | Who Performs                     | Approval Required |
|------------------------|----------------------------------|-------------------|
| Define policy          | Platform engineer                | No                |
| Submit PR              | Platform engineer                | No                |
| Review PR              | Security team + platform engineer | Yes              |
| Approve PR             | Security team lead               | Yes               |
| Deploy policy          | CI/CD pipeline                   | Manual gate       |

**Key Points**:
- Security team must approve policy changes (they own risk)
- Platform engineer reviews technical correctness
- Both approvals required before deployment

### RBAC Assignment

| Function               | Who Performs                     | Approval Required |
|------------------------|----------------------------------|-------------------|
| Request RBAC change    | Delivery team or platform engineer | No              |
| Review request         | Platform lead                    | Yes               |
| Approve request        | Security team (if Tier 0 or Tier 1) | Yes            |
| Implement change       | Platform engineer                | No                |
| Audit change           | Security team (quarterly)        | No                |

**Key Points**:
- Requester cannot approve their own RBAC request
- Tier 0 and Tier 1 RBAC requires security team approval
- Regular audits catch unauthorised RBAC assignments

### Incident Response (Production)

| Function               | Who Performs                     | Approval Required |
|------------------------|----------------------------------|-------------------|
| Detect incident        | Monitoring/Defender alerts       | No                |
| Triage incident        | On-call engineer (delivery team) | No                |
| Request elevated access | On-call engineer                | Yes (if PIM required) |
| Approve elevated access | Platform lead or security team  | Yes               |
| Implement fix          | On-call engineer                 | No (emergency)    |
| Codify fix in Terraform | On-call engineer (post-incident) | Yes (PR review)   |

**Key Points**:
- Emergency access via PIM (approval workflow)
- Incident fixes may bypass normal review during active incident
- All emergency changes must be codified in Terraform post-incident

## Separation Failures (Anti-Patterns)

### 1. Owner at Subscription Scope for Daily Work

**Problem**: User has standing Owner access for convenience.

**Risk**: Can modify RBAC, delete resources, bypass policy.

**Correct Approach**: User has Contributor for daily work. Owner via PIM only when needed.

### 2. Service Principal with Excessive Permissions

**Problem**: CI/CD service principal has Owner at management group scope.

**Risk**: Compromise of service principal credentials = full tenant compromise.

**Correct Approach**: Service principal scoped to specific subscriptions or resource groups.

### 3. Shared Accounts

**Problem**: Multiple engineers share same admin account.

**Risk**: Audit trail is ambiguous (who made the change?). Credential rotation is difficult.

**Correct Approach**: Each engineer has individual identity. No shared accounts.

### 4. Break-Glass Account Used for Routine Work

**Problem**: Engineer uses break-glass account to bypass PIM approval.

**Risk**: Break-glass alerts ignored as "routine". Real emergencies not detected.

**Correct Approach**: Break-glass accounts are for emergencies only. All routine work via PIM.

### 5. Self-Approval

**Problem**: Engineer submits PR and approves it themselves.

**Risk**: No peer review. Errors and malicious changes not caught.

**Correct Approach**: Author cannot approve their own PR. At least one peer reviewer required.

## Separation of Duties Checklist

- [ ] No single user has both Contributor and Owner on same subscription
- [ ] All Terraform changes require PR approval by different person
- [ ] Azure Policy changes require security team approval
- [ ] RBAC changes (Tier 0, Tier 1) require security team approval
- [ ] Service principals scoped to resource groups or subscriptions, not management groups
- [ ] Break-glass accounts excluded from routine work
- [ ] Shared accounts prohibited (each engineer has individual identity)
- [ ] Quarterly RBAC audit to detect permission creep

## Role Separation by Team

### Platform Team Internal Separation

Even within platform team, separation exists.

| Role                  | Junior Engineer | Senior Engineer | Platform Lead |
|-----------------------|-----------------|-----------------|---------------|
| Write Terraform       | ✅              | ✅              | ✅            |
| Approve PR            | ❌              | ✅              | ✅            |
| Deploy to production  | ❌              | ✅ (approved PR) | ✅           |
| RBAC at mgmt group    | ❌              | ❌              | ✅ (PIM)      |
| Policy assignment     | ❌              | ✅              | ✅            |

**Why**: Junior engineers cannot deploy to production without senior review. Platform lead has final authority.

### Security Team Internal Separation

| Role                         | Security Analyst | Security Engineer | Security Lead |
|------------------------------|------------------|-------------------|---------------|
| Review Defender alerts       | ✅               | ✅                | ✅            |
| Approve policy exemptions    | ❌               | ✅                | ✅            |
| Approve Conditional Access   | ❌               | ❌                | ✅            |
| Approve break-glass usage    | ❌               | ❌                | ✅            |

**Why**: High-risk approvals escalate to security lead. Analyst role is detective, not approver.

## Separation in Automation

Automation (CI/CD pipelines) also follows separation of duties.

### Service Principal Permissions

CI/CD service principals have:
- Contributor on specific subscriptions (not Owner)
- Cannot modify RBAC
- Cannot create policy assignments
- Cannot access Key Vault secrets directly (uses managed identity where possible)

**Why**: Compromised service principal is contained to deployment actions, not security controls.

### Manual Approval Gates

CI/CD pipelines include manual approval gates for high-risk actions:

1. **Terraform plan review**: Human reviews plan output before apply
2. **Production deployment**: Platform lead approves before deployment
3. **Policy changes**: Security team approves before deployment

**Why**: Automation is fast but not infallible. Human checkpoint catches errors.

## Monitoring Separation of Duties

### 1. Quarterly RBAC Audit

Security team reviews all RBAC assignments:
- Users with Owner at subscription or management group scope
- Service principals with broad permissions
- RBAC assignments that bypass PIM

**Action Items**: Remove excessive permissions. Enforce PIM where appropriate.

### 2. Break-Glass Usage Audit

Every break-glass usage reviewed:
- Who activated break-glass
- Why (incident details)
- What actions taken
- Was it justified (emergency) or misused (convenience)

**Action Items**: Misuse results in disciplinary action.

### 3. Terraform PR Review Audit

Random sample of Terraform PRs audited quarterly:
- Was PR reviewed by different person than author?
- Was security team approval obtained for policy/RBAC changes?
- Were manual approval gates used for production?

**Action Items**: Process violations result in re-training.

## Known Challenges

### 1. Operational Friction

Separation adds delay. Engineers must wait for approvals.

**Trade-Off**: Security vs speed. Security wins for production.

**Mitigation**: Clear SLAs for approvals (4 hours for urgent, 1 day for non-urgent).

### 2. Approval Bottleneck

Single approver becomes bottleneck if unavailable.

**Mitigation**: Multiple approvers per role. Approver unavailability does not block work.

### 3. Emergency Bypass Pressure

During incidents, pressure to bypass separation of duties.

**Mitigation**: Break-glass process provides emergency access. Post-incident review ensures compliance.

## Separation of Duties Summary

- **No single point of control**: Deployment and approval separated
- **Peer review mandatory**: PRs require different approver than author
- **Service principals scoped narrowly**: Contributor, not Owner
- **Manual gates for high-risk changes**: Terraform plan review, production deployment
- **Quarterly audits**: RBAC, break-glass usage, PR reviews

Separation of duties is friction by design. Friction prevents errors and malicious changes.