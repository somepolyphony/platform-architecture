# Terraform Landing Zone

## Conceptual Link to Enforcement Repositories

This document provides a **conceptual** overview of how Terraform-based landing zone enforcement repositories align with the architectural decisions in this repository.

**This is not an implementation guide.** For implementation details, refer to the actual Terraform repository.

## What is a Landing Zone?

A landing zone is the **baseline infrastructure** provisioned when a new Azure subscription is created.

It includes:
- Management group assignment
- Azure Policy assignments
- RBAC assignments
- Networking (spoke VNet, NSGs, route tables)
- Logging (diagnostic settings, Log Analytics connection)
- Tagging (owner, cost centre, environment)

**Purpose**: Ensure every subscription starts with security and compliance controls pre-applied.

## Terraform Landing Zone Repository Structure

A typical Terraform landing zone repository is organised as:

```
terraform-landing-zone/
├── management-groups/
│   ├── root.tf               # Tenant Root management group
│   ├── platform.tf           # Platform management group
│   ├── production.tf         # Production management group
│   └── non-production.tf     # Non-Production management group
├── policies/
│   ├── definitions/          # Custom policy definitions
│   ├── assignments/          # Policy assignments at mgmt group scope
│   └── exemptions/           # Time-boxed policy exemptions
├── networking/
│   ├── hub-vnet.tf           # Hub VNet configuration
│   ├── spoke-vnet-module/    # Reusable spoke VNet module
│   └── firewall.tf           # Azure Firewall rules
├── logging/
│   ├── log-analytics.tf      # Central Log Analytics workspace
│   └── diagnostic-settings.tf # Enforce diagnostic settings
├── defender/
│   ├── defender-plans.tf     # Enable Defender on subscriptions
│   └── defender-alerts.tf    # Configure Defender alerts
├── subscriptions/
│   ├── provisioning.tf       # Subscription creation automation
│   └── rbac.tf               # RBAC assignments
└── modules/
    ├── storage-account/      # Storage account module (see module-boundaries.md)
    ├── virtual-machine/      # VM module
    └── sql-database/         # SQL Database module
```

## Alignment with Architecture Decisions

### Management Groups (02-subscriptions-and-management-groups/)

**Architectural Decision**: Management group hierarchy with root, platform, production, non-production, and sandbox groups.

**Terraform Implementation**:
```hcl
# management-groups/root.tf
resource "azurerm_management_group" "root" {
  display_name = "Tenant Root"
}

# Policies assigned here apply to all subscriptions
resource "azurerm_management_group_policy_assignment" "require_encryption" {
  name                 = "require-encryption-at-rest"
  management_group_id  = azurerm_management_group.root.id
  policy_definition_id = azurerm_policy_definition.encryption.id
}
```

**Reference**: [hierarchy-design.md](../02-subscriptions-and-management-groups/hierarchy-design.md)

### Azure Policy (04-security-and-defender/)

**Architectural Decision**: Policies enforced at management group level. Exemptions time-boxed.

**Terraform Implementation**:
```hcl
# policies/assignments/production.tf
resource "azurerm_management_group_policy_assignment" "block_public_ips" {
  name                 = "block-public-ips"
  management_group_id  = azurerm_management_group.production.id
  policy_definition_id = azurerm_policy_definition.block_public_ips.id
}

# policies/exemptions/ecommerce-exception.tf
resource "azurerm_policy_exemption" "ecommerce_public_ip" {
  name                 = "exception-ecommerce-publicip"
  policy_assignment_id = azurerm_management_group_policy_assignment.block_public_ips.id
  exemption_category   = "Waiver"
  expires_on           = "2026-03-07T00:00:00Z"
  metadata = jsonencode({
    ticket_id = "INC123456"
  })
}
```

**Reference**: [policy-authority.md](../04-security-and-defender/policy-authority.md), [time-boxed-exceptions.md](../07-exceptions-and-risk-acceptance/time-boxed-exceptions.md)

### Networking (03-networking-guardrails/)

**Architectural Decision**: Hub-spoke model with Azure Firewall. Private endpoints for PaaS services.

**Terraform Implementation**:
```hcl
# networking/hub-vnet.tf
resource "azurerm_virtual_network" "hub" {
  name                = "vnet-hub-uks"
  location            = "uksouth"
  resource_group_name = azurerm_resource_group.platform_networking.name
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_firewall" "hub" {
  name                = "afw-hub-uks"
  location            = "uksouth"
  resource_group_name = azurerm_resource_group.platform_networking.name
  sku_name            = "AZFW_VNet"
  sku_tier            = "Standard"
  threat_intel_mode   = "Alert"
}

# networking/spoke-vnet-module/main.tf
module "spoke_vnet" {
  source = "./modules/spoke-vnet"
  
  name                = var.spoke_name
  address_space       = var.address_space
  hub_vnet_id         = azurerm_virtual_network.hub.id
  firewall_private_ip = azurerm_firewall.hub.ip_configuration[0].private_ip_address
}
```

**Reference**: [hub-spoke-rationale.md](../03-networking-guardrails/hub-spoke-rationale.md), [private-endpoints.md](../03-networking-guardrails/private-endpoints.md)

### Defender for Cloud (04-security-and-defender/)

**Architectural Decision**: Defender plans enabled on all production subscriptions.

**Terraform Implementation**:
```hcl
# defender/defender-plans.tf
resource "azurerm_security_center_subscription_pricing" "servers" {
  tier          = "Standard"
  resource_type = "VirtualMachines"
}

resource "azurerm_security_center_subscription_pricing" "storage" {
  tier          = "Standard"
  resource_type = "StorageAccounts"
}
```

**Reference**: [defender-scope.md](../04-security-and-defender/defender-scope.md)

### Terraform Modules (06-terraform-and-automation/)

**Architectural Decision**: Reusable modules enforce security baselines.

**Terraform Implementation**:
```hcl
# modules/storage-account/main.tf
resource "azurerm_storage_account" "this" {
  name                     = var.name
  location                 = var.location
  resource_group_name      = var.resource_group_name
  account_tier             = "Standard"
  account_replication_type = var.environment == "production" ? "GRS" : "LRS"
  
  # Security baselines (non-configurable)
  enable_https_traffic_only       = true
  min_tls_version                 = "TLS1_2"
  allow_nested_items_to_be_public = false
  public_network_access_enabled   = var.environment == "production" ? false : true
}

# Private endpoint in production
resource "azurerm_private_endpoint" "this" {
  count               = var.environment == "production" ? 1 : 0
  name                = "${var.name}-pe"
  location            = var.location
  resource_group_name = var.resource_group_name
  subnet_id           = var.private_endpoint_subnet_id
  
  private_service_connection {
    name                           = "${var.name}-psc"
    private_connection_resource_id = azurerm_storage_account.this.id
    subresource_names              = ["blob"]
  }
}
```

**Reference**: [module-boundaries.md](../06-terraform-and-automation/module-boundaries.md)

## Landing Zone Deployment Workflow

1. **Management Groups and Policies**: Deployed first. Foundation for all subscriptions.
2. **Hub Networking**: Deployed second. Shared infrastructure for spokes.
3. **Logging and Defender**: Deployed third. Monitoring and security telemetry.
4. **Subscription Provisioning**: Delivery teams request subscriptions. Platform team provisions via Terraform.
5. **Spoke VNets**: Created per subscription. Peered to hub automatically.

**Timeline**: Platform foundation (steps 1-3) deployed once. Subscription provisioning (steps 4-5) on-demand.

## State Management

Landing zone Terraform state is stored in Azure Storage Account with:
- Encryption at rest
- Soft delete (90-day retention)
- Versioning enabled
- Access restricted to platform team service principal

**State File Location**:
```
stplatformtfstate (storage account)
└── tfstate (container)
    ├── management-groups.tfstate
    ├── networking.tfstate
    ├── logging.tfstate
    ├── defender.tfstate
    └── subscriptions.tfstate
```

**Why Separate State Files**: Blast radius containment. Corruption in one state file does not affect others.

**Reference**: [drift-and-breakage.md](../06-terraform-and-automation/drift-and-breakage.md)

## Drift Detection and Remediation

Landing zone Terraform runs nightly drift detection:

```bash
# Scheduled pipeline
terraform plan -detailed-exitcode
```

**Exit Codes**:
- 0: No changes (no drift)
- 1: Error
- 2: Changes detected (drift)

**On Drift Detection**:
- Alert sent to platform team with diff
- Platform team reviews and remediates (auto-apply or manual investigation)

**Reference**: [drift-and-breakage.md](../06-terraform-and-automation/drift-and-breakage.md)

## CI/CD Pipeline

Landing zone Terraform deploys via CI/CD pipeline (Azure DevOps or GitHub Actions).

**Pipeline Steps**:
1. Checkout code
2. `terraform init` (initialise backend and providers)
3. `terraform fmt -check` (validate formatting)
4. `terraform validate` (syntax check)
5. `terraform plan -out=tfplan` (generate plan)
6. Manual approval gate (platform lead reviews plan)
7. `terraform apply tfplan` (apply changes)
8. Post-deployment validation (Azure Policy compliance check)

**Reference**: [why-terraform.md](../06-terraform-and-automation/why-terraform.md)

## Relationship to This Repository

| This Repository (Architecture)          | Terraform Repository (Implementation)       |
|-----------------------------------------|---------------------------------------------|
| Defines **why** decisions exist         | Implements **how** decisions are enforced   |
| Explains trade-offs and constraints     | Contains Terraform code and modules         |
| Owned by architecture and security team | Owned by platform team                      |
| Updated quarterly or as needed          | Updated continuously (PRs merged weekly)    |
| No executable code                      | Executable Terraform code                   |

**This repository is the source of truth for intent.** Terraform repository must align with decisions here.

If Terraform implementation diverges, either:
1. Terraform is incorrect (update Terraform to align)
2. Architecture decision is outdated (update this repo, then update Terraform)

**Divergence is unacceptable.** Regular audits ensure alignment.

## Further Reading

For implementation details, refer to the actual Terraform landing zone repository (internal link).

For conceptual understanding, refer to the architectural sections in this repository:
- [Management Groups](../02-subscriptions-and-management-groups/hierarchy-design.md)
- [Networking](../03-networking-guardrails/hub-spoke-rationale.md)
- [Security](../04-security-and-defender/policy-authority.md)
- [Terraform Philosophy](../06-terraform-and-automation/why-terraform.md)