# ADR-008: AI Hub and AI Gateway as Shared Platform Baseline

**Status**: Accepted  
**Date**: 08/05/2026  
**Decision Scope**: Whether the enterprise AI platform baseline now includes an AI Hub with Azure API Management as AI Gateway for shared AI capability exposure  
**Depends on**: ADR-001 (Processing Paradigm), ADR-002 (Runtime Platform and EU Region Selection), ADR-003 (Network Isolation and Data Residency Enforcement), ADR-004 (Identity, Authentication, and Authorisation), ADR-006 (Observability and Operational Model)  
**Depended on by**: ADR-009 (Publication Model for Shared Models, Tools, and Reusable Agents), ADR-010 (MCP Exposure Through the AI Hub), ADR-011 (Agent-to-Agent Discovery and Invocation Through the AI Hub)  

---

## Context

The earlier platform baseline explicitly treated APIM as an optional future scaling step for a single-workload landing zone. That posture is no longer sufficient. The platform now needs a shared enterprise boundary for governed AI capability exposure, not only private per-workload backend access.

The infrastructure baseline has already moved in this direction. The platform diagram now includes an AI Gateway-oriented infrastructure view. The written architecture and ADR set need to promote that from an implied future direction to an explicit platform baseline.

---

## Decision Drivers

- **Enterprise reuse**: shared models, tools, and reusable agents need a common publication boundary.
- **Consumer isolation**: multiple consumers require stable quota, onboarding, and revocation boundaries.
- **Backend abstraction**: shared contracts should survive backend deployment, routing, and version changes.
- **Governance consistency**: policy enforcement, telemetry, and chargeback belong at a shared boundary, not only inside each workload.
- **Private platform posture**: the shared boundary must fit the platform's private-network and managed-identity model.

---

## Considered Alternatives

### Option A: Keep direct workload-to-backend access as the only baseline

Each workload consumes AI services directly and any shared capability is coordinated informally between teams.

**Why not chosen**: no real shared control plane, duplicated contracts, weak consumer isolation, and weak chargeback and audit consistency.

### Option B: Introduce an AI Hub and APIM AI Gateway as part of the baseline (SELECTED)

Adopt a shared mediation boundary for published AI capabilities while preserving direct private backend access for workload-internal logic.

**Why chosen**: creates a real shared platform capability without forcing every internal workload call through a gateway.

### Option C: Let each use case own its own gateway or publication surface

Each team chooses its own mediation layer for reusable capabilities.

**Why not chosen**: leads to duplicated governance, inconsistent security posture, and fragmented observability.

---

## Decision

Adopt an AI Hub with APIM as AI Gateway as part of the shared platform baseline.

APIM is the preferred mediation boundary for shared AI capabilities. The shared publication scope in phase 1 is models, tools, and Foundry-hosted reusable agents. MCP is a phase 1 capability. A2A is a defined platform capability and target pattern for shared reusable-agent interaction.

Direct in-spoke backend access remains allowed for non-shared internal workload logic. The AI Hub governs shared capability exposure; it does not replace every internal service call path.

---

## Consequences

### Positive

- The platform has a clear shared publication boundary for reusable AI capabilities.
- Shared contracts become decoupled from backend deployment details.
- Consumer onboarding, quota enforcement, and auditability become part of the baseline rather than an afterthought.
- The platform can define MCP and A2A consistently rather than as isolated experiments.

### Negative

- The platform now has another critical shared service to design and operate.
- Teams must distinguish clearly between shared publication and workload-private logic.
- APIM policy and onboarding design become architecture concerns, not just implementation details.

---

## Constraints

- The AI Hub is the preferred boundary for shared capabilities, not for every internal workload call.
- Current Container Apps use-case agents are not automatically promoted into the hub publication model.
- Shared publication must preserve the platform's private-network and managed-identity posture.

---

## Follow-ups

- Define the publication model for shared models, tools, and Foundry-hosted reusable agents.
- Define the MCP and A2A contract patterns in dedicated ADRs.
- Define the standard onboarding template for AI Hub consumers.
