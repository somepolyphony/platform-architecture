# Hub-Spoke Rationale

## Why Hub-Spoke is Chosen

The hub-spoke network model centralises shared networking services whilst providing isolation for workloads.

This is not the only viable model. It is chosen because it balances security, scalability, and operational reality.

## Hub-Spoke Architecture

### Hub VNet

The hub VNet contains shared networking services consumed by all workloads.

**Components**:
- Azure Firewall (inspection and filtering of all inter-VNet and internet-bound traffic)
- VPN Gateway or ExpressRoute Gateway (connectivity to on-premises or third-party networks)
- Azure Bastion (secure admin access to VMs without public IPs)
- DNS forwarders (custom DNS resolution for hybrid scenarios)

**Ownership**: Platform team. Changes require change control.

**Blast Radius**: Hub failure affects all workloads. Treated as Tier 0 infrastructure.

### Spoke VNets

Spoke VNets contain workload-specific resources.

**Examples**:
- VNet for ecommerce application (web tier, app tier, database tier)
- VNet for data pipeline (data ingestion VMs, analytics services)

**Ownership**: Delivery teams. Platform team has Reader access for troubleshooting.

**Isolation**: Spokes cannot communicate directly. All inter-spoke traffic routes through hub firewall.

### Peering Relationships

- Each spoke VNet peers to hub VNet (one peering per spoke)
- Spokes do **not** peer to each other
- Hub VNet has transitive routing configured (user-defined routes ensure traffic between spokes is forced through Azure Firewall)

**Why No Direct Spoke-to-Spoke Peering**: Direct peering bypasses firewall inspection. Security policies cannot be enforced.

## Why This Model is Chosen

### 1. Centralised Security Enforcement

All inter-VNet and internet-bound traffic passes through Azure Firewall in the hub.

**Benefit**: Security policies (allow/deny rules, threat intelligence) are enforced centrally. No per-workload firewall configuration required.

**Alternative (Rejected)**: Decentralised firewalls per spoke. Does not scale. Policy fragmentation.

### 2. Shared Services Efficiency

Hub VNet provides shared services (VPN Gateway, Bastion, DNS) consumed by all spokes.

**Benefit**: Cost efficiency. Single VPN Gateway instead of one per spoke. Single Bastion instead of one per spoke.

**Alternative (Rejected)**: Duplicate shared services in each spoke. Expensive. Operationally complex.

### 3. Workload Isolation

Spokes are isolated by default. A compromise in Spoke A does not allow lateral movement to Spoke B without firewall traversal.

**Benefit**: Blast radius containment. Network segmentation is enforced architecturally.

**Alternative (Rejected)**: Flat network with NSGs only. NSG misconfigurations are common. Architectural isolation is stronger.

### 4. Scalability

Hub-spoke scales to hundreds of spokes (limited by VNet peering quotas, ~500 per hub).

**Benefit**: New workloads are onboarded by creating a spoke and peering to hub. No re-architecture required.

**Alternative (Rejected)**: Mesh networking. Peering relationships grow exponentially (N*(N-1)/2 peerings for N VNets). Does not scale.

## When Hub-Spoke is Inappropriate

### 1. Ultra-Low Latency Requirements

**Scenario**: Workload requires sub-millisecond latency between services.

**Problem**: Hub-spoke adds an additional network hop (spoke → hub firewall → spoke). Latency increases.

**Alternative**: Deploy workload in a single VNet with internal services. No hub transit.

**Trade-Off**: Security policies are enforced via NSGs only. Weaker than firewall-based inspection.

### 2. Global Multi-Region Architectures

**Scenario**: Workloads span multiple Azure regions with frequent cross-region communication.

**Problem**: Hub-spoke is regional. Cross-region traffic requires inter-hub peering or Virtual WAN. Complexity and cost increase.

**Alternative**: Azure Virtual WAN provides native multi-region hub-spoke with global transit. Higher cost.

**Trade-Off**: Virtual WAN adds abstraction layer. Less control over routing and firewall configuration.

### 3. Regulatory Isolation Requirements

**Scenario**: Workload must be fully isolated from all other workloads due to compliance (e.g., PCI-DSS, HIPAA).

**Problem**: Hub-spoke shares hub infrastructure. Shared blast radius.

**Alternative**: Dedicated VNet with no hub peering. Workload is fully isolated.

**Trade-Off**: No shared services (VPN, Bastion, Firewall). Higher cost. No inter-workload connectivity.

### 4. Third-Party Network Appliances

**Scenario**: Organisation requires Palo Alto, Fortinet, or other third-party firewall instead of Azure Firewall.

**Problem**: Third-party firewalls require dedicated VMs in hub. Operational complexity increases.

**Alternative**: Deploy third-party appliance in hub VNet. Complexity accepted.

**Trade-Off**: Azure-native automation (Azure Firewall Manager, Azure Policy integration) is not available. Manual configuration required.

## Hub-Spoke Failure Modes

### 1. Hub VNet Outage

**Scenario**: Azure Firewall in hub VNet becomes unavailable.

**Impact**: All inter-spoke communication is blocked. Internet egress is blocked.

**Mitigation**: Azure Firewall has 99.95% SLA. Deploy multiple firewall instances in different availability zones. Configure UDRs to fail over to secondary firewall.

**Residual Risk**: Regional Azure outage affects entire hub. No perfect mitigation.

### 2. VNet Peering Limit Reached

**Scenario**: Hub VNet has 500 spoke peerings. New workloads cannot be onboarded.

**Impact**: Onboarding blocked until existing spokes are consolidated or removed.

**Mitigation**: Monitor peering count. Plan for multiple hubs per region if scale exceeds limits. Migrate to Azure Virtual WAN for larger scale.

### 3. Misconfigured User-Defined Routes (UDRs)

**Scenario**: UDR incorrectly configured. Traffic does not route through firewall.

**Impact**: Security policies are bypassed. Lateral movement is possible.

**Mitigation**: Terraform manages all UDRs. Manual UDR changes are blocked via Azure Policy. Quarterly audit of routing configuration.

### 4. Azure Firewall Misconfiguration

**Scenario**: Firewall rule permits unintended traffic (e.g., RDP from internet).

**Impact**: Attack surface increased.

**Mitigation**: Firewall rules managed via Terraform with peer review. Default-deny policy. Quarterly rule audit.

## Hub-Spoke vs Alternatives

### Hub-Spoke vs Flat Network

| Aspect              | Hub-Spoke                          | Flat Network                  |
|---------------------|------------------------------------|-------------------------------|
| Security            | Firewall inspects all traffic      | NSGs only                     |
| Scalability         | Scales to 500 spokes per hub       | Scales poorly (routing complexity) |
| Cost                | Moderate (shared hub costs)        | Low (no hub infrastructure)   |
| Operational Complexity | Moderate (UDRs, peering)        | Low (simpler routing)         |
| Blast Radius        | Contained per spoke                | High (lateral movement easier)|

**Recommendation**: Hub-spoke for production. Flat network acceptable for sandbox or small non-production environments.

### Hub-Spoke vs Mesh Networking

| Aspect              | Hub-Spoke                          | Mesh Networking               |
|---------------------|------------------------------------|-------------------------------|
| Security            | Centralised firewall               | Decentralised (per-VNet firewalls) |
| Scalability         | Scales to 500 spokes               | Does not scale (N² peering relationships) |
| Cost                | Moderate                           | High (duplicate firewalls)    |
| Operational Complexity | Moderate                        | High (complex routing)        |

**Recommendation**: Hub-spoke. Mesh networking does not scale and fragments security policy.

### Hub-Spoke vs Azure Virtual WAN

| Aspect              | Hub-Spoke                          | Azure Virtual WAN             |
|---------------------|------------------------------------|-------------------------------|
| Multi-Region        | Requires inter-hub peering         | Native global transit         |
| Scalability         | 500 spokes per hub                 | 1,000+ spokes per region      |
| Cost                | Lower                              | Higher (VWAN gateway costs)   |
| Control             | Full control over routing/firewall | Abstracted (less control)     |
| Operational Complexity | Moderate                        | Lower (Microsoft-managed)     |

**Recommendation**: Hub-spoke for single-region or cost-sensitive environments. Virtual WAN for global multi-region requirements.

## Operational Considerations

### Hub Network is a Bottleneck

All platform changes to hub network affect every workload.

**Examples**:
- Azure Firewall upgrade requires maintenance window
- VPN Gateway failover causes brief connectivity loss
- DNS forwarder changes affect name resolution for all spokes

**Mitigation**: Change control for hub network. Maintenance windows during low-traffic periods. Redundancy via availability zones.

### Spoke Onboarding Process

New spokes are provisioned via Terraform.

**Steps**:
1. Delivery team requests spoke VNet via ServiceNow
2. Platform team provisions spoke VNet with:
   - Peering to hub VNet
   - UDRs forcing traffic through hub firewall
   - NSGs with default-deny rules
   - Subnets pre-configured (web, app, data tiers)
3. Delivery team deploys workload resources in spoke VNet

**Timeline**: New spoke VNets provisioned within 1 business day.

### Firewall Rule Requests

Delivery teams do **not** have access to Azure Firewall configuration.

**Process**:
1. Delivery team submits firewall rule request (source, destination, port, protocol, justification)
2. Platform team reviews and approves
3. Firewall rule added via Terraform
4. Change deployed during next maintenance window

**Timeline**: Non-urgent changes deployed within 1 week. Urgent changes deployed within 24 hours.

## Future Evolution

Hub-spoke is not permanent. As the organisation scales or requirements change, migration to Azure Virtual WAN may be necessary.

**Triggers for Migration**:
- Spoke count approaches 500 per hub
- Multi-region workloads require global transit
- Operational complexity of managing multiple regional hubs becomes excessive

Migration is a multi-month effort and requires executive approval. Not undertaken lightly.