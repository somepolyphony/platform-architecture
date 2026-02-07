# MSP vs Internal

## Differences Between MSP-Led and Internal Models

Platform operating models differ significantly between Managed Service Provider (MSP) environments and internal IT teams.

This document explains the trade-offs and why some decisions are MSP-specific or internal-specific.

## MSP Model

### Characteristics

- **Multi-Tenant by Default**: MSP manages Azure environments for 10-100+ clients
- **Standardisation Mandatory**: Bespoke configurations do not scale across clients
- **Centralised Control**: MSP team has full control over platform. Client has limited self-service.
- **Cost Sensitivity**: MSP profitability depends on automation and efficiency
- **Security Posture**: High (MSP breach affects all clients; reputational and legal risk)

### Platform Design Implications

1. **One Architecture for All Clients**

   MSP cannot afford client-specific platform architectures.

   **Result**:
   - Hub-spoke networking model for all clients
   - Standardised Terraform modules
   - No exceptions for "client preferences"

   **Trade-Off**: Some clients constrained by MSP's opinionated design.

2. **Centralised Automation**

   MSP uses centralised automation (Terraform, Intune, Defender) across all clients.

   **Result**:
   - MSP engineers can manage multiple clients without context switching
   - Automation failures affect multiple clients simultaneously

   **Trade-Off**: Single point of failure. Bug in automation impacts all clients.

3. **Limited Client Self-Service**

   Clients cannot modify platform controls (policies, networking, RBAC).

   **Result**:
   - MSP maintains security posture across all clients
   - Clients submit requests for changes (firewall rules, RBAC, policy exemptions)

   **Trade-Off**: Client agility reduced. All changes bottleneck through MSP.

4. **Cost Pass-Through**

   MSP passes infrastructure costs to clients with markup.

   **Result**:
   - Clients pay for Defender for Cloud, Firewall, Bastion
   - Cost-conscious clients push back on security controls

   **Trade-Off**: MSP must justify every cost. "Free" security controls (policy, RBAC) preferred over paid controls.

### MSP-Specific Challenges

**Challenge 1: Client Wants Custom Architecture**

- Client requests mesh networking instead of hub-spoke
- MSP policy: No bespoke architectures

**Resolution**: Client accepts MSP architecture or finds different MSP.

**Challenge 2: Client Wants Direct Azure Portal Access**

- Client wants Owner access to modify subscriptions directly
- MSP policy: Clients have Contributor only. Changes via MSP-managed Terraform.

**Resolution**: Client accepts limited access or migrates to internal IT model.

**Challenge 3: Cost Pressure**

- Client refuses to pay for private endpoints (£7/month per endpoint × 50 endpoints = £350/month)
- MSP policy: Private endpoints mandatory in production.

**Resolution**: MSP absorbs cost or client accepts security risk with documented exception.

## Internal IT Model

### Characteristics

- **Single Tenant**: One organisation, one Azure environment
- **Flexibility Possible**: Internal teams can accommodate workload-specific requirements
- **Distributed Control**: Platform team enforces guardrails. Delivery teams have more autonomy.
- **Cost Absorbed Internally**: No direct client billing. Costs allocated to business units.
- **Security Posture**: High but context-dependent (regulated industries stricter than startups)

### Platform Design Implications

1. **Workload-Specific Architectures**

   Internal teams can support multiple networking models or exceptions.

   **Result**:
   - Most workloads use hub-spoke
   - Niche workloads (ML, HPC) have dedicated VNets with custom routing

   **Trade-Off**: Operational complexity increases. More architectures to support.

2. **Delegated Automation**

   Delivery teams write their own Terraform for workloads. Platform team provides modules.

   **Result**:
   - Delivery teams iterate faster
   - Platform team reviews but does not gate every change

   **Trade-Off**: Inconsistency increases. Delivery teams bypass modules.

3. **Self-Service Within Guardrails**

   Delivery teams have Contributor access to their subscriptions.

   **Result**:
   - Can deploy workloads without platform team involvement
   - Azure Policy enforces guardrails (encryption, private endpoints)

   **Trade-Off**: Delivery teams may misconfigure within policy limits.

4. **Cost Allocated Internally**

   Business units absorb Azure costs via chargeback or showback.

   **Result**:
   - Cost pressure is indirect (budget reviews, FinOps team)
   - Security controls less likely to be rejected based on cost

   **Trade-Off**: Cost visibility weaker. Overprovisioning common.

### Internal IT-Specific Challenges

**Challenge 1: Delivery Team Bypasses Modules**

- Delivery team writes raw Terraform instead of using approved modules
- Platform team discovers during audit

**Resolution**: Azure Policy blocks non-compliant resources. Delivery team forced to use modules.

**Challenge 2: Shadow IT**

- Delivery team creates personal Azure subscription outside platform control
- Workload deployed without security controls

**Resolution**: Difficult to prevent. Detection via Entra ID audit logs (unauthorised subscription creation). Workload migrated or shut down.

**Challenge 3: Competing Priorities**

- Delivery team prioritises feature delivery over security compliance
- Platform team blocks deployment until compliance achieved

**Resolution**: Escalation to executive leadership. Business vs security trade-off decided at leadership level.

## MSP vs Internal Comparison

| Aspect                    | MSP Model                          | Internal IT Model                    |
|---------------------------|------------------------------------|--------------------------------------|
| Standardisation           | Mandatory                          | Recommended                          |
| Client/Delivery Autonomy  | Low (request-driven)               | High (self-service within guardrails)|
| Platform Team Size        | Small (manages many clients)       | Larger (dedicated to one org)        |
| Cost Sensitivity          | High (client pays directly)        | Moderate (indirect via chargeback)   |
| Custom Architectures      | Prohibited                         | Permitted with approval              |
| Terraform Ownership       | Centralised (MSP team)             | Distributed (delivery teams)         |
| RBAC Model                | Contributor only for clients       | Contributor/Owner for delivery teams |
| Exception Frequency       | Rare (standardisation enforced)    | More common (flexibility needed)     |
| Security Posture          | Very high (reputational risk)      | High (context-dependent)             |

## Hybrid Model (Rare)

Some organisations use a hybrid model:
- Platform team operates like an internal MSP
- Delivery teams treated as "clients"
- Standardisation enforced but with internal cost dynamics

**When to Use**: Large enterprises (1,000+ employees) with multiple business units requiring strict governance.

**Trade-Off**: Combines MSP rigidity with internal politics. Requires strong executive backing.

## Decision Tree: MSP or Internal?

**Use MSP Model If**:
- Serving external clients
- Standardisation is non-negotiable
- Small platform team managing many environments
- Cost pass-through to clients

**Use Internal IT Model If**:
- Single organisation
- Workload diversity requires flexibility
- Large platform team with dedicated resources
- Cost allocated internally

**Use Hybrid Model If**:
- Large enterprise with multiple business units
- Business units operate semi-independently
- Strong governance required but some flexibility needed

## MSP-to-Internal Migration

Organisations outgrow MSPs and migrate to internal IT.

**Triggers**:
- MSP constraints limit business agility
- Cost of MSP services exceeds internal team cost
- Organisation wants direct control over platform

**Migration Challenges**:
1. **Knowledge Transfer**: MSP holds platform knowledge. Internal team must ramp up.
2. **Operational Continuity**: Cannot pause production during migration.
3. **Architecture Refactoring**: MSP architecture may not suit internal needs.

**Timeline**: 6-12 months for full migration.

## Shared Principles (MSP and Internal)

Regardless of model, these principles apply:

1. **Security is Non-Negotiable**: Encryption, MFA, private endpoints mandatory.
2. **Automation Over Manual**: Terraform for infrastructure. Intune for endpoints.
3. **Least Privilege**: RBAC scoped to minimum necessary.
4. **Audit and Compliance**: Immutable logs. Quarterly reviews.
5. **Documentation**: Architecture decisions documented (this repo).

## MSP vs Internal Summary

- **MSP**: Standardisation, centralised control, low client autonomy, cost-sensitive
- **Internal IT**: Flexibility, distributed control, high delivery autonomy, moderate cost sensitivity
- **Hybrid**: Internal MSP model for large enterprises
- **Migration**: MSP to internal is common as organisations scale

**Model choice depends on organisational structure, scale, and risk appetite.**

Neither model is inherently better. Context determines appropriateness.