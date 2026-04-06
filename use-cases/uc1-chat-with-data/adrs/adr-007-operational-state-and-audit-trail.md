# UC1-ADR-007: Operational State and Audit Trail

**Status**: Draft  
**Date**: 01/04/2026  
**Decision Scope**: Where UC1 stores durable operational truth for conversations, retrieval decisions, and audit metadata.  
**Depends on**: ADR-002 (Runtime Platform and EU Region Selection), ADR-006 (Observability and Auditability placeholder), UC1-ADR-005, UC1-ADR-006  
**Depended on by**: operational implementation guidance and monitoring design

---

## Context

UC1 is not a long-running workflow like UC2, but it is still a regulated enterprise assistant operating over governed data. Each conversation involves a sequence of decisions: which retrieval mode to invoke, which sources to query, whether permission-aware retrieval succeeded, whether the evidence was strong enough to ground an answer, and what response metadata to return. These decisions have governance and audit implications. If the only record of a conversation is the transient chat transcript in Teams or the ephemeral runtime logs in Application Insights, the system cannot support structured investigation, compliance reporting, or post-incident analysis with the rigour that an enterprise deployment requires.

The architecture needs durable visibility into the full lifecycle of a conversational request: what question was asked, what retrieval mode was chosen (UC1-ADR-002), which sources were queried, whether permission-aware retrieval succeeded or was denied (UC1-ADR-006), whether grounding was strong enough to produce an evidence-backed answer (UC1-ADR-005), and what source references accompanied the response. This metadata must be queryable not buried in unstructured log lines and must survive beyond the lifetime of the runtime process, the channel session, or the Azure Monitor retention window.

Relying solely on channel transcripts is insufficient because channel behaviour and retention policies are controlled by the channel provider, not by the platform. Teams chat history may be subject to organisational retention policies that do not align with the platform's audit requirements. Microsoft 365 Copilot conversation history has its own lifecycle. Neither channel provides the structured, queryable, platform-controlled record that compliance and investigation workflows require. Similarly, relying solely on Azure Monitor is insufficient because while Application Insights and Log Analytics provide excellent telemetry for runtime diagnostics, they are optimised for trace-level observability rather than structured business-event queries. 

---

## Decision Drivers

- Operational truth must survive beyond transient runtime memory and channel rendering.
- UC1 needs structured, queryable audit metadata for investigation and reporting.
- Azure Monitor is strong for telemetry, but not ideal as the sole structured operational store.
- The shared platform already treats Azure SQL as the durable operational truth store pattern.

---

## Considered Alternatives

### Option A: Runtime logs only

Under this approach, all conversation metadata (retrieval decisions, source references, permission outcomes, grounding evaluations) would be recorded as structured log entries in Application Insights and queryable through Log Analytics. No separate durable store would be maintained. Investigation and audit would be performed through Kusto queries against the Log Analytics workspace. This is the lowest-implementation-cost option because it requires no additional schema design, no database provisioning, and no write-path implementation beyond the structured logging that the application already performs for diagnostic purposes.

Trade-off: Lowest implementation cost and avoids maintaining a separate data store, but makes structured business-event queries dependent on log retention policies and Kusto query complexity. Log stores are optimised for time-series telemetry and trace correlation, not for structured queries like "show me all conversations where grounding was insufficient and the user was in the finance department last quarter."

### Option B: Channel transcript only

Under this approach, the platform would treat the channel's own conversation history as the authoritative operational record. For Teams-based delivery, the Teams chat history would serve as the audit trail. For Microsoft 365 Copilot delivery, the Copilot conversation log would serve the same purpose. No platform-side conversation record would be maintained. This approach has zero implementation cost on the platform side because the channel already stores the transcript as part of its normal operation.

Trade-off: Zero platform-side implementation cost and leverages existing channel infrastructure, but cedes control of the audit trail to the channel provider. Retention policies, data format, queryability, and availability are all governed by the channel, not by the platform. The platform cannot guarantee that the record exists, that it contains the required metadata (retrieval mode, source references, permission outcomes), or that it is accessible for structured compliance queries.

### Option C: Azure SQL for structured operational truth plus Azure Monitor for detailed telemetry (SELECTED)

Under this approach, the platform maintains a dual operational-state model. Azure SQL stores durable structured records for each conversational request, including the question, the retrieval mode selected, the sources queried, permission outcomes, grounding evaluation results, and response metadata including source references. Azure Monitor and Log Analytics store detailed runtime traces, latency metrics, error diagnostics, and distributed trace correlation. The SQL store is the authoritative source for structured audit queries; the telemetry store is the authoritative source for runtime diagnostics and performance investigation.

Trade-off: More implementation work than a logs-only approach (requires schema design, write-path implementation, and data-minimisation decisions) but provides a platform-controlled, structured, queryable audit record that does not depend on channel retention policies or log-store query patterns. Aligns with the platform's existing use of Azure SQL for UC2 operational truth.

---

## Decision

UC1 adopts a dual operational-state model.

**Azure SQL** stores durable structured operational truth for each conversational request. This includes the request identifier, the question asked, the retrieval mode selected (UC1-ADR-002), the sources queried and their identifiers, whether permission-aware retrieval succeeded or was denied (UC1-ADR-006), the grounding evaluation outcome (UC1-ADR-005), response metadata, and source references that accompanied the generated answer. This store is platform-controlled, queryable with standard SQL, and subject to the platform's own retention and access policies rather than the channel's.

**Azure Monitor / Log Analytics** stores detailed runtime traces, latency breakdowns, error diagnostics, and distributed trace correlation for each request. This telemetry supports performance investigation, anomaly detection, and operational alerting. It complements the SQL store but does not replace it for structured audit queries.

This mirrors the shared platform's pattern where durable operational truth and runtime telemetry are treated as related but distinct concerns. UC2 already uses Azure SQL for workflow state and job metadata; UC1 extends this pattern to conversational audit metadata. The two stores serve different query patterns: SQL for "what happened in this conversation and why" and Log Analytics for "how did the system perform and where did latency occur."

---

## Consequences

### Positive

- Provides a structured, queryable audit record for every UC1 conversation that can be investigated independently of channel transcripts or log retention.
- Makes conversation-level decisions (retrieval mode, permission outcome, grounding evaluation) visible as first-class queryable records rather than buried in unstructured log lines.
- Aligns with the platform and UC2 pattern of maintaining durable operational truth in Azure SQL, reducing architectural divergence between use cases.
- Supports compliance reporting workflows that require structured queries over conversation metadata (e.g., "all conversations involving finance-department users where grounding was insufficient").

### Negative

- Introduces schema design and write-path implementation work that a logs-only approach would avoid.
- Requires deliberate data-minimisation decisions about which conversational metadata is retained durably and which remains only in telemetry, particularly given GDPR requirements around personal data in conversation content.
- Adds a write dependency on Azure SQL to the response path, which must be handled asynchronously or with appropriate resilience to avoid impacting response latency.

### Constraints introduced

- UC1 must define an explicit conversation-audit schema specifying which metadata fields are retained durably in SQL and which remain only in telemetry.
- The SQL store is for operational truth and audit metadata, not for holding the enterprise corpus or search index content.
- Transcript-retention policy must be a deliberate design choice with documented rationale, not an accidental byproduct of channel defaults or log retention settings.
- The write path to Azure SQL must not block or degrade the user-facing response path; conversation metadata must be written asynchronously or with appropriate resilience patterns.

---

## Follow-ups

- Define the exact conversation-audit schema during implementation.
- Align retention and minimisation policy with GDPR and enterprise governance requirements.

---