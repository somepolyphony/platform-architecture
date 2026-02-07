# Windows Update for Business Guardrails

## Update Enforcement Philosophy

Patching is non-negotiable. Unpatched systems are compromised systems.

Windows Update for Business (WUfB) enforces patching without requiring manual intervention.

## Why Patching is Mandatory

### 1. Known Vulnerabilities Are Exploited Immediately

When Microsoft releases a security patch, attackers reverse-engineer the patch to identify the vulnerability.

**Timeline**:
- Patch released: Day 0
- Vulnerability identified by attackers: Day 1-3
- Exploit code published: Day 7-14
- Mass exploitation begins: Day 14-30

**Implication**: Devices must be patched within days, not weeks or months.

### 2. Zero-Day Exploits Are Rare; N-Day Exploits Are Common

Most successful attacks exploit **known** vulnerabilities, not zero-days.

**Why**: Organisations delay patching due to fear of breakage or inconvenience. Attackers exploit this window.

**Example**: WannaCry ransomware (2017) exploited EternalBlue vulnerability. Patch was available **two months** before the attack.

### 3. Manual Patching Does Not Scale

Relying on users to install updates manually is ineffective.

**Reality**:
- Users defer updates indefinitely
- Users disable automatic updates to "improve performance"
- Users ignore update notifications

**Solution**: Automatic updates enforced via WUfB. No user opt-out.

## Update Types

### Feature Updates

**What**: Major Windows version updates (e.g., Windows 10 21H2 → 22H2, Windows 11 23H2 → 24H2).

**Frequency**: Annually or bi-annually.

**Risk**: High. Feature updates can break applications or introduce instability.

**Enforcement**:
- Feature updates deferred **60 days** after release
- Pilot group (5% of devices) receives updates first
- Production group (95%) receives updates after 60 days if no issues detected

**Why Deferred**: Early adopters absorb breakage risk. Waiting allows Microsoft to release fixes for initial bugs.

### Quality Updates (Security Patches)

**What**: Monthly security patches and bug fixes (released on "Patch Tuesday").

**Frequency**: Monthly (second Tuesday of each month).

**Risk**: Low to moderate. Patches occasionally break specific applications but are generally safe.

**Enforcement**:
- Quality updates deferred **7 days** after release
- Pilot group receives updates immediately
- Production group receives updates after 7 days

**Why Deferred**: Allows detection of critical bugs before mass deployment.

### Driver Updates

**What**: Hardware driver updates (graphics, network, storage).

**Frequency**: Ad-hoc (varies by vendor).

**Risk**: Moderate. Driver updates can cause hardware failures or compatibility issues.

**Enforcement**:
- Driver updates deferred **30 days** after release
- Manual approval required for critical driver updates (e.g., network adapters)

**Why Deferred**: Driver updates are less critical than security patches and pose higher breakage risk.

## Update Rings

Devices are assigned to **update rings** based on role and risk tolerance.

### Pilot Ring (5% of Devices)

**Who**: IT staff, platform team, early adopters.

**Update Schedule**:
- Feature updates: Immediate (no deferral)
- Quality updates: Immediate (no deferral)
- Driver updates: 14-day deferral

**Why**: Early detection of issues before production rollout.

**Trade-Off**: Pilot users experience breakage first. Acceptable because they are technical and can provide feedback.

### Production Ring (95% of Devices)

**Who**: All other corporate users.

**Update Schedule**:
- Feature updates: 60-day deferral
- Quality updates: 7-day deferral
- Driver updates: 30-day deferral

**Why**: Stability prioritised over bleeding-edge features.

### Critical Systems Ring (Exceptions Only)

**Who**: Kiosks, production systems, medical devices.

**Update Schedule**:
- Feature updates: Deferred indefinitely (manual approval required)
- Quality updates: 30-day deferral
- Driver updates: Deferred indefinitely

**Why**: These systems require stability above all else. Breakage is unacceptable.

**Trade-Off**: Security patches are delayed. Risk is accepted for critical systems that are network-isolated or have compensating controls.

**Approval**: Requires executive approval. Reviewed quarterly.

## Automatic Installation and Restart

### Installation Behaviour

Updates are **automatically downloaded and installed** without user interaction.

**When**: During active hours (user-configured) or idle time.

**User Experience**: User sees notification: "Update is installing. Do not turn off your device."

**No User Opt-Out**: Users cannot disable automatic installation.

### Restart Behaviour

Updates that require restart are **automatically applied** during maintenance windows.

**Maintenance Windows**:
- **Default**: 03:00 - 05:00 local time (outside business hours)
- **Configurable**: Organisations can adjust based on shift work or global teams

**User Deferral**:
- Users can defer restart for up to **3 days**
- After 3 days, restart is **mandatory** and cannot be deferred

**Why Mandatory Restart**: Security patches do not take effect until restart. Devices without restart are vulnerable.

**User Experience**:
- 2 hours before restart: Notification appears with countdown timer
- 30 minutes before restart: Persistent notification with "Restart Now" button
- 0 minutes: Device restarts automatically. Unsaved work may be lost.

**Mitigation for Data Loss**: Applications with auto-save (Microsoft 365 apps) recover unsaved work. Users are trained to save work frequently.

## Failure Modes

### 1. Update Breaks Application

**Scenario**: Quality update breaks line-of-business application.

**Example**: Update changes Windows API behaviour. Application crashes on startup.

**Detection**:
- Pilot group reports issue within 24 hours
- IT validates issue is update-related

**Remediation**:
- Pause production rollout immediately
- Microsoft releases fix within 7-14 days (if widespread issue)
- If no fix available, application vendor must update app

**Timeline**: Production rollout resumed after fix validated in pilot.

**Known Example (2026)**: Windows 11 23H2 update broke legacy printer drivers. Microsoft released emergency patch within 5 days.

### 2. Update Installation Fails

**Scenario**: Update downloads but fails to install.

**Causes**:
- Insufficient disk space
- Corrupted update files
- Conflicting third-party software (antivirus, VPN)

**Detection**: Windows Update error code logged. Intune flags device as non-compliant.

**Remediation**:
- Intune runs troubleshooter remotely (frees disk space, resets update components)
- If automated remediation fails, IT manually intervenes
- Worst case: Device is reimaged

**Frequency**: ~1-2% of devices per update cycle.

### 3. User Defers Restart Indefinitely

**Scenario**: User defers restart for 3 days, then continues working without restarting.

**Reality**: After 3 days, restart is **forced**. User cannot defer further.

**User Experience**: "Your device will restart in 15 minutes. Save your work."

**Complaints**: Users complain about losing unsaved work.

**Response**: Training and communication. Unsaved work is user responsibility.

### 4. Update Causes Boot Failure

**Scenario**: Update corrupts bootloader. Device cannot boot.

**Causes**:
- Faulty hardware (disk corruption)
- Interrupted update (user forcibly powered off device during update)

**Detection**: Device does not reconnect to Intune. User reports issue.

**Remediation**:
- Windows Recovery Environment (WinRE) automatically rolls back update
- If rollback fails, device must be reimaged

**Frequency**: Rare (~0.1% of devices).

**Mitigation**: BitLocker recovery key is stored in Entra ID. IT can unlock device remotely.

## Patching for Non-Windows Devices

### macOS Devices

**Tool**: Intune enforces macOS updates but with less control than Windows.

**Limitations**:
- Cannot force immediate restart
- Users can defer updates longer than 3 days
- No pilot ring concept

**Trade-Off**: macOS devices have weaker patching enforcement. Accepted because macOS is less targeted by malware.

### Linux Devices (Azure VMs)

**Tool**: Azure Update Management or custom automation (Ansible, Chef).

**Enforcement**:
- Security patches auto-installed during maintenance windows
- Reboots scheduled during low-traffic periods

**Trade-Off**: Linux patching is more complex. Requires workload-specific testing.

### Mobile Devices (iOS/Android)

**Tool**: Intune enforces minimum OS version via compliance policy.

**Behaviour**:
- Device must be on latest OS version (or within 30 days of latest release)
- If non-compliant, Conditional Access blocks access

**Trade-Off**: Cannot force OS updates. User must manually update. Compliance policy blocks access if outdated.

## Patching and Compliance

Devices with pending updates are **not** marked non-compliant unless the deferral period has expired.

**Example**:
- Quality update released on Patch Tuesday
- 7-day deferral period for production devices
- On day 8, if update is not installed, device is marked non-compliant
- Conditional Access blocks access until update is installed

**Why**: Allows time for automatic installation. Does not penalise users for updates in progress.

## Communication and Change Management

Users are notified of:
- Upcoming feature updates (60-day notice)
- Mandatory restart schedules (2-hour notice)
- Update-related outages or issues (real-time via email/Teams)

**Messaging Tone**:
- "Security updates are mandatory and protect your device."
- "Your device will restart tonight at 03:00 to complete installation."
- "If you experience issues after an update, contact IT immediately."

**No Apologies**: Patching is non-negotiable. Communication is informative, not apologetic.

## Update Enforcement Checklist

- [ ] Windows Update for Business configured for all corporate devices
- [ ] Pilot ring (5%) receives updates first
- [ ] Production ring (95%) receives updates after deferral period
- [ ] Automatic restart enforced after 3-day deferral
- [ ] Devices with pending updates (beyond deferral) marked non-compliant
- [ ] Conditional Access blocks non-compliant devices
- [ ] Feature updates deferred 60 days
- [ ] Quality updates deferred 7 days
- [ ] Driver updates deferred 30 days
- [ ] Critical systems ring requires executive approval

## Patching Philosophy Summary

**Patching is not optional. It is mandatory.**

- Feature updates deferred for stability
- Quality updates deferred minimally for risk management
- Automatic installation and restart enforced
- Users cannot opt out
- Pilot group absorbs breakage risk
- Non-compliant devices are blocked by Conditional Access

**Trade-Off**: User convenience vs security. Security wins.