# Baseline Philosophy

## Why Baselines Exist

Security baselines standardise device configuration across the organisation.

Without baselines, every device is a unique configuration. Unique configurations are ungovernable.

## What is a Baseline?

A baseline is a **collection of configuration settings** applied to all devices in a specific category.

**Examples**:
- Windows 10/11 security baseline
- Microsoft Edge baseline
- Microsoft 365 Apps baseline
- Defender for Endpoint baseline

Baselines are provided by Microsoft and adapted by the platform team.

## Why Microsoft's Baselines Are Not Sufficient

Microsoft publishes security baselines as **recommendations**, not mandates.

**Problems with Microsoft Baselines**:
1. **Too permissive**: Designed for broad compatibility, not maximum security
2. **Not organisation-specific**: Do not account for your threat model or risk appetite
3. **Rarely updated**: Baselines lag behind new threats
4. **Optional features**: Critical hardening is marked as "recommended", not required

**Example**:
- Microsoft baseline recommends enabling BitLocker but does not enforce it
- Microsoft baseline allows local admin accounts (with restrictions)
- Microsoft baseline allows RDP (with strong authentication)

**Our Approach**: Start with Microsoft baseline. Harden further based on threat model.

## Baseline Categories

### 1. Windows Security Baseline

**Purpose**: Harden Windows OS against common attacks.

**Key Settings**:
- BitLocker full-disk encryption: **Enforced** (not recommended)
- Windows Firewall: **Enforced** on all network profiles
- UAC (User Account Control): **Enabled** and cannot be disabled by user
- SMBv1 (legacy file sharing protocol): **Disabled** (known vulnerabilities)
- Local admin accounts: **Disabled** except for break-glass scenarios
- Credential Guard: **Enabled** (protects cached credentials from extraction)
- Windows Defender Antivirus: **Enabled** with real-time protection

**Exceptions**: None for corporate-managed devices in production.

### 2. Microsoft Edge Baseline

**Purpose**: Harden browser against phishing, malware downloads, and data exfiltration.

**Key Settings**:
- SmartScreen: **Enabled** (blocks known phishing sites)
- Download restrictions: Block potentially unwanted applications (PUAs)
- Extension installation: Restricted to approved extensions only
- Password manager: Disabled (corporate passwords must be in KeePass or password vault)
- Autofill: Disabled for credit cards and addresses
- InPrivate mode: Allowed (does not bypass corporate controls)

**Why Strict**: Browser is the most common attack vector. Users click links. Phishing is inevitable.

### 3. Microsoft 365 Apps Baseline

**Purpose**: Harden Office apps (Word, Excel, PowerPoint) against macro-based attacks.

**Key Settings**:
- Macros: Blocked unless digitally signed by trusted publisher
- ActiveX controls: Blocked
- External content (embedded objects): Blocked unless from trusted sources
- AutoSave: Enabled to OneDrive or SharePoint (not local disk)

**Trade-Off**: Some legacy Office documents with macros will not work. Users must request exceptions.

### 4. Defender for Endpoint Baseline

**Purpose**: Ensure Defender for Endpoint is configured for maximum detection.

**Key Settings**:
- Real-time protection: **Enabled** and cannot be disabled by user
- Cloud-delivered protection: **Enabled** (behavioural analysis via Microsoft cloud)
- Automatic sample submission: **Enabled** (suspicious files sent to Microsoft for analysis)
- Tamper protection: **Enabled** (prevents malware from disabling Defender)
- Attack surface reduction (ASR) rules: **Enabled** (blocks common attack techniques)

**Exceptions**: Developers may request ASR rule exemptions for specific tools (e.g., reverse engineering tools).

### 5. Windows Update for Business Baseline

**Purpose**: Ensure devices receive security patches without excessive disruption.

**Key Settings**:
- Feature updates: Deferred 60 days after release (stability)
- Quality updates: Installed within 7 days of release (security)
- Update rings: Pilot group (5%) receives updates first, production group (95%) receives updates after 7 days
- Update enforcement: Updates automatically installed and devices automatically restarted during maintenance windows
- Deferral: Users can defer restart for up to 3 days

**Why Not Immediate Updates**: Zero-day patches occasionally break applications. Pilot group absorbs risk.

## Baseline Enforcement

### How Baselines Are Applied

Baselines are delivered via Microsoft Intune as **configuration profiles**.

**Process**:
1. Platform team creates configuration profile in Intune
2. Profile is assigned to device group (e.g., "All Corporate Windows Devices")
3. Device checks in with Intune (every 8 hours or on demand)
4. Intune applies settings
5. Device reports compliance

**Timeline**: New devices receive baselines within 1 hour. Existing devices receive updates within 8 hours.

### Baseline Conflicts

**Scenario**: User manually changes a setting that conflicts with baseline.

**Example**: User disables Windows Firewall to "fix" a connectivity issue.

**Behaviour**: Intune detects conflict. Automatically reverts setting to baseline value. User sees "This setting is managed by your organisation."

**Why**: Baselines are **enforced**, not recommended. Users cannot opt out.

### Baseline Exemptions

**Rare but Necessary**: Some devices require exemptions due to technical incompatibility.

**Examples**:
- Developer workstations requiring local admin access (for debugging)
- Kiosks or single-purpose devices with restricted functionality

**Process**:
1. User requests exemption via ServiceNow
2. Request includes device, setting, justification, and duration
3. Security team reviews and approves or rejects
4. If approved, device is moved to different Intune group with relaxed baseline
5. Exemption is time-boxed and reviewed quarterly

**Frequency**: Exemptions should be <1% of devices. If higher, baseline is too restrictive and must be revised.

## Why Exceptions Are Dangerous

Every exemption weakens the security posture.

### 1. Exemptions Create Inconsistency

**Scenario**: 10 devices have exemptions for different settings.

**Impact**: Security team cannot assume baseline controls apply to all devices. Audit complexity increases.

### 2. Exemptions Are Forgotten

**Scenario**: Temporary exemption granted for 30 days. User forgets to request renewal. Exemption persists indefinitely.

**Impact**: Device remains non-compliant long-term.

**Mitigation**: Exemptions are time-boxed and automatically expire. User must re-request if still needed.

### 3. Exemptions Are Abused

**Scenario**: User requests exemption for "developer workstation" but uses device for email and browsing.

**Impact**: Device is over-privileged. Risk of compromise increases.

**Mitigation**: Exemptions are audited quarterly. Abuse results in revocation.

### 4. Exemptions Enable Lateral Movement

**Scenario**: Developer device with local admin is compromised. Attacker uses local admin to disable Defender and pivot to other devices.

**Impact**: Single compromised device affects multiple workloads.

**Mitigation**: Exempted devices are network-isolated where possible. Assume exempted devices are higher risk.

## Baseline Updates

Baselines are **not static**. They evolve as threats change and technology improves.

### Update Cadence

Baselines are reviewed and updated:
- **Quarterly**: Routine review of Microsoft baseline updates
- **Ad-hoc**: When new vulnerability or attack technique is discovered

**Example**:
- Microsoft releases new Windows security feature (e.g., Kernel DMA Protection)
- Platform team evaluates feature
- If beneficial, feature is added to baseline
- Updated baseline deployed to all devices

### Baseline Change Process

1. Platform team proposes baseline change
2. Change tested on pilot group (10 devices)
3. If no issues after 1 week, rolled out to production
4. If issues detected, change is rolled back and refined

**No emergency baseline changes.** Baseline changes are planned and tested.

### Communicating Baseline Changes

Users are notified of significant baseline changes via email or Teams.

**Examples of Significant Changes**:
- New policy that blocks previously allowed action (e.g., blocking macros in Office)
- New software installation required (e.g., new VPN client)

**Examples of Non-Significant Changes**:
- Tightening existing control (e.g., reducing password expiry from 90 to 60 days)
- Enabling new Defender feature

## Baseline Failures

### 1. Baseline Breaks Application

**Scenario**: Baseline change breaks line-of-business application.

**Example**: Office macro blocking prevents Excel-based reporting tool from functioning.

**Detection**: User reports issue. IT investigates and identifies baseline conflict.

**Remediation**: Application vendor updates app to remove macro dependency, or user requests exemption.

**Timeline**: Exemption granted within 24 hours. Permanent fix within 30 days.

### 2. Baseline Not Applied

**Scenario**: Device does not receive baseline due to Intune misconfiguration or network issue.

**Detection**: Intune reports device as non-compliant. Conditional Access blocks device.

**Remediation**: Device reconnects to internet. Intune re-applies baseline. Device becomes compliant.

**Timeline**: Automatic remediation within 8 hours.

### 3. User Bypasses Baseline

**Scenario**: User with local admin disables baseline-enforced control.

**Detection**: Intune detects non-compliance. Conditional Access blocks device.

**Remediation**: Intune re-applies baseline. If user repeatedly bypasses, local admin is revoked.

**Timeline**: Automatic remediation within 8 hours.

### 4. Baseline is Too Restrictive

**Scenario**: Baseline blocks legitimate business activity.

**Example**: Baseline blocks file downloads from internet. User cannot download vendor software.

**Detection**: High volume of exemption requests for same setting.

**Remediation**: Baseline is revised to relax overly restrictive control.

**Timeline**: Baseline revision within 2-4 weeks.

## Baseline vs Policy

**Baseline**: Technical configuration settings applied to devices.

**Policy**: Organisational rules and procedures.

**Example**:
- **Baseline**: BitLocker enabled on all drives
- **Policy**: All corporate data must be encrypted at rest

Baselines enforce policy. Policy defines intent.

## Baseline Governance

### 1. Baseline Ownership

**Owner**: Platform team (technical implementation).

**Approver**: Security team (risk acceptance).

### 2. Baseline Documentation

All baselines are documented with:
- **Why**: Security rationale
- **What**: Specific settings and values
- **Trade-Off**: What breaks or becomes harder
- **Exceptions**: How to request exemption

Documentation is stored in platform architecture repository (this repo).

### 3. Baseline Audit

Baselines are audited quarterly.

**Audit Questions**:
- Are baselines still applied to all devices?
- Are exemptions still justified?
- Have new threats emerged requiring baseline updates?
- Are baselines aligned with Microsoft's latest recommendations?

## Baseline Philosophy Summary

- **Standardisation over freedom**: All devices follow the same baseline
- **Enforcement over recommendation**: Baselines are mandatory, not optional
- **Exceptions are rare**: <1% of devices should have exemptions
- **Baselines evolve**: Quarterly reviews and updates
- **Testing before deployment**: All baseline changes tested on pilot group

Baselines are the foundation of endpoint security. Without them, devices are ungovernable.