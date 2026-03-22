# UC2-ADR-007: Azure SQL as the Workflow System of Record

**Status**: Draft
**Date**: 17/03/2026
**Decision Scope**: Where workflow state and operational truth are persisted.
**Depends on**: Enterprise AI Platform reference architecture, UC2-ADR-001 (Layered Hexagonal Architecture), UC2-ADR-003 (Confidence-Based Workflow Routing), UC2-ADR-004 (Worker as the Single Write Path)
**Depended on by**: None

---

## Context

UC2 document processing involves a multi-step workflow with explicit status transitions. Each transition records what happened, when, and why. The human review loop means a document can pause for hours or days between being flagged for review and being approved or rejected.

The system needs a durable store for this state that survives Worker restarts, supports queries from the API and Frontend (job status, review queue, operational dashboards), and provides the audit trail that regulators, security teams, and operations staff rely on.

The question is where this state lives and what role it plays relative to other stores in the system.

---

## Decision Drivers

- Workflow state must survive Worker and container restarts. Azure Container Apps can scale to zero or restart at any time. State that only exists in memory or in-flight messages is lost on restart.
- The human review loop requires persistent state that the Frontend and API can query independently of the Worker. A reviewer opening the queue hours after extraction must see the current state and the flagged fields without the Worker being active.
- Operational dashboards need to answer: how many documents are in each status, what is the average review wait time, how many were rejected this week, what is the extraction-to-staging throughput. These are relational queries over structured workflow data.
- The audit trail must be queryable after the fact. "Show me every document processed on this date, which path it took, what confidence scores it had, and who reviewed it" is a reporting requirement, not a log search.

---

## Considered Alternatives

### Option A: Queue as state

Service Bus messages carry workflow state. Each stage reads state from the inbound message, processes, and publishes an updated message to the next stage. No external state store.

Trade-off: No database dependency. But state exists only in messages. If a message is consumed and the Worker fails before publishing the next message, state is lost. Dead-letter queues capture failures but do not provide queryable workflow history. The Frontend cannot query Service Bus for job status or review queues. Operational reporting requires reconstructing state from message logs, which is fragile and slow.

### Option B: Storage-based state

Workflow state written as JSON files in Blob Storage or ADLS Gen2. Each status transition appends or overwrites a state file per document.

Trade-off: Simple, no database. But querying across documents (all items pending review, average processing time, throughput metrics) requires scanning files. No transactional guarantees on concurrent updates. The Frontend and API would need a custom query layer over storage. Not suited for the relational queries that operational dashboards require.

### Option C: Event sourcing

Every state transition is an immutable event appended to an event store. Current state is derived by replaying events. Full history preserved by design.

Trade-off: Complete audit trail with no data loss. But adds significant complexity. Requires an event store (Cosmos DB, Event Hubs, or custom), projection logic to derive current state, and a read model for queries. The operational queries UC2 needs (current status, review queue, throughput) are straightforward relational queries that do not justify the event sourcing infrastructure. The audit trail requirement is met by recording each transition, which does not require full event sourcing.

### Option D: Azure SQL as system of record (SELECTED)

Azure SQL holds the workflow state for UC2. It persists the current workflow record for each document together with transition metadata required for audit and operational reporting, including timestamps, confidence scores, reviewer identity, and rejection reason where applicable. The Worker writes state. The API reads state for job status and review queue queries. Operational dashboards query workflow state through approved reporting or API access patterns.

Trade-off: Relational database dependency. But Azure SQL is already part of the platform baseline. The queries UC2 needs are inherently relational. Transactional writes ensure state consistency. The Frontend and API read state independently of the Worker. Audit trail is a query, not a reconstruction.

---

## Decision

Azure SQL is the workflow system of record for UC2.

Every workflow state transition is recorded in Azure SQL by the Worker (UC2-ADR-004). The workflow record captures sufficient metadata to reconstruct the processing path, support the human review queue, and satisfy audit and operational reporting requirements.

The full structured extraction output is not stored in SQL. SQL holds the workflow metadata, the reference to the staged output, and the audit data. The extraction payload lives in staging (UC2-ADR-006).

The API layer reads workflow state from SQL to serve the Frontend: job status queries, the human review queue (documents pending review with their flagged fields), and operational metrics. These reads do not go through the Worker. The API queries SQL directly.

Azure SQL is the single source of truth for what happened to a document and why. Staging is the source of truth for what was produced. They are complementary, not overlapping.

---

## Consequences

### Positive

- Workflow state survives container restarts, scaling events, and Worker redeployments. A document pending review remains in that state regardless of Worker lifecycle.
- The review queue, job status, and operational dashboards are relational queries against structured workflow data. No log parsing, message reconstruction, or file scanning required.
- The audit trail is a direct query: all transitions for a document, all documents reviewed by a specific reviewer, all rejections in a date range. This satisfies operational, compliance, and incident investigation requirements.
- The Worker writes state. The API reads state. The separation is clean.

### Negative

- Azure SQL is a stateful dependency. Availability, backup, and recovery of the SQL database directly affect UC2's ability to process documents and serve the review queue.
- Workflow metadata accumulates over time. Long-running deployments need a retention or archival strategy for completed workflow records.

### Constraints introduced

- The Worker is the only component that writes workflow state to SQL. The API reads but does not write workflow transitions. Human review decisions flow through the API to Service Bus to the Worker, which then writes the state update.
- SQL holds workflow metadata and audit data, not extraction payloads. The extraction output reference in SQL points to the staged output in ADLS Gen2.
- Every status transition must be recorded with timestamped audit metadata sufficient to reconstruct the workflow path. Gaps in the transition history indicate a failure that must be investigated.
