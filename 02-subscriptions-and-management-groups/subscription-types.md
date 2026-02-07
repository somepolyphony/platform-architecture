# Subscription Types

## Platform vs Workload Subscriptions

Azure subscriptions are not equal. They serve different purposes and have different security postures.

## Platform Subscriptions

Platform subscriptions provide **foundational services** consumed by workload subscriptions.

### Characteristics

- Owned by platform team, not delivery teams
- Contain shared infrastructure (hub network, central logging, Defender for Cloud management)
- Changes are infrequent and require change control
- Downtime affects multiple workloads simultaneously

### Examples

#### Platform-Hub-Network

**Purpose**: Central hub network. All spoke VNets peer to this hub.

**Contains**:
- Azure Firewall
- VPN Gateway or ExpressRoute Gateway
- Azure Bastion
- DNS forwarders

**RBAC**: Platform team has Contributor. Delivery teams have no access.

**Blast Radius**: Failure of hub network impacts all workloads. Single point of failure by design.

#### Platform-Logging

**Purpose**: Central logging and monitoring.

**Contains**:
- Log Analytics workspace
- Azure Monitor
- Sentinel (SIEM)

**RBAC**: Security team has Contributor. Platform team has Reader. Delivery teams have no access.

**Blast Radius**: Loss of logging impacts audit and incident response. Does not impact workload availability.

#### Platform-Security

**Purpose**: Security tooling and Defender for Cloud management.

**Contains**:
- Microsoft Defender for Cloud configuration
- Defender for DevOps connectors
- Security contact email configuration

**RBAC**: Security team has Contributor. Platform team has Reader.

**Blast Radius**: Failure impacts security telemetry. Does not impact workload availability.

### Platform Subscription Policies

- No public endpoints permitted (private endpoints mandatory)
- Change control via Terraform only
- RBAC assignments require executive approval
- Deletion of resources blocked via Azure Policy

## Workload Subscriptions

Workload subscriptions contain **applications and services** delivered by delivery teams.

### Characteristics

- Owned by delivery teams
- Contain application-specific resources (VMs, App Services, databases, storage accounts)
- Changes are frequent and do not require platform team approval
- Downtime affects only the specific workload, not others

### Production Workload Subscriptions

**Purpose**: Live customer-facing services.

**Contains**:
- Application infrastructure
- Spoke VNet peered to hub
- Application-specific PaaS services (SQL Database, Cosmos DB, App Service)

**RBAC**: Delivery team has Reader. Changes via CI/CD pipelines only.

**Policies**:
- No public endpoints permitted
- Mandatory private endpoints for PaaS services
- Audit logging with 365-day retention
- Terraform state stored in centralised storage account

**Naming Convention**: `workload-production-<workload-name>`

### Non-Production Workload Subscriptions

**Purpose**: Development, staging, UAT, and testing environments.

**Contains**:
- Same infrastructure as production but with relaxed controls

**RBAC**: Delivery team has Contributor.

**Policies**:
- Public endpoints permitted (for developer convenience)
- Shorter log retention (90 days)
- Cost controls (VM sizes capped, auto-shutdown after business hours)

**Naming Convention**: `workload-nonprod-<workload-name>`

### Sandbox Subscriptions

**Purpose**: Experimentation, proof-of-concept, training.

**Contains**:
- Whatever the user needs to experiment

**RBAC**: User has Owner within their subscription.

**Policies**:
- Minimal enforcement (cost caps, time-boxing)
- Mandatory expiry (subscription auto-deleted after 90 days unless extended)
- Network isolation (no connectivity to hub or production VNets)

**Naming Convention**: `sandbox-<username>-<date>`

**Why Sandbox Exists**: Delivery teams need space to innovate without platform gatekeeping. Sandboxes provide freedom whilst containing blast radius.

## Subscription Boundaries

Subscriptions are **blast radius containers**.

A compromised subscription must not allow lateral movement to other subscriptions.

### Network Boundary

Subscriptions in different VNets cannot communicate unless explicitly peered.

**Default**: No connectivity between subscriptions.

**Exception**: Hub-spoke model allows workload VNets to communicate via hub firewall with explicit allow rules.

### Identity Boundary

Subscriptions share the same Entra ID tenant. A compromised identity with Owner on one subscription can potentially pivot to others.

**Mitigation**:
- Principle of least privilege enforced via RBAC
- Service principals scoped to specific subscriptions, not tenant root
- PIM enforced for Owner access

### Resource Boundary

Resources in one subscription cannot directly access resources in another subscription unless explicitly granted.

**Example**: VM in Subscription A cannot access storage account in Subscription B unless:
- Storage account has public endpoint (discouraged)
- Storage account has private endpoint and VNets are peered
- VM has managed identity with RBAC on storage account in Subscription B

## Subscription Provisioning

Subscriptions are **not** created manually via Azure Portal.

### Provisioning Process

1. Delivery team submits subscription request via ServiceNow (or equivalent)
2. Request includes:
   - Workload name
   - Environment (production, non-production, sandbox)
   - Cost centre
   - Owner (person responsible for RBAC and costs)
3. Platform team reviews and approves
4. Terraform provisions subscription with:
   - Placement in correct management group
   - Pre-configured spoke VNet peered to hub
   - Azure Policy assignments
   - RBAC assignments
   - Tagging (owner, cost centre, environment)
5. Subscription is handed to delivery team

**Timeline**: New subscriptions provisioned within 1 business day.

## Subscription Lifecycle

### Creation

- Automated via Terraform
- Subscription placed in correct management group
- Networking pre-configured (spoke VNet, NSGs, route tables)
- Logging enabled (forwarding to central Log Analytics workspace)

### Operation

- Delivery team deploys workloads via Terraform or CI/CD pipelines
- Platform team monitors subscription health and policy compliance
- Costs tracked and reported monthly

### Decommissioning

- Delivery team requests subscription deletion
- All resources deleted via Terraform
- RBAC assignments removed
- Subscription moved to "Decommissioned" management group (quarantine period)
- After 30 days, subscription is cancelled

**Why Quarantine Period**: Accidental deletion is recoverable. Immediate cancellation is not.

## Subscription Limits and Quotas

Azure imposes hard limits on subscriptions.

### Subscription Limits (2026)

- **Resource Groups**: 980 per subscription
- **VNet Peerings**: 500 per VNet (includes global peerings)
- **Storage Accounts**: 250 per subscription (can be increased via support ticket)
- **Public IP Addresses**: 1,000 per subscription
- **VM Cores**: Varies by VM family (typically 10-100 per region, increasable via quota request)

**Impact**: Large workloads may require multiple subscriptions.

**Mitigation**: Subscription design must account for scale limits. Split large workloads across multiple subscriptions if necessary.

### Known Bottlenecks

1. **VNet Peering Limit**

   Hub-spoke model is constrained by 500 peerings per hub VNet.

   **Solution**: Deploy multiple hub VNets per region or use Virtual WAN (increases cost and complexity).

2. **Subscription Creation Rate Limit**

   Azure limits subscription creation to ~10 per day per Enterprise Agreement.

   **Solution**: Batch subscription requests. Plan ahead for large platform rollouts.

3. **RBAC Assignment Limit**

   2,000 RBAC assignments per subscription.

   **Solution**: Use groups for RBAC assignments, not individual users.

## Subscription Naming Convention

Subscriptions must follow a consistent naming convention.

### Format

```
<type>-<environment>-<workload>-<region>
```

### Examples

- `platform-hub-network-uks` (platform, hub network, UK South)
- `workload-production-ecommerce-uks` (workload, production, ecommerce app, UK South)
- `workload-nonprod-datapipeline-euw` (workload, non-production, data pipeline, Europe West)
- `sandbox-johndoe-20260207` (sandbox, owned by John Doe, created 7 Feb 2026)

### Why Naming Matters

- Cost reporting aggregates by naming pattern
- RBAC and policy assignments target subscriptions by name
- Incident response requires clear identification of affected subscriptions

## Multi-Region Considerations

Some workloads span multiple Azure regions for high availability or disaster recovery.

**Pattern**: One subscription per region.

**Example**:
- `workload-production-ecommerce-uks` (primary region: UK South)
- `workload-production-ecommerce-euw` (secondary region: Europe West)

**Why**: Azure Policy and RBAC scope to subscriptions. Per-region subscriptions allow region-specific controls (e.g., different VM SKUs, different networking).

**Alternative**: Single subscription with resources in multiple regions. Simpler but less flexible.

## Subscription Security Posture Summary

| Subscription Type       | RBAC               | Public Endpoints | Change Control | Log Retention | Cost Controls |
|-------------------------|--------------------|------------------|----------------|---------------|---------------|
| Platform                | Platform team only | Blocked          | Terraform only | 365 days      | None          |
| Production Workload     | Reader (CI/CD)     | Blocked          | CI/CD only     | 365 days      | Alerts        |
| Non-Production Workload | Contributor        | Permitted        | Manual allowed | 90 days       | Auto-shutdown |
| Sandbox                 | Owner              | Permitted        | Manual allowed | 30 days       | Hard cap      |

Subscription design is the foundation of Azure governance. Get this wrong and everything else fails.