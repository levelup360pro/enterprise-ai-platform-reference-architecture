# AI Hub: Publication Model

**Status:** Draft  
**Date:** 08/05/2026  
**Repository:** `platform/ai-hub/publication-model.md`  

---

## Context

The AI Hub is the shared platform boundary for reusable AI capabilities. Not every backend capability belongs there. The publication model exists to separate capabilities that must be governed and reused through a stable contract from capabilities that remain internal to a single workload.

---

## Scope

### In scope

- shared publication classes
- ownership boundaries between platform and use-case teams
- onboarding model for consumers
- stable contract expectations for shared AI capabilities

### Out of scope

- detailed APIM policy XML
- low-level deployment automation
- use-case-specific internal orchestration contracts

---

## Publication Classes

### 1. Models

Models are shared inference contracts backed by Foundry model deployments. They are exposed through APIM when the model capability must be consumed by more than one workload, team, or channel. The consumer sees a stable contract and quota boundary; the platform team retains control of the backend deployment, routing, failover, and throttling strategy.

### 2. Tools

Tools are approved callable capabilities that can be exposed either as conventional APIs or as MCP endpoints. The publication choice depends on the consumer contract. Conventional APIs remain the default for deterministic application integration. MCP is used when tool discovery and tool-calling interoperability are architectural requirements.

### 3. Foundry-hosted reusable agents

Reusable agents are shared agent capabilities hosted through Microsoft Foundry and published for cross-team or cross-use-case use. They are distinct from workload-internal orchestration agents. A reusable agent must have a stable contract, explicit ownership, bounded purpose, and a defined access model before publication.

---

## Publication Boundary Rule

A capability belongs in the AI Hub when one or more of the following are true:

- it is intended for reuse across multiple workloads or teams
- it needs a stable consumer-facing contract independent of backend deployment details
- it requires platform-managed quotas, throttling, or chargeback
- it needs a standard onboarding and approval path
- it must be discoverable as a shared enterprise AI capability

If none of these are true, the capability remains internal to the workload and can use direct private backend access.

---

## Ownership Model

| Concern | Platform team | Use-case team |
| ------- | ------------- | ------------- |
| AI Hub network boundary and APIM instance | Owns | Consumes |
| Product and subscription governance | Owns | Requests / uses |
| Shared model publication | Owns | May consume |
| Shared tool publication | Owns the publication surface | Proposes and maintains tool implementation where delegated |
| Shared reusable agent publication | Owns the publication boundary | Owns the agent capability when delegated |
| Workload-internal agents and pipelines | Does not own | Owns |

---

## Onboarding Model

The APIM Product is the governance unit for consumer onboarding.

Each approved consumer is onboarded through:

1. a defined Product
2. an approved subscription
3. the required authentication posture for that consumer type
4. documented quota, policy, and observability expectations

This keeps onboarding explicit. Consumers do not receive direct ad hoc backend access to shared capabilities.

---

## Phase 1 Boundary

Phase 1 shared publication scope is:

- models
- tools
- Foundry-hosted reusable agents

Current Container Apps use-case agents remain where they are today. They are not automatically published through the AI Hub simply because they exist.
