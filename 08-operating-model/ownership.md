# Ownership

## Platform Ownership vs Delivery Teams

Clear ownership boundaries prevent confusion, delays, and security gaps.

Platform team owns infrastructure and security guardrails. Delivery teams own workloads within those guardrails.

## Platform Team Responsibilities

### What Platform Team Owns

1. **Management Groups and Subscriptions**

   - Create and manage management group hierarchy
   - Provision subscriptions via Terraform
   - Assign subscriptions to management groups
   - Enforce management group policies

2. **Hub Network**

   - Hub VNet design and configuration
   - Azure Firewall rules and maintenance
   - VPN Gateway or ExpressRoute connectivity
   - Azure Bastion deployment and access control
   - DNS forwarders for hybrid scenarios

3. **Azure Policy**

   - Define policy definitions
   - Assign policies at management group and subscription scope
   - Review and approve policy exemptions
   - Monitor policy compliance

4. **RBAC Assignments (Platform-Level)**

   - Owner and Contributor at management group scope
   - Platform-wide service principals
   - Break-glass account management
   - PIM configuration and approval workflows

5. **Central Logging and Monitoring**

   - Log Analytics workspace configuration
   - Diagnostic settings enforcement
   - Azure Monitor alert rules (platform health)
   - Sentinel (SIEM) configuration and rule tuning

6. **Terraform Modules and Governance**

   - Create and maintain reusable Terraform modules
   - Publish modules to internal registry
   - Review and approve module usage by delivery teams
   - Enforce Terraform coding standards

7. **Defender for Cloud**

   - Enable Defender plans on subscriptions
   - Configure Defender settings and alerts
   - Review Defender recommendations
   - Triage false positives

8. **Platform Documentation**

   - This repository (platform architecture and guardrails)
   - Terraform module documentation
   - Runbooks for incident response and operational procedures

### What Platform Team Does NOT Own

- Application code or CI/CD pipelines (delivery team responsibility)
- Workload-specific resources (VMs, databases, storage accounts in workload subscriptions)
- Application monitoring and logging (delivery team responsibility)
- Workload RBAC assignments (delivery team requests, platform team reviews)
- Secrets management in workloads (delivery team uses Key Vault, platform team provides module)

## Delivery Team Responsibilities

### What Delivery Teams Own

1. **Workload Infrastructure**

   - Virtual machines, App Services, databases, storage accounts
   - Resource groups within their subscriptions
   - Spoke VNets and subnets
   - NSG rules for application-specific traffic

2. **Application Deployment**

   - CI/CD pipelines (Azure DevOps, GitHub Actions)
   - Application code and configuration
   - Container images and Kubernetes manifests
   - Application secrets (stored in Key Vault, accessed by workload)

3. **Workload Monitoring**

   - Application Performance Monitoring (APM)
   - Application logs and telemetry
   - Business metrics and dashboards
   - Application-specific alerts

4. **Workload RBAC**

   - Reader, Contributor, or Owner within their resource groups
   - Service principal management for CI/CD
   - Break-glass access for their workload (not platform-wide)

5. **Compliance and Documentation**

   - Workload-specific security documentation
   - Application architecture diagrams
   - Incident response runbooks for their application
   - Compliance evidence for application-level controls

### What Delivery Teams Do NOT Own

- Management group policies (platform team responsibility)
- Hub network configuration (platform team responsibility)
- Azure Firewall rules (requested via platform team)
- Defender for Cloud configuration (platform team responsibility)
- Cross-subscription networking (platform team approves and implements)

## Shared Responsibilities

Some responsibilities require collaboration between platform and delivery teams.

| Responsibility                  | Platform Team Role                  | Delivery Team Role                    |
|---------------------------------|-------------------------------------|---------------------------------------|
| Spoke VNet Provisioning         | Create VNet, peer to hub            | Define subnet ranges and requirements |
| Firewall Rule Requests          | Review, approve, implement          | Submit request with justification     |
| Policy Exemptions               | Review, approve, implement          | Submit request with justification     |
| Incident Response               | Platform-level issues (hub network) | Workload-level issues (application)   |
| Security Vulnerabilities        | Platform controls and Defender      | Application code and dependencies     |
| Cost Management                 | Subscription-level budgets          | Workload-level optimisation           |

## Ownership Conflicts

Conflicts occur when ownership boundaries are unclear.

### Example 1: NSG Rules

**Conflict**: Delivery team wants to add NSG rule. Platform team says "NSGs are platform-managed."

**Resolution**:
- **Platform team** owns NSG configuration (ensures rules do not conflict with firewall or policy)
- **Delivery team** requests NSG rule changes via ServiceNow
- **Platform team** implements approved changes

**Why**: NSG misconfiguration can bypass firewall. Centralised control necessary.

### Example 2: Defender for Cloud Recommendations

**Conflict**: Defender flags VM without endpoint protection. Who is responsible?

**Resolution**:
- **Platform team** ensures Defender for Endpoint is deployed via policy
- **Delivery team** investigates why policy failed on their VM
- If policy bug: Platform team fixes
- If workload misconfiguration: Delivery team fixes

**Why**: Both teams have partial responsibility. Escalation path resolves ambiguity.

### Example 3: Firewall Rule Breakage

**Conflict**: Firewall rule change breaks workload. Who is accountable?

**Resolution**:
- **Platform team** is accountable if firewall change was not tested or approved
- **Delivery team** is accountable if workload depends on unapproved firewall rule

**Why**: Accountability depends on whether process was followed.

## RACI Matrix (Platform vs Delivery)

| Task                            | Platform Team | Delivery Team | Security Team | Executive   |
|---------------------------------|---------------|---------------|---------------|-------------|
| Management Group Policy         | Responsible   | Informed      | Accountable   | Consulted   |
| Subscription Provisioning       | Responsible   | Consulted     | Informed      | -           |
| Hub Network Configuration       | Responsible   | Informed      | Consulted     | -           |
| Spoke VNet Provisioning         | Responsible   | Consulted     | Informed      | -           |
| Firewall Rule Implementation    | Responsible   | Consulted     | Informed      | -           |
| Workload Resource Deployment    | Consulted     | Responsible   | Informed      | -           |
| Application Security            | Consulted     | Responsible   | Accountable   | -           |
| Defender for Cloud Config       | Responsible   | Informed      | Accountable   | -           |
| Policy Exemption Approval       | Consulted     | Responsible   | Accountable   | Informed    |
| Incident Response (Platform)    | Responsible   | Consulted     | Accountable   | Informed    |
| Incident Response (Workload)    | Consulted     | Responsible   | Informed      | -           |

**Legend**:
- **Responsible**: Does the work
- **Accountable**: Ultimately answerable (only one A per task)
- **Consulted**: Provides input
- **Informed**: Kept informed of decisions

## Escalation Paths

When ownership is ambiguous, escalate.

### Level 1: Peer Discussion

Platform engineer and delivery engineer discuss and resolve.

**Timeline**: Resolve within 1 business day.

### Level 2: Team Lead

Platform lead and delivery lead discuss and resolve.

**Timeline**: Resolve within 2 business days.

### Level 3: Architecture Review Board

Platform lead, security team, and delivery lead present issue to architecture board.

**Timeline**: Resolve within 1 week (next scheduled ARB meeting).

### Level 4: Executive Leadership

CTO or CISO makes final decision.

**Timeline**: Resolve within 2 weeks.

## Ownership Principles

1. **Default Owner**: If ambiguous, platform team is default owner until resolved.
2. **No Orphaned Infrastructure**: Every resource has an owner. Ownerless resources are deleted.
3. **Ownership is Documented**: Every subscription and resource group has owner tag.
4. **Ownership Can Change**: Workload transfers between teams require RBAC and documentation updates.

## Ownership Governance

### Subscription Ownership Tags

Every subscription has mandatory tags:

```hcl
tags = {
  owner            = "delivery-team-a@company.com"
  cost_centre      = "CC-12345"
  environment      = "production"
  workload_name    = "ecommerce"
  platform_managed = "false"  # true for platform subscriptions
}
```

### Quarterly Ownership Audit

Platform team audits subscription ownership quarterly.

**Audit Questions**:
- Does subscription still have valid owner?
- Is owner email address still active?
- Is workload still in use?
- Are costs attributed correctly?

**Action Items**:
- Update ownership tags if team structure changed
- Delete unused subscriptions
- Reallocate costs if ownership transferred

## Ownership Summary

- **Platform team**: Management groups, hub network, policies, RBAC, central logging, Defender, Terraform modules
- **Delivery team**: Workload infrastructure, application deployment, workload monitoring, workload RBAC
- **Shared**: Spoke VNets, firewall rules, policy exemptions, incident response
- **Clear escalation paths**: Peer → team lead → architecture board → executive leadership
- **Documented ownership**: Tags and documentation define who owns what

Ownership clarity prevents delays, reduces conflicts, and ensures accountability.