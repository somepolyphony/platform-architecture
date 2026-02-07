# Defender Hardening

## Conceptual Link to Enforcement Repositories

This document provides a **conceptual** overview of how Microsoft Defender hardening enforcement repositories align with the architectural decisions in this repository.

**This is not an implementation guide.** For implementation details, refer to the actual Defender configuration repository or Microsoft documentation.

## What is Defender Hardening?

Defender hardening refers to the **configuration and tuning** of Microsoft Defender for Cloud and Microsoft Defender for Endpoint to maximise threat detection whilst minimising false positives.

Hardening includes:
- Enabling all relevant Defender plans (Servers, Storage, SQL, Containers)
- Configuring Defender settings (alert thresholds, automated responses)
- Tuning Defender recommendations (suppressing false positives)
- Integrating Defender with Sentinel (SIEM) for correlation
- Enforcing Defender for Endpoint on all corporate devices

**Purpose**: Ensure Defender provides maximum security value without overwhelming security operations team with noise.

## Defender Hardening Repository Structure

A typical Defender configuration repository is organised as:

```
defender-hardening/
├── defender-for-cloud/
│   ├── plans.tf                 # Enable Defender plans per subscription
│   ├── settings.tf              # Defender settings (email contacts, auto-provisioning)
│   ├── recommendations.tf       # Suppress false-positive recommendations
│   └── workflow-automation.tf   # Forward alerts to Sentinel
├── defender-for-endpoint/
│   ├── intune-policies/         # Intune configuration profiles for Defender
│   ├── asr-rules.json           # Attack Surface Reduction rules
│   ├── exclusions.json          # Defender exclusions (paths, processes)
│   └── tamper-protection.json   # Tamper protection settings
├── sentinel-integration/
│   ├── data-connectors.tf       # Connect Defender to Sentinel
│   ├── analytics-rules.tf       # Sentinel alert rules (correlation logic)
│   └── playbooks/               # Automated response playbooks
└── compliance-reporting/
    ├── iso27001.tf              # ISO 27001 compliance dashboard
    └── cis-azure.tf             # CIS Azure Benchmark dashboard
```

## Alignment with Architecture Decisions

### Defender Scope (04-security-and-defender/)

**Architectural Decision**: Defender for Cloud mandatory on all production subscriptions. Defender for Endpoint mandatory on all corporate devices.

**Implementation (Terraform)**:
```hcl
# defender-for-cloud/plans.tf
resource "azurerm_security_center_subscription_pricing" "servers" {
  tier          = "Standard"
  resource_type = "VirtualMachines"
}

resource "azurerm_security_center_subscription_pricing" "storage" {
  tier          = "Standard"
  resource_type = "StorageAccounts"
}

resource "azurerm_security_center_subscription_pricing" "sql" {
  tier          = "Standard"
  resource_type = "SqlServers"
}
```

**Implementation (Intune - Defender for Endpoint)**:
```json
{
  "displayName": "Defender for Endpoint Configuration",
  "assignments": [{
    "target": {
      "groupId": "all-corporate-windows-devices"
    }
  }],
  "settings": {
    "realTimeProtectionEnabled": true,
    "cloudProtectionLevel": "high",
    "automaticSampleSubmission": true,
    "tamperProtectionEnabled": true
  }
}
```

**Reference**: [defender-scope.md](../04-security-and-defender/defender-scope.md)

### Policy Authority (04-security-and-defender/)

**Architectural Decision**: Defender recommendations are reviewed by security team. False positives suppressed. True positives remediated.

**Implementation**:
```hcl
# defender-for-cloud/recommendations.tf
resource "azurerm_security_center_assessment_policy" "suppress_vm_no_backup" {
  display_name = "VMs should have backup configured"
  severity     = "Medium"
  
  # Suppress for dev VMs (backed up manually)
  metadata = jsonencode({
    category = "Compute",
    suppressionRules = [{
      resourceGroup = "rg-dev-*"
    }]
  })
}
```

**Reference**: [policy-authority.md](../04-security-and-defender/policy-authority.md)

### Enforcement vs Monitor (04-security-and-defender/)

**Architectural Decision**: Defender recommendations are monitored (not enforced). Azure Policy enforces preventative controls.

**Example**:
- **Defender**: Detects VM without endpoint protection (alert generated)
- **Azure Policy**: Deploys Defender for Endpoint agent via DeployIfNotExists policy (enforcement)

**Defender's Role**: Detective control. Flags non-compliance.

**Azure Policy's Role**: Preventative control. Blocks or auto-remediates.

**Reference**: [enforcement-vs-monitor.md](../04-security-and-defender/enforcement-vs-monitor.md)

### Device Trust Model (05-endpoint-and-intune/)

**Architectural Decision**: Defender for Endpoint provides real-time threat detection on corporate devices.

**Implementation (Intune - Attack Surface Reduction Rules)**:
```json
{
  "displayName": "ASR Rules - Production",
  "assignments": [{
    "target": {
      "groupId": "all-corporate-windows-devices"
    }
  }],
  "asrRules": [
    {
      "id": "BE9BA2D9-53EA-4CDC-84E5-9B1EEEE46550",
      "action": "Block",
      "description": "Block executable content from email client and webmail"
    },
    {
      "id": "D4F940AB-401B-4EFC-AADC-AD5F3C50688A",
      "action": "Block",
      "description": "Block all Office applications from creating child processes"
    },
    {
      "id": "3B576869-A4EC-4529-8536-B80A7769E899",
      "action": "Block",
      "description": "Block Office applications from creating executable content"
    }
  ]
}
```

**Reference**: [device-trust-model.md](../05-endpoint-and-intune/device-trust-model.md)

## Defender for Cloud Workflow Automation

Defender alerts are forwarded to Sentinel for correlation and automated response.

**Implementation**:
```hcl
# defender-for-cloud/workflow-automation.tf
resource "azurerm_security_center_automation" "forward_to_sentinel" {
  name                = "forward-defender-alerts-to-sentinel"
  location            = "uksouth"
  resource_group_name = "rg-security"
  
  source {
    event_source = "Alerts"
    rule_set {
      rule {
        property_path  = "properties.severity"
        operator       = "Equals"
        expected_value = "High"
        property_type  = "String"
      }
    }
  }
  
  action {
    type = "LogicApp"
    resource_id = azurerm_logic_app_workflow.sentinel_ingestion.id
  }
}
```

**Workflow**:
1. Defender detects suspicious activity (e.g., unusual VM access)
2. Alert generated in Defender for Cloud
3. Workflow automation triggers Logic App
4. Logic App forwards alert to Sentinel
5. Sentinel correlates alert with other telemetry (Entra ID sign-ins, network logs)
6. Sentinel creates incident and assigns to SOC analyst

**Reference**: [defender-scope.md](../04-security-and-defender/defender-scope.md)

## Defender for Endpoint Exclusions

Some legitimate software triggers false positives. Exclusions are configured to reduce noise.

**Implementation (Intune)**:
```json
{
  "displayName": "Defender Exclusions - Development Tools",
  "assignments": [{
    "target": {
      "groupId": "developer-workstations"
    }
  }],
  "exclusions": {
    "paths": [
      "C:\\Program Files\\Docker\\Docker\\resources",
      "C:\\Users\\*\\.vscode",
      "C:\\Users\\*\\AppData\\Local\\Temp\\terraform"
    ],
    "processes": [
      "node.exe",
      "python.exe",
      "terraform.exe"
    ]
  }
}
```

**Why Exclusions Are Needed**: Development tools generate large numbers of file writes and process executions. Defender flags these as suspicious. Exclusions prevent alert fatigue.

**Risk**: Malware can masquerade as excluded processes. Exclusions must be scoped to specific user groups (developers only).

**Reference**: [baseline-philosophy.md](../05-endpoint-and-intune/baseline-philosophy.md)

## Defender Compliance Reporting

Defender provides compliance dashboards for ISO 27001, PCI-DSS, CIS Azure Benchmarks.

**Implementation**:
```hcl
# compliance-reporting/iso27001.tf
resource "azurerm_security_center_assessment_policy" "iso27001" {
  display_name = "ISO 27001:2013 Compliance"
  description  = "Assess Azure resources against ISO 27001:2013 controls"
  
  # Automatically assigns policy to all subscriptions in management group
  management_group_id = azurerm_management_group.production.id
}
```

**Output**: Compliance dashboard showing pass/fail status for each control.

**Use Case**: External audits (ISO 27001 certification). Auditors review Defender compliance dashboard as evidence.

**Reference**: [threat-model.md](../00-platform-charter/threat-model.md)

## Defender Limitations

### 1. No Application-Level Security

Defender does not inspect application code for vulnerabilities (SQL injection, XSS).

**Solution**: Use SAST/DAST tools in CI/CD pipelines (Snyk, SonarQube, Veracode).

### 2. Alert Fatigue

High volume of low-severity alerts overwhelms SOC team.

**Solution**: Tune Defender recommendations. Suppress false positives. Prioritise high-severity alerts.

### 3. Delayed Detection

Defender detects threats **after** they occur (detective control, not preventative).

**Solution**: Combine with preventative controls (Azure Policy, NSGs, private endpoints).

**Reference**: [defender-scope.md](../04-security-and-defender/defender-scope.md)

## Relationship to This Repository

| This Repository (Architecture)          | Defender Repository (Implementation)        |
|-----------------------------------------|---------------------------------------------|
| Defines **why** Defender is mandatory   | Implements **how** Defender is configured   |
| Explains trade-offs and limitations     | Contains Defender configuration code        |
| Owned by architecture and security team | Owned by security team                      |
| Updated quarterly or as needed          | Updated continuously (tuning ongoing)       |
| No executable code                      | Executable configuration (Terraform, JSON)  |

**This repository is the source of truth for intent.** Defender configuration must align with decisions here.

## Further Reading

For implementation details, refer to:
- Microsoft Defender for Cloud documentation (Microsoft Learn)
- Microsoft Defender for Endpoint documentation (Microsoft Learn)
- Internal Defender configuration repository (if available)

For conceptual understanding, refer to:
- [Defender Scope](../04-security-and-defender/defender-scope.md)
- [Policy Authority](../04-security-and-defender/policy-authority.md)
- [Enforcement vs Monitor](../04-security-and-defender/enforcement-vs-monitor.md)