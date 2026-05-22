# ADR-006: Observability and Operational Model

**Status:** Accepted  
**Date:** 08/05/2026  
**Decision Scope:** Telemetry architecture, tracing model, alerting strategy, audit logging, correlation design, and operational evidence across the workload platform and the AI Hub.  
**Depends on:** ADR-003 (Network Isolation and Data Residency Enforcement), ADR-004 (Identity, Authentication, and Authorisation), ADR-005 (Data Protection)  
**Depended on by:** None (terminal ADR in current chain)  

---

## Context

The original platform observability problem was already non-trivial: frontend, API, worker, Service Bus, SQL, Storage, AI Search, and Foundry all need end-to-end traceability in a private-by-default estate. The AI Hub makes observability a first-class architecture concern rather than a supporting concern. APIM now sits between shared AI consumers and backend model, tool, and reusable-agent services. That means the platform must be able to answer not only "what failed" but also "which consumer called which shared capability, under which Product, against which backend route, and with what quota or policy outcome".

The observability model must support operational troubleshooting, FinOps, quota governance, and audit evidence without turning telemetry into an uncontrolled business-data copy.

---

## Decision Drivers

- **Cross-boundary traceability**: Shared AI calls now cross consumer, gateway, runtime, and managed-service boundaries.
- **Consumer attribution**: The platform must attribute usage, throttling, and failures to the correct consumer boundary.
- **Audit defensibility**: Regulated environments need an explainable trail of policy decisions, denials, and execution paths.
- **Private telemetry posture**: Telemetry must follow the private-network baseline defined by ADR-003.
- **Sensitive-data minimisation**: Prompt and completion content must not be logged indiscriminately.
- **Operational simplicity**: One platform observability model is preferable to separate, incompatible telemetry islands.

---

## Considered Alternatives

### Option A: Runtime-only observability

Observe the application runtimes and managed Azure services, but treat APIM as an external black box.

**Why considered**: smallest incremental change from the pre-AI-Hub platform.

**Why not chosen**: breaks attribution and makes shared-capability governance unverifiable. The gateway becomes the least observed part of the shared platform.

### Option B: Separate observability stacks per consumer or per protocol

Give API, MCP, A2A, and workload runtimes separate workspaces, dashboards, and correlation models.

**Why considered**: strong isolation and potentially simpler ownership per team.

**Why not chosen**: poor end-to-end correlation, duplicated operational effort, and fragmented audit evidence.

### Option C: Unified platform observability with APIM as a first-class telemetry source (SELECTED)

Use a unified correlation model spanning APIM, workload runtimes, and managed services, with shared ingestion into the platform observability stack and explicit guardrails for sensitive data logging.

**Why chosen**: gives the platform one coherent operational and audit story while preserving consumer attribution and private-only ingestion.

---

## Decision

Adopt a unified observability model across the workload platform and the AI Hub.

Application Insights and Log Analytics remain the core observability sinks, with AMPLS enforcing private-only ingestion for workload telemetry. APIM becomes a first-class telemetry source in the same model. Gateway diagnostics, policy outcomes, per-consumer usage data, token-related metrics, throttling events, and protocol-specific metadata for MCP and A2A are part of the shared platform observability contract.

Telemetry is correlated across user entry point, APIM, workload runtime, queue-driven processing, and managed Azure services through platform correlation identifiers. Prompt or completion content is not logged by default. Any request/response body logging must be selective, justified, and reviewed against the platform data-protection posture.

---

## Decision Details

### 1. Correlation starts at the first governed boundary

Each governed interaction receives a correlation identifier at the earliest platform-controlled boundary:

- frontend/API entry for direct application access
- Copilot Studio / Power Platform integration boundary for conversational access
- APIM gateway boundary for shared AI capability consumption

That identifier is propagated through workload services, backend AI calls, queue messages, and managed-service interactions where supported.

### 2. APIM telemetry is part of the platform truth

For shared AI capabilities, the platform records at minimum:

- consumer Product and subscription context
- API / MCP / A2A contract invoked
- gateway policy outcomes
- quota or throttling events
- backend route selected
- latency and failure signals

This is required for chargeback, incident analysis, and compliance evidence.

### 3. Sensitive content logging is opt-in and tightly controlled

The default telemetry posture is metadata-first, not content-first. Prompt and completion content are not logged globally. Request or response body capture must be justified per capability and reviewed for data-protection impact. This is especially important for MCP streaming paths and A2A payloads, where indiscriminate body logging can either leak sensitive content or interfere with protocol behaviour.

### 4. Protocol-specific observability is standardised

The platform observability model includes protocol-specific metadata where relevant:

- API: consumer, route, backend, policy, latency, quota
- MCP: tool endpoint, consumer, protocol transport, policy, error path
- A2A: published agent identity, caller identity context where contractually required, target agent contract, policy and routing outcome

The protocol changes; the correlation model does not.

### 5. Operational dashboards and alerts are platform responsibilities

The platform maintains dashboards and alerts for:

- APIM availability and error-rate changes
- throttling and quota breach events
- backend dependency failures
- queue backlog and dead-letter growth
- SQL and storage access failures affecting audit or workflow continuity
- AI Search and Foundry dependency health

Shared capability operations are not considered observable if APIM is omitted from the dashboard and alert baseline.

---

## Consequences

### Positive

- One coherent operational model across direct applications, conversational access, and shared AI mediation.
- Better attribution of failures, quota breaches, and usage spikes to the correct consumer boundary.
- Stronger audit trail for policy enforcement and shared capability use.
- Shared dashboards and alerting reduce operational blind spots between platform and AI Hub teams.

### Negative

- More telemetry sources and more correlation work than a runtime-only model.
- Teams must understand when metadata is sufficient and when content logging is prohibited.
- APIM diagnostic configuration becomes production-critical rather than optional.

---

## Constraints

- Telemetry ingestion remains private-only via AMPLS where the platform controls the path.
- Prompt and completion content are not logged globally.
- APIM must be configured as a monitored production dependency, not just as a gateway component.
- Observability design must preserve the distinction between operational telemetry and audit evidence, even when both live in the same broader monitoring estate.

---

## Risks and Mitigations

|Risk|Likelihood|Impact|Mitigation|
|---|---|---|---|
|Prompt or completion bodies are logged too broadly.|Medium|High|Default to metadata-only logging and require explicit per-capability approval for body capture.|
|Teams monitor backend health but ignore APIM policy or quota failures.|Medium|High|Make APIM telemetry and alerts part of the mandatory platform dashboard baseline.|
|Correlation IDs stop at protocol boundaries, leaving shared calls hard to trace.|Medium|Medium|Define correlation propagation rules for API, MCP, and A2A contracts and test them in validation paths.|
|Protocol-specific logging breaks MCP or overloads A2A traces with low-value data.|Medium|Medium|Apply selective protocol-aware logging and review changes as architecture concerns rather than ad hoc debug settings.|

---

## Follow-ups

- Define the standard dashboard set for platform operators covering APIM, workload runtimes, queues, SQL, AI Search, and Foundry.
- Define the minimum telemetry fields required for per-consumer chargeback and audit reporting.
- Validate the exact APIM diagnostic categories and Application Insights / Log Analytics mappings used for MCP and A2A in the target deployment.
