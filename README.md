# Platform Architecture

Architectural guardrails and decision authority for Azure and Microsoft 365 platform design, security, and standardisation.

## Purpose

This repository defines **why** platform controls exist, not how they are implemented.

It serves as the design authority layer above enforcement tooling (Terraform, Microsoft Defender, Intune). Every decision documented here must be defensible to senior engineers, security architects, and external audit.

This is not an implementation repository. It contains no Terraform code, PowerShell scripts, or step-by-step tutorials.

## Intended Audience

- Cloud Platform Architects
- Principal and Senior Platform Engineers
- Cloud Security Architects
- Internal platform ownership teams
- MSP platform leads

If you are implementing platform controls, you should read this first to understand the **rationale** before touching automation.

## What This Repository Contains

- **Platform charter**: Mission, threat model, and explicit non-goals
- **Identity and access boundaries**: Admin models, break-glass, and exception handling
- **Subscription and management group design**: Blast radius containment and organisational hierarchy
- **Networking guardrails**: Hub-spoke rationale, private endpoints, and trust boundaries
- **Security and Defender scope**: Policy ownership and enforcement vs monitoring trade-offs
- **Endpoint and Intune baselines**: Device trust models and update enforcement philosophy
- **Terraform and automation boundaries**: Why Terraform, module design, and drift handling
- **Exception and risk acceptance processes**: How exceptions are granted and time-boxed
- **Operating model**: Ownership, RBAC separation, and MSP vs internal differences

## What This Repository Does NOT Contain

- Terraform modules or configuration
- PowerShell scripts or automation code
- Step-by-step implementation guides
- Click-through Azure Portal tutorials
- Marketing content or vendor guidance
- Emojis, motivational language, or apologetic phrasing

## Relationship to Enforcement Repositories

This repository **informs** but does not **implement**.

Platform enforcement is handled in separate repositories:

- **Terraform Landing Zone**: Implements subscription scaffolding, networking, and Azure Policy
- **Defender Hardening**: Deploys Microsoft Defender for Cloud and Office configurations
- **Intune Baselines**: Applies device compliance and security baselines via Microsoft Intune

Those repositories must align with the decisions documented here. If they diverge, this repository is the source of truth for **intent**, and the enforcement repo must be corrected or an exception documented.

## Authorial Stance

This repository is opinionated, security-first, and pragmatic.

- Guardrails over freedom
- Defaults over exceptions
- Enforcement over guidance
- Explicit acknowledgement of platform constraints and failure modes

If you disagree with a decision, propose a change via pull request with a clear rationale and trade-off analysis.

## Structure

```
00-platform-charter/         Platform mission, threat model, and non-goals
01-identity-and-access/      Identity boundaries, admin model, break-glass
02-subscriptions-and-management-groups/  Hierarchy design and blast radius
03-networking-guardrails/    Hub-spoke, private endpoints, trust boundaries
04-security-and-defender/    Defender scope, policy authority, enforcement
05-endpoint-and-intune/      Device trust, baselines, Windows Update for Business
06-terraform-and-automation/ Terraform rationale, modules, drift handling
07-exceptions-and-risk-acceptance/  Exception processes and time-boxing
08-operating-model/          Ownership, RBAC, MSP vs internal
references/                  Links to enforcement repositories
```

## Contributing

Changes to this repository must include:

1. **Why** a decision exists
2. **What** trade-offs were accepted
3. **What** breaks if this control is removed
4. Explicit callout of Azure/M365 hard limits or known failure scenarios

Pull requests without this context will be rejected.

## Licence

This repository represents internal platform design authority. Redistribution is permitted but attribution is required.
