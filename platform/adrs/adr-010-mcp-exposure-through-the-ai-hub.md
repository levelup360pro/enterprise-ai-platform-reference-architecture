# ADR-010: MCP Exposure Through the AI Hub

**Status**: Accepted  
**Date**: 08/05/2026  
**Decision Scope**: When and how MCP is used in phase 1 to expose shared tools through the AI Hub  
**Depends on**: ADR-004 (Identity, Authentication, and Authorisation), ADR-008 (AI Hub and AI Gateway as Shared Platform Baseline), ADR-009 (Publication Model for Shared Models, Tools, and Reusable Agents)  
**Depended on by**: None  

---

## Context

The platform has already decided that MCP is a phase 1 capability. That does not mean every shared tool should become an MCP endpoint. The architecture needs a clear rule for when a shared tool is exposed as a conventional API and when it is exposed as MCP.

---

## Decision Drivers

- **Tool interoperability**: some consumers need a discoverable tool contract rather than a bespoke API integration.
- **Governed exposure**: MCP endpoints still need the same policy, quota, and audit controls as any other shared capability.
- **Pragmatism**: not every deterministic integration benefits from MCP.
- **Phase 1 readiness**: the platform needs a usable target pattern now, not only a placeholder capability statement.

---

## Considered Alternatives

### Option A: Expose every shared tool only as a conventional API

**Why not chosen**: simple, but it ignores the runtime tool-discovery use cases that MCP is meant to serve.

### Option B: Expose every shared tool as MCP

**Why not chosen**: over-rotates toward agent interoperability even where a normal API is the clearer and simpler contract.

### Option C: Use API by default and MCP where tool discovery is the architectural requirement (SELECTED)

**Why chosen**: keeps phase 1 practical while still making MCP a real platform capability.

---

## Decision

Use conventional API contracts as the default shared exposure mode for tools. Use MCP when the consumer is an agent or agent-capable runtime that benefits from a discoverable tool contract.

APIM is the governance surface for published MCP endpoints. The platform may either publish a native MCP backend through APIM or project an approved API into an MCP surface through APIM when that creates the right shared contract.

---

## Consequences

### Positive

- MCP becomes a real phase 1 platform capability without forcing it onto every integration.
- Tool publication stays aligned to consumer need instead of technology preference.
- The AI Hub keeps one governance surface for both API and MCP forms.

### Negative

- Some tools may exist in both API and MCP form, which requires clear ownership and versioning discipline.
- Teams must decide deliberately whether a tool is integration-first or discovery-first.

---

## Constraints

- Published MCP endpoints must still follow the platform identity, network, and observability baseline.
- MCP is for shared tool exposure, not a blanket replacement for conventional APIs.
- Phase 1 MCP publication is governed through APIM, not through unmanaged direct exposure.

---

## Follow-ups

- Define the publication checklist for a shared tool that needs both API and MCP forms.
- Validate the logging and streaming configuration for MCP paths so observability controls do not break protocol behaviour.
