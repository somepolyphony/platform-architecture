# Platform Mission

## Core Purpose

The platform exists to enforce organisational security posture whilst enabling delivery teams to ship workloads without manual platform gatekeeping.

This is not a support function. This is an enforcement function.

## Platform Goals

### 1. Security Enforcement at Scale

Security controls must be applied consistently across all Azure subscriptions and Microsoft 365 tenants without relying on human compliance.

**Why**: Manual security reviews do not scale. Delivery teams bypass controls when they are inconvenient. Enforcement must be automatic and non-negotiable.

**Trade-off**: Some workloads will be incompatible with enforced controls. These require explicit exceptions, not silent bypasses.

### 2. Blast Radius Containment

A compromise in one subscription, workload, or identity must not cascade to the entire tenant.

**Why**: Azure and Microsoft 365 share a single Entra ID tenant. A privileged identity compromise can pivot across boundaries unless explicitly prevented.

**Trade-off**: Strict isolation increases operational overhead. Break-glass accounts and emergency access become critical single points of failure.

### 3. Standardisation Over Freedom

Delivery teams receive pre-configured, opinionated subscriptions with networking, identity, and monitoring already applied.

**Why**: Every bespoke configuration is a security debt. Non-standard environments break automation, complicate audit, and increase attack surface.

**Trade-off**: Innovation and experimentation are constrained. Sandbox environments must be explicitly time-boxed and isolated.

### 4. Immutable Audit Trail

Every platform change must be logged, attributed, and non-repudiable.

**Why**: Compliance and incident response require proof of who changed what and when. Entra ID audit logs, Azure Activity logs, and Defender for Cloud alerts are non-negotiable.

**Trade-off**: High-frequency automation generates noise. Log retention costs scale with activity.

## Security Posture

### Defence-in-Depth is Not Optional

Perimeter controls (network segmentation, firewalls) are assumed to fail. Identity controls (Conditional Access, PIM) are assumed to fail. Endpoint controls (Defender, BitLocker) are assumed to fail.

**Platform design must tolerate any single control failure.**

This means:

- Network isolation AND identity-based access controls
- Just-in-time access AND audit logging
- Encryption at rest AND encryption in transit
- Preventative controls AND detective controls

### Security Defaults are Enforced, Not Recommended

Microsoft's security defaults and recommended baselines are starting points, not sufficient outcomes.

Where Microsoft provides optional hardening, we enforce it by default unless there is a documented business case for relaxation.

Examples:

- Multi-factor authentication is mandatory for all identities, including service principals where technically possible
- Privileged access is time-limited via PIM, not permanently assigned
- Defender for Cloud is enabled on all subscriptions with no opt-out
- Intune compliance is required for device access to corporate resources

### Shared Responsibility Model is Explicit

Microsoft is responsible for the physical infrastructure, hypervisor, and core platform services.

We are responsible for identity, data, endpoints, and workload configuration.

**There is no grey area.** If Microsoft does not explicitly commit to a control in their compliance documentation, we must implement it ourselves.

## Standardisation Philosophy

### Opinionated Defaults

Platform teams provide opinionated, pre-configured environments. Delivery teams do not choose networking models, identity providers, or logging destinations.

**Why**: Every decision deferred to delivery teams becomes a security inconsistency.

**Trade-off**: Some workloads genuinely require non-standard configurations. Exceptions must be explicit, time-boxed, and risk-accepted by leadership.

### Terraform Modules as Contracts

Infrastructure-as-code modules are contracts, not suggestions.

Delivery teams consume pre-approved modules. They do not write raw Terraform or deploy via the Azure Portal for production workloads.

**Why**: Every bespoke deployment is a unique security review burden. Modules centralise security logic and reduce variance.

**Trade-off**: Module limitations constrain delivery teams. Module defects impact multiple workloads simultaneously.

### Centralised Identity and Networking

All Azure subscriptions connect to a central hub network. All identities originate from a single Entra ID tenant.

**Why**: Decentralised identity and networking create ungovernable security boundaries.

**Trade-off**: Central hub becomes a single point of failure. Hub network team becomes a bottleneck for delivery teams.

## What Success Looks Like

- New subscriptions are provisioned in under 1 hour with full security controls pre-applied
- Delivery teams do not request firewall changes; private endpoints and service endpoints are used by default
- Security exceptions are rare, time-boxed, and require executive approval
- Audit passes without remediation findings related to platform controls
- Platform drift is detected and auto-remediated within 24 hours

## What Failure Looks Like

- Delivery teams bypass platform controls to meet deadlines
- Subscriptions exist outside management group hierarchy
- Admin accounts have permanent privileged access
- Workloads are deployed without Defender for Cloud coverage
- Azure Portal is used for production changes instead of Terraform

If any of these conditions are true, the platform has failed regardless of uptime or feature velocity.