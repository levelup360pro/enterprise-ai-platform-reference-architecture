# AI Hub: Protocol Patterns

**Status:** Draft  
**Date:** 08/05/2026  
**Repository:** `platform/ai-hub/protocol-patterns.md`  

---

## Context

The AI Hub supports three exposure patterns in the platform target state: conventional API, MCP, and A2A. They do not solve the same problem. This document defines when each pattern is preferred and how they fit the shared platform boundary.

---

## Consumer Classes

| Consumer class | Typical example | Preferred path |
| -------------- | --------------- | -------------- |
| Direct application consumer | Frontend/API consuming shared model or shared tool | APIM API |
| Conversational channel consumer | Copilot Studio / Power Platform calling shared capability | APIM API or APIM-published MCP endpoint |
| Agent consumer needing tool discovery | Foundry-hosted reusable agent or governed tool runner | APIM-published MCP endpoint |
| Agent consumer needing runtime agent discovery or delegation | Shared reusable agent invoking another shared reusable agent | APIM-published A2A endpoint |
| Workload-internal private logic | Internal use-case orchestration | Direct private backend access |

---

## Pattern 1: API

Use a conventional API contract when:

- the consumer already expects request/response API integration
- the capability is deterministic or tightly scoped
- tool discovery is not required
- the integration should remain simple and explicit

This is the default exposure mode for shared models and many shared tools.

---

## Pattern 2: MCP

Use MCP when:

- the consumer is an agent or agent-capable runtime that benefits from tool discovery
- the tool should appear as a governed tool contract rather than as a bespoke API integration
- the same tool may be consumed by multiple agent runtimes with minimal per-consumer adaptation

MCP is a phase 1 capability for this platform. APIM is the governance surface for published MCP endpoints. A shared tool may be implemented as a native MCP server or projected from an approved API through APIM where that gives a cleaner enterprise contract.

---

## Pattern 3: A2A

Use A2A when:

- one shared agent must discover and invoke another shared agent at runtime
- the interaction is agent-to-agent rather than user-to-agent or app-to-tool
- the platform needs an explicit delegation contract for reusable agent capabilities

A2A is a platform capability and target pattern. It is not required for initial UC1 or UC2 delivery, but the architecture defines it now so future reusable agents can be published consistently.

---

## Authentication Boundary

For all three patterns, the preferred shared path is:

1. consumer authenticates to APIM
2. APIM enforces product, policy, and quota controls
3. APIM authenticates to the backend with a managed identity or an approved credential mediation path
4. backend execution remains private

The consumer-facing contract and the backend credential are deliberately separate concerns.

---

## Direct Access Exception

Direct workload-to-backend access remains valid for non-shared internal workload logic. The AI Hub is not intended to force every internal backend call through APIM. It exists to govern shared capability exposure.
