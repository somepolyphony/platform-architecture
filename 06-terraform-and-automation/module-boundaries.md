# Module Boundaries

## Contract-Driven Module Design

Terraform modules are **contracts** between platform team and delivery teams.

A module defines:
- **Inputs**: What parameters delivery teams must provide
- **Outputs**: What resources or values the module returns
- **Behaviour**: What the module creates and how it enforces policy

Modules are not suggestions. They are the **only** approved way to provision infrastructure.

## Why Modules Exist

### 1. Centralised Security Logic

Security controls are embedded in modules, not duplicated across workloads.

**Example**:
- Storage account module enforces private endpoints, encryption, and logging
- Delivery teams cannot create storage accounts without these controls

**Without Modules**: Every delivery team reimplements security controls. Inconsistencies emerge.

### 2. Reduced Configuration Surface

Modules expose only necessary parameters. Complex configuration is abstracted.

**Example**:
- Storage account has 50+ configuration options in Azure Provider
- Module exposes 10 parameters (name, environment, data classification, etc.)
- Remaining 40+ options are set to secure defaults

**Why**: Reduces chance of misconfiguration. Delivery teams cannot disable security controls.

### 3. Upgradability

When security requirements change, modules are updated once. All consumers inherit updates.

**Example**:
- New requirement: All storage accounts must have soft delete enabled
- Storage account module updated to enforce soft delete
- Next Terraform run auto-enables soft delete for all existing storage accounts

**Without Modules**: Manual remediation across hundreds of storage accounts.

## Module Structure

### Input Variables

Modules expose limited, well-defined inputs.

**Example (Storage Account Module)**:
```hcl
variable "name" {
  description = "Storage account name (must be globally unique)"
  type        = string
}

variable "environment" {
  description = "Environment (dev, staging, production)"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be dev, staging, or production."
  }
}

variable "data_classification" {
  description = "Data classification (public, internal, confidential, restricted)"
  type        = string
}
```

**What is NOT Exposed**:
- Encryption settings (always enabled, no opt-out)
- Public network access (always disabled in production)
- Soft delete (always enabled)

**Why**: Delivery teams should not control security settings.

### Output Values

Modules return resource IDs and connection details.

**Example**:
```hcl
output "storage_account_id" {
  description = "Storage account resource ID"
  value       = azurerm_storage_account.this.id
}

output "primary_blob_endpoint" {
  description = "Primary blob endpoint (private IP)"
  value       = azurerm_storage_account.this.primary_blob_endpoint
}
```

**What is NOT Output**:
- Access keys (stored in Key Vault, not exposed)
- Sensitive configuration (internal module implementation)

### Module Behaviour

Modules enforce policy via Terraform code.

**Example (Storage Account Module)**:
```hcl
resource "azurerm_storage_account" "this" {
  name                     = var.name
  location                 = var.location
  resource_group_name      = var.resource_group_name
  account_tier             = "Standard"
  account_replication_type = var.environment == "production" ? "GRS" : "LRS"
  
  # Security controls (non-negotiable)
  enable_https_traffic_only       = true
  min_tls_version                 = "TLS1_2"
  allow_nested_items_to_be_public = false
  public_network_access_enabled   = var.environment == "production" ? false : true
  
  # Soft delete
  blob_properties {
    delete_retention_policy {
      days = 90
    }
  }
}

# Private endpoint (production only)
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
    is_manual_connection           = false
  }
}
```

**Key Points**:
- HTTPS enforced (no HTTP)
- TLS 1.2 minimum
- Public blob access disabled
- Private endpoint in production
- Geo-redundant storage in production

**Delivery teams cannot bypass these controls without editing the module (requires platform team approval).**

## Module Versioning

Modules are versioned using semantic versioning (semver).

**Format**: `MAJOR.MINOR.PATCH`

**Example**: `1.2.3`

### Version Increments

- **MAJOR**: Breaking changes (input variable renamed, output removed, behaviour changed)
- **MINOR**: New features (new input variable added, new resource type supported)
- **PATCH**: Bug fixes (no API changes)

**Example**:
- `1.0.0`: Initial release
- `1.1.0`: Added support for lifecycle management rules
- `1.1.1`: Fixed bug in soft delete configuration
- `2.0.0`: Renamed `environment` variable to `env` (breaking change)

### Version Pinning

Delivery teams pin module versions in their Terraform code.

**Example**:
```hcl
module "storage" {
  source  = "internal/azurerm/storage-account"
  version = "1.2.3"
  # ...
}
```

**Why**: Prevents unexpected behaviour from module updates.

**Update Process**: Delivery teams update module versions during planned maintenance windows.

## Module Registry

Modules are published to an internal Terraform registry.

**Options**:
1. **Terraform Cloud/Enterprise**: Commercial offering with private registry
2. **Git-based registry**: Modules stored in Git repository with tags
3. **Azure DevOps Artifacts**: Terraform module support added in 2024

**Our Choice**: Git-based registry (cost-effective, simple).

**URL Format**:
```
git::https://github.com/org/terraform-modules.git//azurerm/storage-account?ref=v1.2.3
```

## Module Ownership

### Platform Team Responsibilities

- Create and maintain modules
- Enforce security and compliance requirements
- Version modules and publish to registry
- Document module usage and examples
- Respond to module bug reports and feature requests

### Delivery Team Responsibilities

- Consume modules (not write raw Terraform)
- Pin module versions
- Update module versions during maintenance windows
- Report bugs and request features

### Security Team Responsibilities

- Review module code for security issues
- Approve new modules and major version changes
- Audit module usage quarterly

## Module Boundaries (What to Modularise)

### Always a Module

- Storage accounts
- Virtual networks
- SQL databases
- App Services
- Key Vaults
- Azure Firewall
- Private endpoints

**Why**: High security impact. Standardisation mandatory.

### Sometimes a Module

- Virtual machines (may vary significantly by workload)
- Container instances
- Function Apps

**Why**: Workload-specific requirements may not fit a single module.

**Approach**: Provide reference modules. Allow customisation.

### Never a Module

- Resource groups (simple, low variance)
- Tags (inline in code)
- RBAC assignments (context-specific, not generic)

**Why**: Overhead of modularity outweighs benefits.

## Module Design Principles

### 1. Secure by Default

Modules enforce security by default. Insecure options are not exposed.

**Example**: Storage accounts always have encryption and HTTPS. No option to disable.

### 2. Minimal Configuration Surface

Modules expose only necessary inputs. Complex configuration is hidden.

**Trade-Off**: Flexibility vs security. Security wins.

### 3. Fail Loudly

Modules validate inputs and fail fast with clear error messages.

**Example**:
```hcl
variable "environment" {
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be dev, staging, or production. Got: ${var.environment}"
  }
}
```

### 4. Document Trade-Offs

Module documentation explains why certain options are not exposed.

**Example**:
```markdown
## Why Public Network Access is Not Configurable

Public network access is disabled in production by design. This prevents
accidental data exfiltration. If your workload requires public access,
request an exception via the exception process.
```

## Module Testing

Modules are tested before publication.

### Unit Tests

Terraform validation and formatting checks.

```bash
terraform fmt -check
terraform validate
```

### Integration Tests

Deploy module to test environment. Validate resources created correctly.

**Tools**: Terratest (Go-based testing framework).

**Example**:
```go
func TestStorageAccountModule(t *testing.T) {
    terraformOptions := &terraform.Options{
        TerraformDir: "../modules/storage-account",
        Vars: map[string]interface{}{
            "name":        "testsa",
            "environment": "dev",
        },
    }
    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)
    
    // Validate storage account exists and has correct configuration
    storageAccountID := terraform.Output(t, terraformOptions, "storage_account_id")
    assert.NotEmpty(t, storageAccountID)
}
```

### Security Tests

Azure Policy compliance checks.

**Example**: Storage account must have private endpoint in production.

## Module Documentation

Every module includes README.md with:

- **Purpose**: What the module creates
- **Inputs**: Required and optional variables
- **Outputs**: What the module returns
- **Examples**: Working code snippets
- **Security Rationale**: Why certain controls are enforced
- **Limitations**: What the module does not support

**Example README.md**:
```markdown
# Storage Account Module

## Purpose

Creates Azure Storage Account with security baselines enforced.

## Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| name | string | Yes | - | Storage account name |
| environment | string | Yes | - | dev, staging, or production |

## Outputs

| Name | Description |
|------|-------------|
| storage_account_id | Resource ID |

## Security Rationale

- HTTPS enforced (no HTTP)
- TLS 1.2 minimum
- Private endpoint in production
- Public network access disabled in production

## Limitations

- Geo-redundant storage only in production (cost trade-off)
- Does not support static website hosting (security risk)
```

## Known Issues with Modules

### 1. Module Inflexibility

**Problem**: Module does not support edge case workload requirement.

**Example**: Workload requires public network access for external integration. Module blocks it in production.

**Solution**: Request exception or create custom module variant (requires security review).

**Frequency**: <5% of workloads.

### 2. Module Bug

**Problem**: Module has a bug. Affects all consumers.

**Example**: Module incorrectly configures soft delete retention period.

**Detection**: Integration tests catch some bugs. Others discovered in production.

**Remediation**: Patch released within 24-48 hours. All consumers update module version.

### 3. Module Version Lag

**Problem**: Delivery teams do not update module versions. Use outdated modules.

**Impact**: Miss security improvements or bug fixes.

**Mitigation**: Quarterly audit of module versions. Enforce updates for critical patches.

### 4. Module Documentation Outdated

**Problem**: Module code changes but documentation is not updated.

**Impact**: Delivery teams use incorrect examples or misunderstand behaviour.

**Mitigation**: Documentation is part of code review. Outdated docs block PR approval.

## Module Boundaries Summary

- **Modules are contracts**: Define inputs, outputs, and behaviour
- **Secure by default**: Security controls non-negotiable
- **Versioned and tested**: Semver versioning with integration tests
- **Centralised ownership**: Platform team creates and maintains
- **Minimal configuration**: Only necessary inputs exposed

Modules are the **enforcement mechanism** for platform policy. Without modules, policy is optional.