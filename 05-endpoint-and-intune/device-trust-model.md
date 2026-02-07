# Device Trust Model

## Corporate vs Personal Trust Assumptions

Endpoints (laptops, mobile devices) are the weakest link in cloud security.

Device compromise enables credential theft, data exfiltration, and lateral movement.

The platform must explicitly define which devices are trusted and why.

## Device Trust Tiers

### Tier 1: Corporate-Managed, Compliant Devices

**Characteristics**:
- Enrolled in Microsoft Intune
- Compliance policy enforced (encryption, antivirus, OS patches)
- Microsoft Defender for Endpoint installed
- Windows Hello for Business or FIDO2 authentication
- Conditional Access enforced

**Trust Level**: High (but not absolute).

**What These Devices Can Access**:
- Azure Portal
- Microsoft 365 (Outlook, Teams, SharePoint)
- Internal applications (via Entra ID authentication)
- Privileged access (PIM activation, break-glass accounts)

**Why We Trust Them**:
- Platform team controls configuration
- Security baselines enforced
- Real-time threat detection via Defender for Endpoint

**Why We Don't Fully Trust Them**:
- User-initiated malware installation is possible
- Phishing can compromise credentials
- Lost/stolen devices can be accessed if BitLocker is bypassed

### Tier 2: Corporate-Managed, Non-Compliant Devices

**Characteristics**:
- Enrolled in Intune
- Compliance policy **not** met (e.g., OS is outdated, BitLocker disabled)

**Trust Level**: Low.

**What These Devices Can Access**:
- Nothing. Conditional Access blocks sign-in until compliance is restored.

**Why**: Non-compliant devices are assumed compromised.

**User Experience**: User sees "Your device does not meet security requirements. Contact IT."

**Remediation**: User updates OS, enables BitLocker, or waits for Intune to auto-remediate.

### Tier 3: Personal Devices (BYOD)

**Characteristics**:
- Not enrolled in Intune
- Owned by employee
- Used for work via mobile app (Outlook, Teams) or browser

**Trust Level**: Very low.

**What These Devices Can Access**:
- Email and Teams (via Intune App Protection Policies)
- Web-based applications (via browser with Conditional Access)

**What These Devices Cannot Access**:
- Azure Portal (blocked by Conditional Access)
- Privileged access (PIM activation blocked)
- Internal applications requiring device compliance

**Why We Don't Trust Them**:
- Platform team has no control over configuration
- No visibility into installed software
- User may have local admin access (malware installation risk)

**Trade-Off**: BYOD increases productivity (users can check email on personal phones) but creates security risk.

### Tier 4: Unmanaged Devices

**Characteristics**:
- Not enrolled in Intune
- Unknown ownership (could be contractor device, internet café, etc.)

**Trust Level**: None.

**What These Devices Can Access**:
- Nothing. Conditional Access blocks all access.

**Exceptions**:
- External partners may have temporary Conditional Access bypass (see [exception-handling.md](../01-identity-and-access/exception-handling.md))
- Break-glass accounts bypass Conditional Access (emergencies only)

## Device Compliance Requirements

Intune compliance policies define minimum security posture.

Devices must meet all requirements to be considered "compliant".

### Windows Devices

**Mandatory Requirements**:
- Windows 10 version 22H2 or later (or Windows 11)
- BitLocker enabled on all drives
- Defender for Endpoint installed and active
- Firewall enabled
- Device not jailbroken or rooted
- Password or PIN configured (minimum 8 characters)

**Recommended Requirements**:
- Secure Boot enabled
- TPM 2.0 present and functional
- No pending OS updates (grace period: 7 days)

**Non-Compliance Actions**:
- Grace period: 3 days to remediate
- After 3 days: Conditional Access blocks sign-in

### macOS Devices

**Mandatory Requirements**:
- macOS Monterey (12.0) or later
- FileVault enabled
- Defender for Endpoint installed
- Firewall enabled
- Device not jailbroken

**Limitations**: macOS compliance is weaker than Windows. Defender for Endpoint on macOS has fewer features.

**Trade-Off**: Some organisations allow macOS for developer productivity despite weaker controls.

### iOS/Android Devices

**Mandatory Requirements** (for corporate-owned):
- Latest OS version (grace period: 30 days after release)
- Device encryption enabled
- Device not jailbroken or rooted
- Intune Company Portal app installed

**Mandatory Requirements** (for BYOD):
- Intune App Protection Policies applied
- No jailbreak/root
- PIN or biometric authentication on device

**What App Protection Provides**:
- Copy/paste restrictions (cannot copy corporate data to personal apps)
- Save restrictions (cannot save corporate files to personal cloud storage)
- Screenshot restrictions (cannot screenshot sensitive data)

## Conditional Access Integration

Device trust is enforced via Conditional Access policies in Entra ID.

### Policy 1: Require Compliant Device for Azure Portal

**Conditions**:
- User is accessing Azure Portal
- Device is Windows or macOS

**Grant Controls**:
- Require device to be marked as compliant (Intune compliance check)
- Require MFA

**Result**: Non-compliant devices cannot access Azure Portal.

### Policy 2: Allow BYOD for Email/Teams Only

**Conditions**:
- User is accessing Exchange Online or Teams
- Device is iOS or Android

**Grant Controls**:
- Require approved client app (Outlook, Teams)
- Require app protection policy (Intune App Protection)

**Result**: Personal phones can access email but cannot access Azure Portal or other resources.

### Policy 3: Block Unmanaged Devices

**Conditions**:
- User is accessing any cloud app
- Device is not enrolled in Intune

**Grant Controls**:
- Block access

**Exceptions**:
- Break-glass accounts excluded
- External partners may have time-boxed exclusion

**Result**: Unmanaged devices cannot sign in.

## Device Lifecycle

### Provisioning

1. Device purchased by IT
2. Device enrolled in Intune during Windows Autopilot or Apple Business Manager setup
3. Security baselines applied automatically
4. Defender for Endpoint installed
5. User signs in with Entra ID credentials
6. Conditional Access evaluates compliance
7. User granted access

**Timeline**: New device provisioned and compliant within 1 hour.

### Operation

- Intune checks compliance every 8 hours
- Defender for Endpoint scans continuously
- Windows Update for Business enforces patching
- Non-compliant devices are flagged and blocked

### Decommissioning

1. User returns device or device is lost/stolen
2. IT admin retires device in Intune
3. Device is wiped remotely (corporate data removed)
4. Conditional Access blocks device even if wipe is not successful

**Timeline**: Device retired within 1 hour of request.

## Lost or Stolen Devices

### Corporate Devices

**Immediate Actions**:
1. IT admin marks device as lost/stolen in Intune
2. Intune issues remote wipe command
3. Conditional Access blocks device
4. User's credentials are reset (in case attacker extracted cached credentials)

**Timeline**: Device blocked within minutes. Wipe occurs when device next connects to internet.

**Risk**: If device is offline, wipe cannot complete. Conditional Access block prevents cloud resource access, but local data may be accessible.

**Mitigation**: BitLocker encryption protects local data. Attacker cannot decrypt without user credentials or recovery key.

### Personal Devices (BYOD)

**Immediate Actions**:
1. IT admin revokes user's Intune App Protection Policies
2. Corporate data in apps (Outlook, Teams) is wiped
3. User's credentials are reset

**Timeline**: App data wipe occurs within minutes.

**Risk**: Corporate data in personal storage (e.g., screenshots, saved files) cannot be remotely wiped.

**Mitigation**: Intune App Protection blocks screenshots and saving files to personal storage. Risk is minimised but not eliminated.

## Device Trust Failures

### 1. User Bypasses Compliance

**Scenario**: User disables BitLocker or uninstalls Defender for Endpoint to "improve performance".

**Detection**: Intune detects non-compliance within 8 hours.

**Response**: Conditional Access blocks user. User cannot sign in until compliance is restored.

**Remediation**: User re-enables BitLocker or reinstalls Defender. Intune re-evaluates compliance.

### 2. Defender for Endpoint Detects Malware

**Scenario**: User downloads malware. Defender for Endpoint detects it.

**Detection**: Immediate. Defender quarantines malware and sends alert to security team.

**Response**: Defender may automatically remediate (delete malware). If automatic remediation fails, device is flagged as "high risk" and Conditional Access blocks sign-in.

**Remediation**: Security team investigates. Device may be wiped and rebuilt.

### 3. Device Stolen but Offline

**Scenario**: Device is stolen. Thief keeps device offline to prevent remote wipe.

**Risk**: Local data is accessible if BitLocker is bypassed (requires physical hardware attack).

**Mitigation**: BitLocker with TPM 2.0 makes offline attacks difficult but not impossible.

**Outcome**: Conditional Access blocks cloud resource access. Local data risk accepted.

### 4. Personal Device Used for Privileged Access

**Scenario**: Admin attempts to activate PIM from personal phone.

**Detection**: Conditional Access blocks PIM activation from non-compliant device.

**Response**: User sees "PIM activation requires a compliant device. Use your corporate laptop."

**Outcome**: Privileged access restricted to corporate-managed devices only.

## Device Trust vs Zero Trust

"Zero trust" implies no device is trusted by default.

**Reality**: This is aspirational, not practical.

**Compromises We Accept**:
- Corporate-managed, compliant devices are **more trusted** than unmanaged devices
- Device compliance is a signal, not proof of security
- Phishing and credential theft can occur on compliant devices

**Defence-in-Depth**: Device trust is one layer. Identity (Conditional Access, PIM), network (private endpoints), and data (encryption) are additional layers.

## Device Trust Summary

| Device Type                  | Trust Level | Azure Portal | Email/Teams | PIM Activation |
|------------------------------|-------------|--------------|-------------|----------------|
| Corporate, Compliant         | High        | ✅           | ✅          | ✅             |
| Corporate, Non-Compliant     | Low         | ❌           | ❌          | ❌             |
| Personal (BYOD)              | Very Low    | ❌           | ✅          | ❌             |
| Unmanaged                    | None        | ❌           | ❌          | ❌             |

Device trust is enforced via Intune compliance and Conditional Access. No exceptions for convenience.