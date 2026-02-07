# Drift and Breakage

## State Drift Realities

**Drift**: Infrastructure actual state diverges from Terraform state.

Drift is inevitable. The question is how to detect and remediate it.

## Human-Caused Drift

### 1. Portal Changes

**Scenario**: User modifies resource via Azure Portal despite policy forbidding it.

**Example**:
- Storage account created by Terraform
- User enables public network access via Portal
- Terraform state says public access is disabled
- Reality: public access is enabled

**Detection**: `terraform plan` shows drift.

```
Terraform will perform the following actions:

  # azurerm_storage_account.example will be updated in-place
  ~ resource "azurerm_storage_account" "example" {
      ~ public_network_access_enabled = true -> false
    }
```

**Remediation**: `terraform apply` reverts change. User is notified and trained.

**Prevention**: Azure Policy blocks non-Terraform changes where possible. RBAC limits Portal access.

### 2. Manual Emergency Changes

**Scenario**: Incident occurs. Engineer makes emergency change via Portal (with PIM access).

**Example**:
- App Service down due to misconfigured firewall rule
- Engineer adds firewall exception via Portal to restore service
- Terraform state does not reflect change

**Detection**: `terraform plan` after incident shows drift.

**Remediation**: Engineer codifies change in Terraform post-incident. Submits PR. Change becomes permanent.

**Why This is Acceptable**: Speed is critical during incidents. Codifying changes post-incident is acceptable trade-off.

### 3. Accidental Deletion

**Scenario**: User deletes resource via Portal or CLI.

**Detection**: Terraform state shows resource exists. Azure shows resource does not exist.

**Terraform Behaviour**: `terraform apply` recreates resource.

**Data Loss Risk**: If resource contained data (storage account, database), data is lost unless backup exists.

**Prevention**: Resource locks on critical infrastructure (hub VNet, state storage account). Backups for stateful resources.

## Platform-Caused Drift

### 1. Azure API Changes

**Scenario**: Azure changes default behaviour or API response format.

**Example**:
- Azure changes default TLS version for storage accounts from 1.0 to 1.2
- Terraform state shows TLS 1.0 (old default)
- Azure shows TLS 1.2 (new default)
- Terraform plan shows drift even though no manual change occurred

**Remediation**: Update Terraform code to explicitly set TLS version. Apply change.

**Frequency**: Rare but happens. Azure is a living platform.

### 2. Terraform Provider Bugs

**Scenario**: Terraform provider incorrectly reads resource state.

**Example**:
- Provider bug causes storage account `allow_blob_public_access` to always show as `true` in state even if `false` in Azure
- Every `terraform plan` shows drift
- Applying drift "fix" does nothing (bug prevents correct state read)

**Detection**: Persistent drift that does not remediate.

**Remediation**: Upgrade provider to version with bug fix. Re-run `terraform apply`.

**Known Example (2025)**: Azure Provider 3.75.0 had bug reading NSG rules. Fixed in 3.76.0.

### 3. Azure Resource Manager Delays

**Scenario**: Terraform applies change. ARM accepts change but propagation is delayed.

**Example**:
- Terraform disables public network access on storage account
- ARM returns success
- Terraform refreshes state immediately
- ARM has not yet propagated change to storage account
- State shows public access disabled, Azure shows public access enabled
- Appears as drift

**Behaviour**: Drift self-resolves after ARM propagation completes (typically <5 minutes).

**Remediation**: Wait and re-run `terraform plan`. Drift disappears.

### 4. Azure Policy Auto-Remediation

**Scenario**: Azure Policy detects non-compliance and auto-remediates.

**Example**:
- Terraform creates storage account without diagnostic settings
- Azure Policy (DeployIfNotExists) automatically deploys diagnostic settings
- Terraform state does not know about diagnostic settings
- Drift detected

**Remediation**: Import diagnostic settings into Terraform state or exclude from Terraform management.

**Trade-Off**: Terraform and Azure Policy can conflict. Clear ownership boundaries required.

## Drift Detection

### Scheduled Drift Detection

`terraform plan` runs nightly on all Terraform workspaces.

**Output**:
- No drift: No action required
- Drift detected: Alert sent to platform team with diff

**Why Nightly**: Detects drift quickly without manual intervention.

### Manual Drift Detection

Engineers run `terraform plan` before making changes.

**Why**: Ensures understanding of current state before applying new changes.

## Drift Remediation Strategies

### 1. Auto-Remediate (Preferred)

`terraform apply` automatically reverts drift.

**When to Use**: Drift is clearly incorrect (Portal change violated policy).

**Example**: Public network access enabled on production storage account. Auto-revert.

### 2. Import into State (Sometimes)

Drift is legitimate (emergency change or Azure Policy auto-remediation). Import into Terraform state.

**Command**:
```bash
terraform import azurerm_monitor_diagnostic_setting.example /subscriptions/.../providers/Microsoft.Insights/diagnosticSettings/example
```

**When to Use**: Change was intentional and should be preserved.

**Example**: Azure Policy deployed diagnostic settings. Import into Terraform state.

### 3. Ignore Drift (Rare)

Drift is tolerated because remediation is more disruptive than drift itself.

**Example**:
- Azure changes resource tag format
- Terraform state shows old format
- Azure shows new format
- Re-applying old format causes resource recreation
- Decision: Tolerate drift and update Terraform code to match new format

**When to Use**: Remediation is destructive or overly complex.

## Terraform State Corruption

State corruption is catastrophic. Terraform loses track of infrastructure.

### Causes of Corruption

1. **Concurrent Terraform Runs**

   Two engineers run `terraform apply` simultaneously. State lock prevents this but lock can fail.

2. **Manual State Editing**

   Engineer edits state file manually. Introduces inconsistency.

3. **Storage Account Failure**

   State storage account becomes unavailable during `terraform apply`. State write fails.

### Detection

```bash
terraform plan
```

**Output**: Errors referencing missing resources or invalid state schema.

### Recovery

1. **Restore from Backup**

   Azure Storage Account versioning retains previous state versions.

   ```bash
   # List state versions
   az storage blob list --account-name stplatformtfstate --container-name tfstate
   
   # Download previous version
   az storage blob download --account-name stplatformtfstate --container-name tfstate --name production.tfstate --version-id <version-id>
   ```

2. **Import Existing Resources**

   If backup is unavailable or too old, import resources into new state.

   ```bash
   terraform import azurerm_storage_account.example /subscriptions/.../resourceGroups/.../providers/Microsoft.Storage/storageAccounts/example
   ```

   **Trade-Off**: Time-consuming. Must import every resource managed by Terraform.

3. **Rebuild from Scratch**

   Worst case: Destroy all resources and recreate from Terraform code.

   **Trade-Off**: Downtime. Data loss if backups are incomplete.

## Breakage Scenarios

Terraform changes can break workloads. Breakage is a risk of infrastructure-as-code.

### 1. Destructive Change

**Scenario**: Terraform code change causes resource recreation.

**Example**:
- Storage account name changed in Terraform code
- Azure does not support renaming storage accounts
- Terraform deletes old storage account and creates new one
- Data in old storage account is lost (unless backed up)

**Detection**: `terraform plan` shows resource will be destroyed and recreated.

```
Terraform will perform the following actions:

  # azurerm_storage_account.example must be replaced
-/+ resource "azurerm_storage_account" "example" {
```

**Prevention**:
- Code review catches destructive changes
- Manual approval gate before `terraform apply`
- Resource locks prevent deletion of critical resources

### 2. Configuration Regression

**Scenario**: Terraform code change introduces misconfiguration.

**Example**:
- Engineer accidentally removes firewall rule
- Terraform applies change
- Application loses connectivity

**Detection**: Application monitoring alerts on connectivity failure.

**Remediation**: Revert Terraform change (`git revert` + `terraform apply`).

**Prevention**: Peer review and testing in non-production.

### 3. Provider Bug

**Scenario**: Terraform provider bug causes incorrect configuration.

**Example**: Provider bug sets storage account replication type incorrectly.

**Detection**: Manual validation post-deployment or monitoring alerts.

**Remediation**: Downgrade provider version. Report bug to HashiCorp.

**Prevention**: Pin provider versions. Test updates in non-production.

### 4. Race Condition

**Scenario**: Terraform applies changes in incorrect order due to dependency misspecification.

**Example**:
- Terraform creates VNet and subnet simultaneously
- Subnet creation fails because VNet is not yet fully created
- Terraform apply fails

**Detection**: Terraform error during apply.

**Remediation**: Add explicit dependency (`depends_on`). Re-run apply.

**Prevention**: Test Terraform code in clean environment before production.

## Blast Radius Containment for Terraform Failures

### 1. Separate State Per Environment

Dev, staging, and production have separate Terraform state files.

**Why**: Terraform failure in dev does not affect production.

### 2. Separate State Per Workload

Each workload (subscription or logical group) has its own state file.

**Why**: Terraform failure in one workload does not affect others.

**Trade-Off**: More state files to manage. Complexity increases.

### 3. Resource Locks

Critical resources (hub VNet, state storage account) have CanNotDelete locks.

**Why**: Prevents accidental deletion via Terraform or Portal.

**Limitation**: Does not prevent misconfiguration (only deletion).

## Drift and Breakage Summary

**Drift is inevitable**: Humans, platform changes, and bugs cause drift.

**Detection is mandatory**: Nightly `terraform plan` detects drift.

**Remediation is context-dependent**: Auto-revert, import, or tolerate based on situation.

**Breakage is a risk**: Destructive changes, misconfigurations, and provider bugs cause breakage.

**Mitigation**: Code review, testing, state backups, resource locks, and blast radius containment.

**Terraform is not perfect. It is better than manual infrastructure.**