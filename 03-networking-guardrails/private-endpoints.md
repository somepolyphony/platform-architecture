# Private Endpoints

## Security Trade-Offs

Private endpoints remove public internet exposure of Azure PaaS services.

This is a security win. It also introduces operational pain.

## What Private Endpoints Provide

A private endpoint assigns a **private IP address** within your VNet to an Azure PaaS service.

**Example**:
- Storage account normally accessible at `storageaccount.blob.core.windows.net` (public endpoint)
- With private endpoint, storage account is accessible at `10.0.1.4` (private IP in your VNet)

**Result**: Storage account is no longer accessible from the public internet.

### Security Benefits

1. **No Public Internet Exposure**

   Attackers cannot reach the service from the internet, even with valid credentials.

   **Why This Matters**: Reduces attack surface. Credential theft alone is insufficient for data exfiltration.

2. **Network-Level Access Control**

   Access to the service requires network connectivity to the VNet (via VPN, ExpressRoute, or VM in the VNet).

   **Why This Matters**: Adds a second authentication factor (network access) beyond identity (credentials).

3. **Traffic Stays on Microsoft Backbone**

   Traffic between VMs and private endpoints never traverses the public internet.

   **Why This Matters**: Reduces man-in-the-middle risk. Lower latency.

## Operational Pain Points

Private endpoints are not frictionless. They introduce complexity.

### 1. DNS Resolution is Mandatory

Private endpoints require DNS records to resolve the service's public FQDN to the private IP.

**Problem**: Azure does not automatically update your DNS servers.

**Solution**: Use Azure Private DNS Zones. Create DNS zone for each service type (e.g., `privatelink.blob.core.windows.net`). Link DNS zone to all VNets.

**Failure Mode**: If DNS is misconfigured, applications resolve the public FQDN to the public IP. Private endpoint is bypassed.

**Example**:
- Storage account `storageaccount.blob.core.windows.net` should resolve to `10.0.1.4` (private IP)
- If DNS is broken, resolves to `20.50.20.30` (public IP)
- Application connects to public IP instead of private endpoint
- If storage account firewall is configured to block public access, connection fails

### 2. Cannot Mix Public and Private Access Easily

Once a private endpoint is created, you typically disable public network access on the PaaS service.

**Problem**: CI/CD pipelines, developer laptops, and external partners lose access.

**Workarounds**:
1. **Allow Specific Public IPs**: Configure PaaS service firewall to allow specific public IPs (e.g., GitHub Actions runners, office IPs). Fragile. IP addresses change.
2. **VPN or Bastion Access**: Developers connect via VPN to access private endpoints. Adds latency and friction.
3. **Self-Hosted Agents**: Deploy CI/CD agents inside VNet. Can access private endpoints. Operational overhead.

**Reality**: Most organisations use a hybrid approach. Some services remain publicly accessible (with IP whitelisting). Others are fully private.

### 3. Private Endpoints Consume IP Addresses

Each private endpoint consumes one private IP address in your VNet.

**Problem**: VNets have finite IP space. Large organisations with hundreds of storage accounts, databases, and Key Vaults can exhaust IP space.

**Mitigation**: Plan VNet address space for growth. Use large CIDR blocks (e.g., `/16` or `/20`). Reserve subnets specifically for private endpoints.

**Example**:
- VNet: `10.0.0.0/16` (65,536 IPs)
- Subnet for private endpoints: `10.0.1.0/24` (256 IPs)

### 4. Regional Availability

Private endpoints must be in the same region as the VNet (or use global VNet peering).

**Problem**: Multi-region architectures require private endpoints in each region.

**Example**:
- Storage account in UK South with private endpoint in UK South VNet
- Workload in Europe West VNet cannot access storage account via private endpoint without inter-region peering

**Mitigation**: Deploy private endpoints per region or use Azure Front Door / Traffic Manager to abstract regional endpoints.

### 5. Cost

Private endpoints cost **£0.01 per hour** (~£7.30/month per endpoint).

**Impact**: Organisations with 100+ PaaS services face significant cost.

**Trade-Off**: Security vs cost. Private endpoints are recommended for production but optional for non-production.

## When to Use Private Endpoints

### Mandatory (No Exceptions)

- Storage accounts containing sensitive data (customer PII, financial records)
- SQL databases and Cosmos DB in production subscriptions
- Key Vaults storing production secrets
- Azure Container Registries in production

### Recommended

- App Services and Function Apps in production (if workload is internal-only)
- Event Hubs and Service Bus queues
- Azure Cognitive Services APIs

### Optional

- Storage accounts in non-production or sandbox subscriptions
- Developer-facing services (e.g., Azure DevOps repos) where public access is intentional

## When NOT to Use Private Endpoints

### 1. Externally Accessible Services

**Scenario**: Web application must be accessible from the public internet.

**Example**: Public-facing website hosted on App Service.

**Solution**: Keep App Service publicly accessible. Use Azure Front Door or Application Gateway with WAF for protection.

**Why**: Private endpoints make the service inaccessible from the internet. If your users are on the internet, private endpoints are counterproductive.

### 2. Development and Sandbox Environments

**Scenario**: Developers need quick access to storage accounts or databases without VPN overhead.

**Solution**: Allow public access with IP whitelisting or Entra ID authentication.

**Trade-Off**: Convenience vs security. Acceptable in non-production.

### 3. Third-Party SaaS Integration

**Scenario**: Third-party SaaS service (e.g., Salesforce, Snowflake) needs to access Azure storage account.

**Problem**: Third-party cannot access private endpoints (they are not on your VNet).

**Solution**: Allow public access with IP whitelisting or shared access signatures (SAS tokens).

**Trade-Off**: Public access is acceptable if third-party IPs are known and stable.

## Private Endpoint Best Practices

### 1. Centralised Private DNS Zones

Create Private DNS Zones in a central subscription (e.g., "Platform-Networking") and link them to all VNets.

**Why**: Avoids duplicate DNS zones per subscription. Simplifies management.

**Example**:
```
Private DNS Zone: privatelink.blob.core.windows.net
Linked to: Hub VNet, Spoke VNet A, Spoke VNet B, etc.
```

### 2. Dedicated Subnet for Private Endpoints

Create a dedicated subnet in each VNet for private endpoints.

**Why**: Simplifies IP address management. NSGs can be applied at subnet level.

**Example**:
```
Subnet: privatelink-subnet (10.0.1.0/24)
Contains: Private endpoints for storage accounts, SQL databases, Key Vaults
```

### 3. Disable Public Network Access

Once private endpoints are deployed, disable public network access on the PaaS service.

**Why**: Prevents accidental public exposure.

**Azure Policy**: Enforce that storage accounts, SQL databases, and Key Vaults have public network access disabled.

**Exception**: Services requiring external partner access may need public access with IP whitelisting.

### 4. Monitor Private Endpoint Health

Private endpoints can become misconfigured or disconnected.

**Monitoring**:
- Azure Monitor alerts on private endpoint connection state
- Weekly validation that DNS resolution points to private IPs, not public IPs

**Failure Detection**: Test connectivity from VMs to PaaS services. If connections fail, investigate DNS or private endpoint configuration.

## Known Failure Modes

### 1. DNS Resolution Fails

**Scenario**: Private DNS Zone is not linked to VNet. DNS queries resolve to public IP instead of private IP.

**Impact**: Applications connect to public endpoint, which may be firewalled or disabled.

**Detection**: Application logs show connection errors or unexpected latency.

**Remediation**: Link Private DNS Zone to VNet. Flush DNS cache on VMs.

### 2. Private Endpoint Deleted

**Scenario**: Private endpoint is accidentally deleted via Terraform or Azure Portal.

**Impact**: Applications can no longer access PaaS service.

**Detection**: Azure Monitor alerts on private endpoint deletion. Application connection failures.

**Remediation**: Re-create private endpoint via Terraform. DNS records may need manual update.

### 3. VNet Peering Breaks DNS

**Scenario**: VNet peering is configured but DNS forwarding is not enabled.

**Impact**: VMs in peered VNet cannot resolve private endpoint DNS records.

**Remediation**: Enable DNS forwarding on VNet peering. Ensure hub VNet has DNS forwarders if using custom DNS.

### 4. IP Address Exhaustion

**Scenario**: Subnet reserved for private endpoints runs out of IP addresses.

**Impact**: New private endpoints cannot be created.

**Detection**: Terraform fails with "no available IPs" error.

**Remediation**: Expand subnet CIDR block or create new subnet for private endpoints.

## Private Endpoints and Hybrid Connectivity

On-premises networks can access Azure private endpoints via ExpressRoute or VPN Gateway.

**Requirement**: On-premises DNS must resolve Azure private endpoint FQDNs to private IPs.

**Solution**:
1. Deploy DNS forwarders in Azure (e.g., in hub VNet)
2. Configure on-premises DNS to forward Azure queries to DNS forwarders
3. DNS forwarders resolve queries using Azure Private DNS Zones

**Complexity**: Requires DNS infrastructure changes on-premises. Not trivial.

**Alternative**: Use Azure Private Link Service to expose on-premises services to Azure. Reverse direction.

## Private Endpoints vs Service Endpoints

**Service Endpoints** are an older technology. They provide Azure backbone routing but do **not** assign a private IP.

**Key Differences**:

| Feature                  | Private Endpoint                | Service Endpoint              |
|--------------------------|----------------------------------|-------------------------------|
| Private IP in VNet       | Yes                              | No                            |
| Public Endpoint Disabled | Can be disabled                  | Public endpoint remains accessible |
| DNS Configuration        | Required                         | Not required                  |
| Cost                     | £0.01/hour per endpoint          | Free                          |
| Regional Restriction     | Regional                         | Regional                      |

**Recommendation**: Use private endpoints for new deployments. Service endpoints are legacy and less secure.

## Security Trade-Off Summary

**Private Endpoints Win**:
- No public internet exposure
- Traffic on Microsoft backbone only
- Network-level access control

**Private Endpoints Lose**:
- Operational complexity (DNS, VPN access)
- Cost (£0.01/hour per endpoint)
- IP address consumption
- Hybrid scenarios require DNS infrastructure changes

**Verdict**: Use private endpoints for production. Operational pain is acceptable for security gain.