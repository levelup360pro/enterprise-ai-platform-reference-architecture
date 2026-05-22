# ADR-011: Agent-to-Agent Discovery and Invocation Through the AI Hub

**Status**: Accepted  
**Date**: 08/05/2026  
**Decision Scope**: The target platform pattern for shared reusable agent discovery and invocation through A2A contracts in the AI Hub  
**Depends on**: ADR-004 (Identity, Authentication, and Authorisation), ADR-008 (AI Hub and AI Gateway as Shared Platform Baseline), ADR-009 (Publication Model for Shared Models, Tools, and Reusable Agents)  
**Depended on by**: None  

---

## Context

The platform has decided that A2A is a platform capability and that the architecture should define a target pattern now. The question is not whether UC1 or UC2 uses it immediately. The question is how a future shared reusable agent should be discovered and invoked when one agent needs to call another at runtime.

---

## Decision Drivers

- **Runtime agent composition**: reusable agents need a governed delegation pattern.
- **Discoverability**: callers need a stable discovery contract rather than hard-coded backend coordinates.
- **Policy and auditability**: shared agent invocation needs the same governance boundary as any other shared capability.
- **Future readiness**: the platform should define the pattern now even if early use cases do not consume it yet.

---

## Considered Alternatives

### Option A: No platform-level A2A pattern yet

**Why not chosen**: delays an inevitable shared-agent design concern and encourages ad hoc point-to-point agent calls.

### Option B: Direct agent-to-agent backend calls

**Why not chosen**: weak discovery, inconsistent policy enforcement, and no stable shared contract boundary.

### Option C: A2A discovery and invocation through APIM (SELECTED)

**Why chosen**: keeps reusable agents discoverable and governed through the same shared platform boundary as other published capabilities.

---

## Decision

Adopt A2A as the target platform pattern for shared reusable-agent discovery and invocation. Shared reusable agents that are intended for runtime delegation should publish an A2A contract through APIM. APIM is the discovery and policy boundary. The backend reusable agent remains private behind that boundary.

A2A is a platform capability, not an immediate requirement for every use case. The pattern exists so future shared reusable agents can be published consistently.

---

## Consequences

### Positive

- Shared reusable agents gain a governed discovery and invocation pattern.
- Agent-to-agent interaction is brought under the same policy, identity, and observability model as other shared capabilities.
- Future agent composition can evolve without point-to-point contract sprawl.

### Negative

- A2A introduces another shared contract type that must be documented and operated.
- Teams must distinguish clearly between reusable published agents and workload-internal orchestration agents.

---

## Constraints

- A2A applies to shared reusable agents, not to every internal agent interaction.
- APIM remains the discovery and governance surface.
- Backend reusable agents stay private and are not exposed as unmanaged public endpoints.

---

## Follow-ups

- Define the minimum metadata expected for a published reusable-agent contract.
- Validate the exact caller-identity propagation model for A2A where the target agent needs caller context for downstream policy decisions.
