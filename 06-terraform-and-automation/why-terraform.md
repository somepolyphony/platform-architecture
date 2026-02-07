# Why Terraform

## Why Terraform is Used

Azure infrastructure is provisioned via Terraform, not the Azure Portal or ARM templates.

This is not a technical preference. This is a security and operational requirement.

## Problems with Manual Infrastructure

### 1. Not Auditable

Portal changes are logged in Azure Activity Logs but do not capture **intent**.

**Example**:
- Activity log: "User created storage account"
- Missing: Why was it created? Who approved it? What workload does it support?

**Terraform**: Every change is in Git with commit message, pull request, and approval workflow.

### 2. Not Repeatable

Manual portal deployments are one-off. Recreating the same infrastructure requires clicking through the portal again.

**Problem**: Configuration drift is inevitable. Production and non-production environments diverge.

**Terraform**: Infrastructure is defined as code. Redeployment is idempotent (same input = same output).

### 3. Not Peer-Reviewed

Portal changes are made by individuals without review.

**Problem**: Mistakes (public endpoint enabled, firewall misconfigured) are discovered after deployment, not before.

**Terraform**: All changes require pull request approval. At least one peer reviews before merge.

### 4. Not Testable

Portal changes cannot be tested in a separate environment before applying to production.

**Terraform**: Changes are applied to dev/staging environments first. Validated before production.

### 5. Not Recoverable

Portal changes cannot be rolled back easily.

**Example**: User deletes a resource group by mistake. Recovery requires restoring from backup (if backup exists).

**Terraform**: State is tracked. Resources can be recreated from code. Rollback is `git revert` + `terraform apply`.

## Why Terraform Over ARM Templates or Bicep

### ARM Templates

**Problems**:
- JSON syntax is verbose and error-prone
- No state management (no drift detection)
- Limited abstraction (no modules or reusable components)

**When ARM is Appropriate**: Azure Policy definitions (native ARM support is better than Terraform).

### Bicep

**Problems**:
- Azure-specific (cannot manage Entra ID, Microsoft 365, or multi-cloud)
- Limited ecosystem compared to Terraform
- State management is weaker than Terraform

**When Bicep is Appropriate**: Azure-only shops without Entra ID or multi-cloud requirements.

### Terraform

**Advantages**:
- Multi-provider (Azure, Entra ID, Microsoft 365, AWS, GCP)
- Strong state management (drift detection, plan/apply workflow)
- Mature ecosystem (modules, providers, CI/CD integrations)
- Declarative syntax (HCL is more readable than JSON)

**Disadvantages**:
- Requires separate state storage (Azure Storage Account)
- Learning curve for teams unfamiliar with HCL
- Some Azure features lag behind ARM/Bicep support

**Verdict**: Terraform is the best tool for Azure + Entra ID + Microsoft 365 platform management.

## What Terraform Should Control

### Platform Infrastructure (Always Terraform)

- Management groups
- Subscriptions
- Hub and spoke VNets
- Azure Firewall
- Azure Bastion
- Azure Policy assignments
- RBAC assignments
- Private DNS Zones
- Log Analytics workspaces

**Why**: Platform infrastructure changes infrequently. Requires tight control and audit trail.

### Workload Infrastructure (Terraform Preferred)

- Resource groups
- Virtual machines
- App Services
- Storage accounts
- SQL databases
- Networking (subnets, NSGs)

**Why**: Workload infrastructure changes frequently. Terraform provides repeatability and disaster recovery.

### Entra ID and Microsoft 365 (Terraform Where Possible)

- Entra ID groups
- Service principals
- Conditional Access policies (limited Terraform support in 2026)
- Microsoft 365 tenant settings (limited Terraform support in 2026)

**Why**: Identity configuration is critical. Should be version-controlled and peer-reviewed.

**Limitation**: Terraform support for Entra ID and Microsoft 365 is incomplete. Some configuration must be done via Graph API or Portal.

## What Terraform Should Never Control

### 1. Break-Glass Accounts

**Why**: Break-glass accounts are the recovery path when Terraform fails or Entra ID is unavailable. Creating them via Terraform creates circular dependency.

**Alternative**: Break-glass accounts created manually and validated quarterly.

### 2. Terraform State Storage Account

**Why**: Terraform state is stored in a storage account. If Terraform manages that storage account, deletion creates unrecoverable state loss.

**Alternative**: State storage account created manually and protected with CanNotDelete lock.

### 3. Secrets and Credentials

**Why**: Secrets in Terraform state are stored in plaintext. State file compromise exposes all secrets.

**Alternative**: Secrets stored in Key Vault. Terraform references secrets by ID, not value.

**Example**:
```hcl
# Bad: Secret in Terraform code
resource "azurerm_sql_server" "example" {
  administrator_login_password = "P@ssw0rd123"
}

# Good: Secret in Key Vault
data "azurerm_key_vault_secret" "sql_admin_password" {
  name         = "sql-admin-password"
  key_vault_id = azurerm_key_vault.example.id
}

resource "azurerm_sql_server" "example" {
  administrator_login_password = data.azurerm_key_vault_secret.sql_admin_password.value
}
```

### 4. Defender for Cloud Alert Rules

**Why**: Defender alert rules are managed by security team, not platform team. Separate ownership.

**Alternative**: Defender rules configured via Portal or Graph API.

### 5. Emergency Changes During Outages

**Why**: Terraform requires state access, peer review, and CI/CD pipeline. Too slow during outages.

**Alternative**: Emergency changes made via Portal with PIM access. Changes codified in Terraform post-incident.

## Terraform State Management

### Remote State

Terraform state is stored in Azure Storage Account, not locally.

**Why**:
- Local state is not shared across team members
- Local state is not backed up
- Local state enables drift (different team members have different views of infrastructure)

**Configuration**:
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-platform-terraform-state"
    storage_account_name = "stplatformtfstate"
    container_name       = "tfstate"
    key                  = "production.tfstate"
  }
}
```

### State Locking

Terraform uses Azure Storage Account blob leases for state locking.

**Why**: Prevents concurrent Terraform runs from corrupting state.

**Behaviour**: If two engineers run `terraform apply` simultaneously, second run is blocked until first completes.

### State Encryption

Terraform state is encrypted at rest (Azure Storage Account encryption).

**Why**: State contains sensitive data (resource IDs, some configuration values).

**Limitation**: State does **not** encrypt secrets. Secrets must be in Key Vault.

### State Backup

Terraform state is backed up via Azure Storage Account soft delete and versioning.

**Configuration**:
- Soft delete: 90-day retention
- Versioning: Enabled (all state versions retained)

**Recovery**: If state is corrupted or deleted, previous version can be restored.

## Terraform Workflow

### 1. Local Development

Engineer writes Terraform code on local machine or codespace.

**Validation**:
```bash
terraform init
terraform fmt      # Format code
terraform validate # Syntax check
terraform plan     # Preview changes
```

No `terraform apply` locally. Deployment via CI/CD only.

### 2. Pull Request

Engineer commits code to feature branch and opens pull request.

**PR Checks**:
- Terraform format check (`terraform fmt -check`)
- Terraform validation (`terraform validate`)
- Terraform plan (output posted as PR comment)
- Peer review by platform engineer

**Approval**: At least one approver required.

### 3. CI/CD Pipeline

On merge to main branch, CI/CD pipeline runs:

1. `terraform init` (initialise providers and backend)
2. `terraform plan -out=tfplan` (generate execution plan)
3. Manual approval gate (platform lead reviews plan)
4. `terraform apply tfplan` (apply changes)

**Why Manual Approval**: Prevents accidental deployment of destructive changes.

### 4. Post-Deployment Validation

After `terraform apply`, CI/CD pipeline validates:
- All resources created successfully
- Terraform state is consistent
- Azure Policy compliance checks pass

**If Validation Fails**: Alert sent to platform team. Manual investigation required.

## Terraform and Azure Portal Coexistence

**Reality**: Some users will make changes via Portal despite policy forbidding it.

**Drift Detection**: Terraform plan detects drift (Portal changes vs Terraform state).

**Example**:
```
Terraform will perform the following actions:

  # azurerm_storage_account.example will be updated in-place
  ~ resource "azurerm_storage_account" "example" {
      ~ allow_blob_public_access = true -> false
    }
```

**Interpretation**: Someone enabled public blob access via Portal. Terraform will revert it.

**Remediation**: `terraform apply` reverts Portal changes. Offending user is notified.

## Known Risks with Terraform

### 1. State File Corruption

**Scenario**: Terraform state is corrupted or deleted.

**Impact**: Terraform loses track of resources. Cannot manage infrastructure.

**Recovery**:
- Restore state from backup (Azure Storage Account versioning)
- Worst case: Import existing resources into Terraform (`terraform import`)

**Mitigation**: State backups with 90-day retention.

### 2. Terraform Provider Bug

**Scenario**: Terraform provider has a bug. Applies incorrect configuration.

**Impact**: Resources misconfigured. Potential outage or security gap.

**Mitigation**: Pin provider versions. Test updates in non-production first.

**Example (Real Bug, 2025)**: Azure Provider 3.80.0 incorrectly deleted subnet route tables. Fixed in 3.81.0.

### 3. Accidental Deletion

**Scenario**: Engineer mistakenly deletes resource in Terraform code. CI/CD applies deletion.

**Impact**: Production resource deleted.

**Mitigation**:
- Terraform plan output reviewed in PR
- Manual approval gate before apply
- Resource locks on critical infrastructure (hub VNet, state storage account)

**Recovery**: Recreate resource from Terraform code. Restore data from backup if necessary.

### 4. Secret in State File

**Scenario**: Engineer hardcodes secret in Terraform. Secret stored in state file.

**Impact**: State file compromise exposes secret.

**Mitigation**:
- Code review catches hardcoded secrets
- Secrets scanning in CI/CD (e.g., trufflehog, git-secrets)
- Secrets stored in Key Vault only

## Terraform Governance

### Module Standards

All Terraform code must use approved modules from internal module registry.

**Why**: Prevents duplication and security inconsistencies.

**Example**:
```hcl
module "storage_account" {
  source  = "internal/azurerm/storage-account"
  version = "1.2.0"
  # Module enforces private endpoints, encryption, logging
}
```

### Code Review Requirements

All Terraform changes require:
- Peer review by platform engineer
- Security review for changes to RBAC, networking, or Azure Policy
- Approval by platform lead for production changes

### Terraform Version Pinning

Terraform and provider versions are pinned in code.

**Example**:
```hcl
terraform {
  required_version = "1.9.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "4.0.1"
    }
  }
}
```

**Why**: Prevents unexpected behaviour from version upgrades.

**Update Process**: Terraform versions updated quarterly after testing in non-production.

## Why Terraform Summary

- **Auditability**: Every change in Git with approval workflow
- **Repeatability**: Infrastructure as code. Redeployment is idempotent.
- **Peer Review**: All changes reviewed before deployment
- **Testability**: Changes applied to non-production first
- **Recoverability**: State backups and version control enable rollback

**Trade-Off**: Learning curve and operational overhead vs security and consistency.

**Verdict**: Terraform is mandatory for platform infrastructure. Operational overhead is acceptable.