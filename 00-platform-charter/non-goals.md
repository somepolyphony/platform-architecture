# Non-Goals

## Explicit Exclusions

This document defines what the platform **deliberately does not provide** and why.

These are not oversights. These are conscious decisions.

### 1. Developer Productivity Tooling

**What**: IDEs, CI/CD pipeline templates, container registries, or package feeds.

**Why**: The platform enforces security and governance. Productivity tooling is the responsibility of delivery teams.

**Rejected Approach**: Centrally managing developer tooling creates bottlenecks and slows innovation.

**Exception**: Where productivity tooling has security implications (e.g., container image scanning, dependency vulnerability checks), the platform provides enforcement hooks, not the tools themselves.

### 2. Application-Level Security

**What**: Web application firewalls, API rate limiting, or application authentication logic.

**Why**: The platform secures infrastructure and identity. Application security is the responsibility of delivery teams.

**Rejected Approach**: Centralising application security creates a false sense of safety and delays accountability.

**Exception**: Defender for Cloud provides workload protection (e.g., SQL injection detection, anomalous file access). These are detective controls, not preventative.

### 3. Workload Monitoring and Observability

**What**: Application performance monitoring (APM), distributed tracing, or business metrics dashboards.

**Why**: The platform monitors platform health (identity, networking, security). Workload monitoring is the responsibility of delivery teams.

**Rejected Approach**: Centralised observability becomes a single point of failure and scales poorly.

**Exception**: Platform teams enforce log forwarding to a central SIEM for security events. Delivery teams own their own operational telemetry.

### 4. Bespoke Networking Models

**What**: Mesh networking, SD-WAN, or application-driven network segmentation.

**Why**: The platform provides a hub-spoke network model. Deviations create security inconsistencies and operational complexity.

**Rejected Approach**: Every bespoke network topology is a unique security review burden.

**Exception**: Workloads with regulatory isolation requirements (e.g., PCI-DSS) may have dedicated VNets with explicit connectivity controls.

### 5. Multi-Cloud or Hybrid Cloud

**What**: AWS, Google Cloud, or on-premises VMware integration.

**Why**: The platform is Azure-native. Multi-cloud adds complexity without strategic value for this organisation.

**Rejected Approach**: Multi-cloud strategies dilute expertise, fragment security controls, and increase cost.

**Exception**: Integration with third-party SaaS providers (Salesforce, ServiceNow) is in scope where identity federation is required.

### 6. Zero-Trust Architecture (as a Marketing Term)

**What**: Implementing every vendor-recommended zero-trust control without trade-off analysis.

**Why**: Zero-trust is a model, not a checklist. Blind implementation of all controls creates operational paralysis.

**Rejected Approach**: Adopting zero-trust controls because they are "best practice" without understanding the trade-offs.

**What We Do Instead**: Implement identity-centric controls (Conditional Access, PIM, private endpoints) and accept residual risk where controls are not cost-effective.

### 7. AI/ML Platform Services

**What**: Centralised Jupyter environments, MLOps pipelines, or model registries.

**Why**: Machine learning workloads have unique compute, storage, and tooling requirements. Centralising them constrains innovation.

**Rejected Approach**: Treating ML workloads like standard applications creates friction and drives shadow IT.

**What We Do Instead**: Provide ML teams with dedicated subscriptions and enforce security baselines (encryption, private endpoints, audit logging) without prescribing tooling.

### 8. Desktop-as-a-Service (DaaS)

**What**: Azure Virtual Desktop or Windows 365 for end-user computing.

**Why**: Endpoint management is handled via Intune on physical devices. Virtual desktops add cost and complexity without strategic value.

**Rejected Approach**: Replacing physical endpoints with virtual desktops to "simplify" management.

**What We Do Instead**: Enforce Intune compliance on physical devices. Where remote work requires secure access, Conditional Access and VPN suffice.

### 9. Incident Response and Security Operations

**What**: 24/7 SOC, threat hunting, or incident response playbooks.

**Why**: The platform provides detective controls (Defender, audit logs, alerts). Security operations are a separate function.

**Rejected Approach**: Expecting platform engineers to act as SOC analysts blurs responsibilities and delays response.

**What We Do Instead**: Platform teams ensure Defender for Cloud and Sentinel are configured correctly. Security operations teams action the alerts.

### 10. Shadow IT Prevention

**What**: Blocking delivery teams from using unapproved SaaS services or public cloud accounts.

**Why**: Prevention is impossible. Users will bypass controls if they are too restrictive.

**Rejected Approach**: Network-level blocking of SaaS services creates user frustration and drives VPN/hotspot usage.

**What We Do Instead**: Conditional Access and Microsoft Defender for Cloud Apps provide visibility and risk-based controls. Outright blocking is used only for high-risk services (e.g., personal file-sharing sites).

## Rejected Architectural Patterns

### 1. Flat Network with Firewall Rules

**Why Rejected**: Every workload becomes a unique firewall rule set. Does not scale. Increases attack surface.

**What We Do Instead**: Hub-spoke network model with private endpoints and service endpoints.

### 2. Permanent Administrator Access

**Why Rejected**: Standing privileges are the primary enabler of lateral movement and privilege escalation.

**What We Do Instead**: Privileged Identity Management (PIM) with just-in-time access and approval workflows.

### 3. Azure Portal for Production Changes

**Why Rejected**: Manual changes are not auditable, not repeatable, and not peer-reviewed.

**What We Do Instead**: Terraform with CI/CD pipelines and mandatory pull request approvals.

### 4. Shared Service Accounts

**Why Rejected**: Credentials cannot be rotated safely. Audit trails are ambiguous.

**What We Do Instead**: Managed identities for Azure resources. Service principals scoped to single applications with short-lived credentials.

### 5. Optional Multi-Factor Authentication

**Why Rejected**: Passwords are not sufficient authentication in 2026. Phishing and credential theft are trivial.

**What We Do Instead**: MFA enforced for all identities via Conditional Access. No exceptions.

### 6. Public IP Addresses on VMs

**Why Rejected**: Every public IP is an attack vector. Patching lag is inevitable.

**What We Do Instead**: Azure Bastion for admin access. Application Gateway or Azure Front Door for user-facing workloads.

## Controversial Decisions

These decisions are frequently challenged. They are intentional.

### 1. No Root Access on Managed Services

**Decision**: Delivery teams do not have root access to Azure SQL, App Service, or other PaaS services.

**Rationale**: Managed services are opaque by design. Platform controls are enforced at the Azure Resource Manager layer.

**Pushback**: "We need low-level access for debugging."

**Response**: If you require root access, use IaaS (VMs). PaaS services are deliberately abstracted.

### 2. No Cross-Subscription Networking by Default

**Decision**: VNets in different subscriptions are not peered without explicit justification and approval.

**Rationale**: Every peering relationship increases blast radius and complicates network security policies.

**Pushback**: "We need to share services across workloads."

**Response**: Use private endpoints or expose services via Private Link. Shared services belong in a dedicated subscription.

### 3. No Contributor Access to Production Subscriptions

**Decision**: Delivery teams have Reader access to production subscriptions. Changes are deployed via CI/CD pipelines.

**Rationale**: Human-initiated changes in production are error-prone and unauditable.

**Pushback**: "We need to debug incidents in production."

**Response**: Use PIM to request time-limited access with justification. All changes must be reverted or codified in Terraform post-incident.

### 4. No Windows Server Below 2022

**Decision**: Windows Server 2019 and earlier are not deployed in new subscriptions.

**Rationale**: Older OS versions have degraded security posture and shorter support lifecycles.

**Pushback**: "Our application is only certified on Server 2016."

**Response**: Re-certify or re-architect. Legacy OS versions are unacceptable security debt.

## Future Non-Goals

The following are not currently in scope but may be revisited:

- **Kubernetes Platform-as-a-Service**: AKS is available but not centrally managed. Delivery teams own their clusters.
- **FinOps and Cost Optimisation**: Platform teams enforce tagging and resource limits. Cost optimisation is the responsibility of delivery teams.
- **Data Platform Services**: Data lakes, Synapse, and Databricks are available but not centrally governed beyond security baselines.

These may become platform responsibilities if demand justifies the investment.