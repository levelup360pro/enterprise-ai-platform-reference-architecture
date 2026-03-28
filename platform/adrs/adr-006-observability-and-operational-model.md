# ADR-006: Observability and Auditability (Placeholder)

**Status:** Planned  
**Date:**  
**Decision Scope:** Telemetry architecture, tracing model, alerting strategy, audit logging, correlation design, and operational model for the AI platform.  
**Depends on:** ADR-003 (Network Isolation and Data Residency Enforcement), ADR-004 (Identity, Authentication, and Authorisation), ADR-005 (Data Protection)  
**Depended on by:** None (terminal ADR in current chain)  

---

## Scope

ADR-006 will define the observability and operational model for the platform. It addresses what telemetry is captured, how it flows, where it is stored, how it is secured, and what operational decisions it enables.

**Key dependencies from prior ADRs that shape this decision:**

ADR-003 established Azure Monitor Private Link Scope (AMPLS) and Private Only ingestion mode. All telemetry must flow through the AMPLS private endpoint; no public ingestion is permitted. This constrains the Log Analytics workspace architecture and determines how Application Insights, Diagnostic Settings, and any future monitoring agents connect.

ADR-004 established that `uami-platform` is the shared runtime identity for the FastAPI API app, worker app, and any platform jobs that use the orchestration identity. All telemetry emitted by the orchestration layer is attributable to this identity. Per-agent identities (`uami-agent-{name}`) must produce distinguishable traces so that agent-level behaviour can be audited independently.

ADR-005 established that Log Analytics uses Microsoft-managed keys (MMK) and that this decision depends on a binding code-level governance constraint: the frontend, API, worker, and job code must not log PII, business data, or extraction content in telemetry payloads. ADR-006 must define the enforcement mechanism for this constraint. If the constraint cannot be enforced, the MMK decision must be revisited and a dedicated Log Analytics cluster with CMK may be required.

**Planned sections:**

- The tracing model will define what is captured at each layer (frontend requests, API orchestration, worker execution, Service Bus message handling, SQL workflow-state changes, model interaction, search queries, document processing) and how traces are structured for correlation across asynchronous queue-driven workflows.
- The telemetry governance section will document the binding rule from ADR-005 regarding PII and business data in telemetry, define the enforcement mechanism (code review policy, automated scanning, periodic audit), and establish what the application is permitted and prohibited from emitting in trace payloads, custom dimensions, and custom metrics.
- The Log Analytics workspace architecture section will document the workspace topology, retention policy, AMPLS connectivity, and the conditions under which the architecture would escalate to a dedicated cluster.
- The diagnostic settings section will document which Azure services emit diagnostics to Log Analytics, what categories are enabled, and how diagnostic data is distinguished from application telemetry.
- The alerting strategy section will define what conditions generate alerts, severity classification, notification channels, and escalation paths. This includes Key Vault access failures (critical for CMK dependency monitoring referenced in ADR-005), Container Apps revision health and startup behaviour, Service Bus backlog and dead-letter growth, agent execution failures, and model endpoint availability.
- The queue and workflow-state observability section will define how Service Bus activity, worker execution, SQL-backed workflow state, retry behaviour, and approval checkpoints are surfaced in telemetry, and how those traces correlate with downstream agent and service traces.
- The correlation design section will define how a single business operation (e.g. document ingestion through extraction through indexing through query) is traced end-to-end across the frontend, API app, worker app, Service Bus, Azure SQL Database, AI Foundry, AI Search, and Storage.
- The audit logging section will define what constitutes an auditable event for regulated environments, where audit records are stored, retention requirements, and how audit logs relate to but are distinct from operational telemetry.
- The operational model section will define monitoring responsibilities, dashboard requirements, on-call expectations, and how observability supports the platform's operational lifecycle.

---

## Pre-Committed Decisions

The following decisions are already established by prior ADRs and will not be reconsidered in ADR-006:

- Telemetry ingestion is private-only via AMPLS (ADR-003).
- The shared Container Apps runtime identity for orchestration telemetry is `uami-platform` (ADR-004).
- Log Analytics uses MMK unless the telemetry PII constraint is violated (ADR-005).
- The telemetry PII constraint is binding; ADR-006 defines enforcement, not whether the constraint exists (ADR-005).

---

## Open Questions (to be resolved in ADR-006)

- Whether Application Insights should use workspace-based mode (recommended) or classic mode, and the implications for AMPLS connectivity.
- Whether a single Log Analytics workspace serves both application telemetry and Azure diagnostic logs, or whether separation is warranted.
- How per-agent identity traces are structured to enable agent-level audit without requiring separate Application Insights instances.
- What retention period is appropriate for operational telemetry versus audit logs, and whether audit logs require a separate store.
- Whether Service Bus, Container Apps, and Azure SQL emit sufficient native telemetry or require additional custom instrumentation within the orchestration code.
- How model interaction telemetry (token usage, latency, error rates) is captured without logging prompt or completion content.
