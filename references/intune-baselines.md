# Intune Baselines

## Conceptual Link to Enforcement Repositories

This document provides a **conceptual** overview of how Microsoft Intune baseline enforcement repositories align with the architectural decisions in this repository.

**This is not an implementation guide.** For implementation details, refer to the actual Intune configuration repository or Microsoft documentation.

## What are Intune Baselines?

Intune baselines are **collections of security settings** applied to corporate-managed devices (Windows, macOS, iOS, Android).

Baselines include:
- Windows security settings (BitLocker, Firewall, UAC, Credential Guard)
- Microsoft Edge configuration (SmartScreen, extension restrictions)
- Microsoft 365 Apps configuration (macro blocking, external content restrictions)
- Microsoft Defender for Endpoint settings (real-time protection, ASR rules)
- Windows Update for Business configuration (update deferral, enforcement)

**Purpose**: Standardise device configuration across the organisation. Prevent configuration drift. Enforce security posture.

## Intune Configuration Repository Structure

A typical Intune configuration repository is organised as:

```
intune-baselines/
├── configuration-profiles/
│   ├── windows-security-baseline.json
│   ├── edge-baseline.json
│   ├── office-baseline.json
│   └── defender-baseline.json
├── compliance-policies/
│   ├── windows-compliance.json
│   ├── macos-compliance.json
│   └── ios-compliance.json
├── conditional-access/
│   ├── require-compliant-device.json
│   └── require-mfa.json
├── windows-update-for-business/
│   ├── pilot-ring.json
│   └── production-ring.json
├── app-protection-policies/
│   ├── ios-app-protection.json
│   └── android-app-protection.json
└── assignments/
    ├── all-corporate-windows-devices.json
    ├── all-corporate-macos-devices.json
    └── developer-workstations.json
```

## Alignment with Architecture Decisions

### Device Trust Model (05-endpoint-and-intune/)

**Architectural Decision**: Corporate-managed devices must meet compliance requirements. Non-compliant devices blocked by Conditional Access.

**Implementation (Intune Compliance Policy - Windows)**:
```json
{
  "displayName": "Windows Compliance Policy - Production",
  "assignments": [{
    "target": {
      "groupId": "all-corporate-windows-devices"
    }
  }],
  "scheduledActionsForRule": [{
    "ruleName": "PasswordRequired",
    "scheduledActionConfigurations": [{
      "actionType": "block",
      "gracePeriodHours": 72
    }]
  }],
  "osMinimumVersion": "10.0.22000.0",  // Windows 11 22H2
  "bitLockerEnabled": true,
  "secureBootEnabled": true,
  "codeIntegrityEnabled": true,
  "firewallEnabled": true
}
```

**Result**: Devices not meeting compliance are marked non-compliant. Conditional Access blocks sign-in.

**Reference**: [device-trust-model.md](../05-endpoint-and-intune/device-trust-model.md)

### Baseline Philosophy (05-endpoint-and-intune/)

**Architectural Decision**: Baselines enforce standardisation. Exceptions are rare and time-boxed.

**Implementation (Windows Security Baseline)**:
```json
{
  "displayName": "Windows Security Baseline - Production",
  "assignments": [{
    "target": {
      "groupId": "all-corporate-windows-devices"
    }
  }],
  "settings": {
    "bitLocker": {
      "enabled": true,
      "encryptionMethod": "xtsAes256",
      "requireDeviceEncryption": true
    },
    "firewall": {
      "domainProfile": "enabled",
      "privateProfile": "enabled",
      "publicProfile": "enabled",
      "globalPortRules": "blockAllInbound"
    },
    "defenderAntivirus": {
      "realTimeProtectionEnabled": true,
      "cloudProtectionLevel": "high",
      "tamperProtectionEnabled": true
    },
    "credentialGuard": {
      "enabled": true,
      "configuration": "enabledWithUEFILock"
    },
    "localAdminAccounts": {
      "enabled": false
    }
  }
}
```

**Key Settings**:
- BitLocker enforced (no user opt-out)
- Windows Firewall enabled on all profiles
- Defender antivirus cannot be disabled by user
- Credential Guard protects cached credentials
- Local admin accounts disabled

**Reference**: [baseline-philosophy.md](../05-endpoint-and-intune/baseline-philosophy.md)

### WUfB Guardrails (05-endpoint-and-intune/)

**Architectural Decision**: Windows Update for Business enforces patching. Pilot ring receives updates first. Production ring deferred 7 days.

**Implementation (Pilot Ring)**:
```json
{
  "displayName": "Windows Update - Pilot Ring",
  "assignments": [{
    "target": {
      "groupId": "pilot-devices"
    }
  }],
  "featureUpdatesDeferralPeriodInDays": 0,
  "qualityUpdatesDeferralPeriodInDays": 0,
  "driverUpdatesDeferralPeriodInDays": 14,
  "autoRestartBeforeDeadline": true,
  "deadlineForFeatureUpdatesInDays": 7,
  "deadlineForQualityUpdatesInDays": 3,
  "gracePeriodInDays": 3
}
```

**Implementation (Production Ring)**:
```json
{
  "displayName": "Windows Update - Production Ring",
  "assignments": [{
    "target": {
      "groupId": "all-corporate-windows-devices",
      "exclusionGroupIds": ["pilot-devices"]
    }
  }],
  "featureUpdatesDeferralPeriodInDays": 60,
  "qualityUpdatesDeferralPeriodInDays": 7,
  "driverUpdatesDeferralPeriodInDays": 30,
  "autoRestartBeforeDeadline": true,
  "deadlineForFeatureUpdatesInDays": 14,
  "deadlineForQualityUpdatesInDays": 7,
  "gracePeriodInDays": 3
}
```

**Behaviour**:
- Pilot devices receive updates immediately
- Production devices receive feature updates after 60 days
- Production devices receive security patches after 7 days
- Automatic restart enforced after grace period expires

**Reference**: [wufb-guardrails.md](../05-endpoint-and-intune/wufb-guardrails.md)

### Conditional Access Integration (01-identity-and-access/)

**Architectural Decision**: Conditional Access policies require compliant devices for privileged access.

**Implementation (Conditional Access - Require Compliant Device for Azure Portal)**:
```json
{
  "displayName": "Require Compliant Device for Azure Portal",
  "state": "enabled",
  "conditions": {
    "applications": {
      "includeApplications": ["Microsoft Azure Management"]
    },
    "users": {
      "includeUsers": ["All"]
    },
    "devices": {
      "deviceFilter": {
        "mode": "include",
        "rule": "device.isCompliant -eq true"
      }
    }
  },
  "grantControls": {
    "operator": "AND",
    "builtInControls": [
      "compliantDevice",
      "mfa"
    ]
  }
}
```

**Result**: Users cannot access Azure Portal from non-compliant devices.

**Reference**: [identity-boundaries.md](../01-identity-and-access/identity-boundaries.md)

## Intune App Protection Policies (BYOD)

For personal devices (BYOD), Intune App Protection Policies restrict data leakage.

**Implementation (iOS App Protection)**:
```json
{
  "displayName": "iOS App Protection Policy - BYOD",
  "assignments": [{
    "target": {
      "groupId": "all-users"
    }
  }],
  "apps": [
    "com.microsoft.outlook",
    "com.microsoft.teams"
  ],
  "dataProtection": {
    "allowedInboundDataTransferSources": "managedApps",
    "allowedOutboundDataTransferDestinations": "managedApps",
    "organizationalCredentialsRequired": true,
    "printBlocked": true,
    "saveAsBlocked": true,
    "disableAppPinIfDevicePinIsSet": false,
    "minimumPinLength": 6,
    "pinRequired": true
  }
}
```

**Key Restrictions**:
- Copy/paste between corporate and personal apps blocked
- Cannot save corporate files to personal cloud storage
- Cannot print corporate emails
- PIN required to access corporate apps

**Reference**: [device-trust-model.md](../05-endpoint-and-intune/device-trust-model.md)

## Intune Reporting and Compliance

Intune provides compliance reports showing:
- Device compliance status (compliant, non-compliant, not evaluated)
- Configuration profile deployment status (success, error, not applicable)
- Update deployment status (pending, installed, failed)

**Report Types**:
1. **Device Compliance Dashboard**: Real-time view of compliant vs non-compliant devices
2. **Configuration Profile Status**: Deployment success rate per profile
3. **Windows Update Status**: Update installation progress per update ring
4. **App Protection Policy Status**: BYOD app protection enforcement

**Use Case**: Monthly compliance review. Security team validates all corporate devices are compliant.

## Intune Baseline Exemptions

Some devices require baseline exemptions (e.g., developer workstations, kiosks).

**Process**: See [exception-process.md](../07-exceptions-and-risk-acceptance/exception-process.md).

**Implementation**:
- Device moved to different Intune group with relaxed baseline
- Exemption documented in ServiceNow
- Exemption time-boxed and reviewed quarterly

**Example (Developer Workstation Exception)**:
```json
{
  "displayName": "Windows Security Baseline - Developer Workstations",
  "assignments": [{
    "target": {
      "groupId": "developer-workstations"
    }
  }],
  "settings": {
    "localAdminAccounts": {
      "enabled": true  // Exception: Developers need local admin for debugging
    }
    // All other settings same as production baseline
  }
}
```

**Reference**: [exception-process.md](../07-exceptions-and-risk-acceptance/exception-process.md)

## Intune Automation

Intune configuration is managed via:
1. **Microsoft Graph API**: Automate policy creation and assignment
2. **PowerShell**: Bulk operations (e.g., create 100 configuration profiles)
3. **Terraform (azuread provider)**: Limited support for Intune in 2026

**Example (PowerShell - Create Compliance Policy)**:
```powershell
$policy = @{
    "@odata.type" = "#microsoft.graph.windows10CompliancePolicy"
    displayName = "Windows Compliance - Production"
    osMinimumVersion = "10.0.22000.0"
    bitLockerEnabled = $true
    secureBootEnabled = $true
}

Invoke-MgGraphRequest -Method POST -Uri "https://graph.microsoft.com/v1.0/deviceManagement/deviceCompliancePolicies" -Body ($policy | ConvertTo-Json)
```

## Relationship to This Repository

| This Repository (Architecture)          | Intune Repository (Implementation)          |
|-----------------------------------------|---------------------------------------------|
| Defines **why** baselines exist         | Implements **how** baselines are enforced   |
| Explains trade-offs and exemptions      | Contains Intune configuration (JSON)        |
| Owned by architecture and security team | Owned by platform team                      |
| Updated quarterly or as needed          | Updated continuously (baselines tuned)      |
| No executable code                      | Executable configuration (JSON, PowerShell) |

**This repository is the source of truth for intent.** Intune configuration must align with decisions here.

## Further Reading

For implementation details, refer to:
- Microsoft Intune documentation (Microsoft Learn)
- Microsoft Graph API documentation (Intune endpoints)
- Internal Intune configuration repository (if available)

For conceptual understanding, refer to:
- [Device Trust Model](../05-endpoint-and-intune/device-trust-model.md)
- [Baseline Philosophy](../05-endpoint-and-intune/baseline-philosophy.md)
- [WUfB Guardrails](../05-endpoint-and-intune/wufb-guardrails.md)