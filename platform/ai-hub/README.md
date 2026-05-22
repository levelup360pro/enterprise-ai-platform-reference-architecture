# AI Hub

**Status:** Draft  
**Date:** 08/05/2026  
**Repository:** `platform/ai-hub/README.md`  

---

## Context

The platform baseline now includes an AI Hub with Azure API Management as AI Gateway. The AI Hub exists to provide a single publication and consumption boundary for reusable AI capabilities that need stable contracts, policy enforcement, quota control, and auditable onboarding.

This folder holds the supporting architecture documents for that shared capability layer.

---

## Document Set

| Document | Purpose |
| -------- | ------- |
| [publication-model.md](publication-model.md) | Publication classes, ownership boundaries, onboarding, and governance model |
| [protocol-patterns.md](protocol-patterns.md) | API, MCP, and A2A exposure patterns, consumer classes, and preferred integration paths |

---

## Scope

The AI Hub document set covers:

- the publication model for shared models, tools, and Foundry-hosted reusable agents
- the distinction between shared capability mediation and direct workload-internal backend access
- the target protocol patterns for API, MCP, and A2A exposure
- the consumer taxonomy and the preferred gateway path per consumer type

It does not replace the platform reference architecture or the platform ADRs. It explains how to apply them to the AI Hub boundary.
