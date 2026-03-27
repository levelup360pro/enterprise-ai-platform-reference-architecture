# UC2-ADR-004: Worker as the Single Write Path

**Status**: Draft
**Date**: 14/03/2026
**Decision Scope**: Which component owns authoritative writes to workflow state and staging.
**Depends on**: UC2-ADR-001 (Layered Hexagonal Architecture)
**Depended on by**: UC2-ADR-002 (Post-Extraction PII Classification), UC2-ADR-003 (Policy-Driven Workflow Routing), UC2-ADR-005 (Separate Extraction from Embedding Generation), UC2-ADR-006 (Staging as the Governed Downstream Boundary), UC2-ADR-007 (Azure SQL as the Workflow System of Record)

---

## Context

UC2 processes documents through a multi-stage pipeline running inside the Worker. Each stage calls an external AI capability (Content Understanding, Azure AI Language, embedding model) that returns results to the Worker. None of these external services write to downstream stores. They are request/response APIs.

The Worker is the only processing component in the architecture. The stages are not separate services. The question is not which service writes, but whether other application entry points (the Frontend, the API layer) may also write to workflow state or staging as part of the processing lifecycle, particularly during human review.

---

## Decision Drivers

- The audit trail in Azure SQL must be complete by construction. If multiple application components can write workflow state, audit completeness depends on every component logging correctly.
- Human review decisions originate in the Frontend and flow through the API. The API could write the review outcome directly to SQL and the accepted output directly to staging, bypassing the Worker. That would create a second write path with its own failure modes and audit obligations.
- Downstream consumers (Fabric, AI Search) must consume only governed outputs. If multiple components can write to staging, the governance guarantee of the staging boundary (UC2-ADR-006) depends on every writer enforcing the same controls.

---

## Decision

The Worker  is the only component that performs authoritative writes to workflow state (Azure SQL) and governed outputs (Staging Storage).

All external AI service results return to the Worker. The Worker evaluates results, updates workflow state, and writes governed outputs. No other application component writes to staging or advances the workflow.

Human review decisions flow from the reviewer through the Frontend and API, but the API does not write to staging or update workflow state directly. The API posts a resume message to Service Bus. The Worker consumes the message, reads the review decision from SQL, and resumes the processing pipeline from the correct step. The Worker performs the subsequent writes.

This ensures a single write path for all governed outputs regardless of whether the document was auto-approved or human-reviewed.

---

## Consequences

### Positive

- The audit trail is complete by construction. Every workflow state transition and every staging write passes through one component.
- The staging boundary guarantee (UC2-ADR-006) is enforced by a single writer. There is no second path that could bypass confidence evaluation, PII classification, or review approval.
- Human review resumption is unambiguous. The Worker resumes from the recorded workflow state. There is no split-brain between what the API wrote and what the Worker expected.

### Negative

- The Worker is a throughput constraint. All documents flow through Worker instances. Scaling requires horizontal scaling of the Worker, not independent scaling of review processing.
- The human review path has an additional hop. The API cannot write the outcome directly. It must queue a message and wait for the Worker to resume. This adds latency to the review completion path.

### Constraints introduced

- Only the Worker may write to staging or update workflow state as part of the document processing lifecycle.
- The Frontend and API layer read workflow state for display and review purposes. They do not write governed outputs or advance the workflow directly.
- Human review decisions return through Service Bus to the Worker. The API queues a resume message. The Worker performs the write.
