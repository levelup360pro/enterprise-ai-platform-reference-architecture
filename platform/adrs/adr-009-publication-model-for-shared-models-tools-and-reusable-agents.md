# ADR-009: Publication Model for Shared Models, Tools, and Reusable Agents

**Status**: Accepted  
**Date**: 08/05/2026  
**Decision Scope**: How the platform publishes shared AI capabilities through the AI Hub, including publication classes, ownership boundaries, and onboarding units  
**Depends on**: ADR-004 (Identity, Authentication, and Authorisation), ADR-006 (Observability and Operational Model), ADR-008 (AI Hub and AI Gateway as Shared Platform Baseline)  
**Depended on by**: ADR-010 (MCP Exposure Through the AI Hub), ADR-011 (Agent-to-Agent Discovery and Invocation Through the AI Hub)  

---

## Context

Once the AI Hub becomes part of the platform baseline, the next question is not simply whether something is exposed through APIM. The platform needs a publication model. Different capabilities have different governance needs. A shared model endpoint, a shared tool, and a reusable agent are not the same kind of platform asset even if APIM mediates all three.

---

## Decision Drivers

- **Clear capability taxonomy**: shared capabilities need explicit classes so governance is not ambiguous.
- **Ownership clarity**: platform and use-case teams need a stable ownership split.
- **Stable onboarding**: consumers need one repeatable contract and approval model.
- **Operational accountability**: quota, policy, and telemetry need a defined governance unit.

---

## Considered Alternatives

### Option A: Treat everything as a generic API product

Expose all shared AI capabilities as undifferentiated APIs.

**Why not chosen**: simple on paper, but it hides important differences in contract, lifecycle, and operational ownership.

### Option B: Separate publication classes with a common gateway boundary (SELECTED)

Define explicit publication classes for models, tools, and Foundry-hosted reusable agents while using APIM Products as the onboarding and governance unit.

**Why chosen**: gives the platform one shared publication framework without pretending every capability is the same.

---

## Decision

Adopt three publication classes for the AI Hub:

- **Models**
- **Tools**
- **Foundry-hosted reusable agents**

The APIM Product is the governance and onboarding unit for shared capability consumption. Each published capability belongs to one or more Products with explicit consumer scope, quota, policy, and observability expectations.

Current Container Apps use-case agents remain outside this publication model unless they are explicitly redesigned and approved as shared reusable capabilities.

---

## Consequences

### Positive

- Shared capability types are explicit rather than implied.
- Consumer onboarding becomes consistent across different shared AI assets.
- The platform can apply protocol-specific choices without losing a common governance model.
- Ownership boundaries are easier to document and operate.

### Negative

- Publication review becomes a required step before a capability is shared.
- Some teams may need to separate workload-private implementations from shared reusable surfaces.

---

## Constraints

- APIM Products and subscriptions are the standard onboarding boundary for shared capabilities.
- A published capability must have an explicit owner.
- Shared publication does not eliminate workload-private implementations where those remain appropriate.

---

## Follow-ups

- Define the MCP-specific publication rules for shared tools.
- Define the A2A-specific publication rules for reusable agents.
- Define the approval checklist for moving a capability from workload-private to shared publication.
