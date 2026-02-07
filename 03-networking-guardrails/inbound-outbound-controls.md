# Inbound and Outbound Controls

## Explicit Trust Boundaries

Network security is enforced at both ingress (inbound traffic) and egress (outbound traffic).

Default-deny is the only acceptable posture.

## Inbound Controls

Inbound traffic originates from the internet, on-premises networks, or other Azure VNets.

### Inbound from Internet

**Default Policy**: Block all inbound traffic from the internet.

**Exceptions** (explicit allow):

1. **User-Facing Web Applications**

   Public-facing websites and APIs must accept inbound HTTPS.

   **Control**: Azure Front Door or Application Gateway with Web Application Firewall (WAF).

   **Why**: WAF inspects HTTP/HTTPS traffic for SQL injection, XSS, and other attacks. Provides DDoS protection.

2. **VPN Gateway**

   Remote users connect via VPN for corporate access.

   **Control**: Azure VPN Gateway with certificate-based or Entra ID authentication. MFA enforced.

   **Why**: VPN provides encrypted tunnel. Traffic enters hub VNet and is subject to firewall inspection.

3. **ExpressRoute (On-Premises Connectivity)**

   On-premises networks connect via dedicated circuit.

   **Control**: ExpressRoute Gateway with Border Gateway Protocol (BGP) filtering.

   **Why**: Private connectivity. Traffic does not traverse public internet.

**What is NOT Permitted**:
- Direct RDP or SSH from internet to VMs (Azure Bastion required instead)
- Public IP addresses on VMs (blocked via Azure Policy)
- Inbound traffic on non-standard ports without firewall rule

### Inbound from Other Azure VNets

**Default Policy**: Block all inter-VNet traffic unless explicitly peered and firewall rules permit.

**Hub-Spoke Model**:
- Spoke VNets peer to hub only
- Traffic between spokes routes through Azure Firewall in hub
- Firewall enforces allow/deny rules

**Example**:
```
Spoke A (10.1.0.0/16) → Hub Firewall → Spoke B (10.2.0.0/16)
```

Firewall rule must exist for traffic to flow. Without rule, traffic is blocked.

### Inbound from On-Premises

**Default Policy**: Block all inbound traffic from on-premises.

**Exceptions**:
- Specific services exposed via private endpoints (e.g., SQL databases, storage accounts)
- Hub VNet services (DNS forwarders, management tools)

**Control**: Azure Firewall or Network Security Groups enforce allow rules.

**Why**: On-premises is not fully trusted. Assume on-premises networks can be compromised.

## Outbound Controls

Outbound traffic originates from Azure VNets and targets the internet, other VNets, or on-premises.

### Outbound to Internet

**Default Policy**: Block all outbound traffic to the internet.

**Why**: Prevent data exfiltration, command-and-control communication, and malware downloads.

**Exceptions** (explicit allow):

1. **Operating System Updates**

   VMs require access to Windows Update, Ubuntu apt repositories, Red Hat YUM repositories.

   **Control**: Azure Firewall allows FQDN-based rules (e.g., `*.windowsupdate.microsoft.com`).

2. **Package Managers**

   Application dependencies pulled from npm, PyPI, Maven Central, NuGet.

   **Control**: Azure Firewall allows FQDN-based rules for trusted package repositories.

3. **Azure Services**

   VMs require access to Azure Storage, Azure DevOps, Key Vault.

   **Control**: Private endpoints preferred. If public endpoints used, Azure Firewall allows traffic to Azure service tags.

4. **External APIs**

   Applications integrate with third-party APIs (e.g., Stripe, Twilio, Salesforce).

   **Control**: Azure Firewall allows specific FQDNs or IP ranges.

**What is NOT Permitted**:
- Unrestricted internet access
- Access to file-sharing sites (e.g., personal Dropbox, Google Drive)
- Access to anonymising proxies or Tor
- Outbound traffic to known malicious IPs (threat intelligence feeds block these automatically)

### Outbound to Other Azure VNets

**Default Policy**: Block all inter-VNet traffic unless explicitly peered and firewall rules permit.

Same as inbound. Hub-spoke model enforces firewall transit.

### Outbound to On-Premises

**Default Policy**: Block all outbound traffic to on-premises.

**Exceptions**:
- Access to on-premises Active Directory (if hybrid identity is required)
- Access to on-premises databases or file shares (during migration periods only)

**Control**: Azure Firewall enforces allow rules for specific on-premises IPs or subnets.

**Why**: Outbound to on-premises should be rare. Cloud-native architectures minimise on-premises dependencies.

## Network Security Groups (NSGs)

NSGs enforce subnet-level and NIC-level inbound/outbound rules.

### NSG Best Practices

1. **Default-Deny**

   All NSGs start with implicit deny-all rule. Exceptions are added explicitly.

2. **Least Privilege**

   Open only the ports and protocols required. No "allow all" rules.

   **Example**:
   - Web tier subnet: Allow inbound 443 (HTTPS) from Application Gateway. Block all other inbound.
   - App tier subnet: Allow inbound 8080 (application port) from web tier only. Block all other inbound.
   - Database tier subnet: Allow inbound 1433 (SQL) from app tier only. Block all other inbound.

3. **Service Tags**

   Use Azure service tags instead of IP addresses where possible.

   **Example**: Allow outbound to `Storage` service tag instead of hardcoding storage account IPs.

4. **Application Security Groups (ASGs)**

   Group VMs by role (e.g., web tier, app tier, database tier). NSG rules reference ASGs instead of individual IPs.

   **Why**: Simplifies NSG management. Adding a VM to web tier ASG automatically applies web tier NSG rules.

### NSG Limitations

NSGs are **stateless** (return traffic requires explicit allow rule in older NSG configurations, but modern NSGs track connection state).

NSGs do **not** inspect application-layer traffic. They filter by IP, port, and protocol only.

**Example**: NSG can allow HTTPS (port 443) but cannot inspect HTTP headers or block SQL injection.

**Mitigation**: Use Azure Firewall or WAF for application-layer inspection.

## Azure Firewall

Azure Firewall provides centralised network and application-layer filtering in the hub VNet.

### Firewall Rule Types

1. **Network Rules**

   Filter by IP address, port, and protocol.

   **Example**: Allow outbound TCP port 443 to `*.microsoft.com`.

2. **Application Rules**

   Filter by FQDN (fully qualified domain name).

   **Example**: Allow outbound HTTPS to `*.githubusercontent.com`.

   **Why**: FQDN-based rules are resilient to IP address changes. GitHub's IPs change frequently; FQDN rules do not require updates.

3. **DNAT Rules**

   Destination NAT translates public IPs to private IPs.

   **Example**: Inbound traffic to public IP `20.50.20.30` is translated to private IP `10.1.0.4` (VM in spoke VNet).

   **Use Case**: Exposing specific services to the internet whilst keeping VMs on private IPs.

### Firewall Default Behaviour

**Default**: Deny all traffic.

All firewall rules are explicit allows. There is no "implicit allow".

### Threat Intelligence

Azure Firewall integrates with Microsoft Threat Intelligence feeds.

**Behaviour**: Traffic to known malicious IPs or FQDNs is automatically blocked and logged.

**Examples of Blocked Traffic**:
- Command-and-control servers for malware
- Phishing sites
- Cryptomining pools

**Override**: Threat intelligence alerts can be set to "Alert only" instead of "Alert and deny". Not recommended for production.

## Trust Boundaries Summary

### Internet → Azure

**Trust Level**: Untrusted.

**Controls**:
- Azure Front Door or Application Gateway with WAF
- Azure Firewall threat intelligence
- No public IPs on VMs
- Bastion for admin access

### Azure → Internet

**Trust Level**: Minimally trusted. Allow only necessary outbound traffic.

**Controls**:
- Azure Firewall with FQDN-based application rules
- Block file-sharing sites, anonymising proxies
- Threat intelligence blocks known-bad destinations

### On-Premises → Azure

**Trust Level**: Partially trusted. Assume on-premises can be compromised.

**Controls**:
- ExpressRoute or VPN with MFA
- Azure Firewall enforces inbound rules
- Private endpoints limit exposure

### Azure → On-Premises

**Trust Level**: Minimally trusted. Minimise dependencies.

**Controls**:
- Azure Firewall enforces outbound rules
- Hybrid identity (Entra ID Connect) requires careful scoping

### Azure VNet → Azure VNet

**Trust Level**: Untrusted by default. Treat spoke VNets as separate security zones.

**Controls**:
- Hub-spoke model with Azure Firewall transit
- NSGs enforce least-privilege traffic rules

## Known Failure Modes

### 1. NSG Misconfiguration

**Scenario**: NSG rule accidentally allows overly broad traffic (e.g., `0.0.0.0/0` on inbound SSH).

**Impact**: Attack surface increased. Unauthorised access possible.

**Mitigation**: Azure Policy blocks NSG rules with overly permissive source ranges. Quarterly NSG audit.

### 2. Firewall Rule Creep

**Scenario**: Firewall rules accumulate over time. Unused rules are not removed.

**Impact**: Policy complexity increases. Risk of unintended allow rules.

**Mitigation**: Quarterly firewall rule audit. Remove unused rules. Document business justification for all rules.

### 3. Application Bypasses Firewall

**Scenario**: Application uses public endpoint instead of private endpoint. Traffic bypasses Azure Firewall.

**Impact**: Firewall rules not enforced. Data exfiltration possible.

**Mitigation**: Azure Policy enforces private endpoints. Public network access disabled on PaaS services where possible.

### 4. Threat Intelligence False Positive

**Scenario**: Legitimate service is flagged by threat intelligence. Traffic blocked.

**Impact**: Application connectivity fails.

**Mitigation**: Review threat intelligence alerts. Allowlist legitimate FQDNs if false positive confirmed.

## Inbound/Outbound Controls Checklist

- [ ] Default-deny on all NSGs
- [ ] Azure Firewall enforces hub-spoke transit
- [ ] No public IPs on VMs
- [ ] Bastion for admin access
- [ ] WAF on internet-facing applications
- [ ] FQDN-based firewall rules for outbound traffic
- [ ] Threat intelligence enabled on Azure Firewall
- [ ] Private endpoints for PaaS services
- [ ] Quarterly NSG and firewall rule audit

Network controls are not optional. They are the last line of defence after identity and policy failures.