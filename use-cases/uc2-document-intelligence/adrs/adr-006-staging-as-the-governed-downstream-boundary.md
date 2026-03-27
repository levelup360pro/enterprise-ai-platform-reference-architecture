# UC2-ADR-006: Staging as the Governed Downstream Boundary

**Status**: Draft
**Date**: 17/03/2026
**Decision Scope**: Where the boundary sits between UC2 processing and downstream consumers.
**Depends on**: UC2-ADR-001 (Layered Hexagonal Architecture), UC2-ADR-002 (Post-Extraction PII Classification), UC2-ADR-003 (Policy-Driven Workflow Routing), UC2-ADR-004 (Worker as the Single Write Path), UC2-ADR-005 (Separate Extraction from Embedding Generation)
**Depended on by**: None

---

## Context

UC2 produces two categories of output: structured extraction data (with PII metadata) and document chunks with vector embeddings. These outputs are consumed by Microsoft Fabric (for the relationship-oriented analytics view) and Azure AI Search (for downstream indexing and eventual querying by UC1).

The question is where the handoff boundary sits between UC2's processing pipeline and these downstream consumers. Specifically: what qualifies an output to cross that boundary, and what guarantees does the boundary provide to anything reading from it.

---

## Decision Drivers

- Downstream consumers must not receive extraction outputs that have not passed confidence evaluation, human review (where required), and PII classification. The boundary must enforce this by construction, not by convention.
- Fabric and AI Search have different consumption patterns. Fabric reads structured data into Delta tables. AI Search indexes document chunks and embeddings. Both need a stable, governed location to read from.
- The Worker owns all authoritative writes (UC2-ADR-004). The boundary must be a location the Worker writes to, not a location that downstream consumers share with the processing pipeline.
- Staging must be distinguishable from raw document storage (Blob) and workflow state (Azure SQL). Each store has a different purpose and different access controls.

---

## Considered Alternatives

### Option A: Direct write to downstream systems

The Worker writes extraction outputs directly to Fabric Delta tables and pushes embeddings directly to AI Search. No intermediate staging layer.

Trade-off: Fewer storage layers. But couples the Worker to downstream system availability and schema. If Fabric is unavailable, the Worker's pipeline stalls. If the AI Search index schema changes, the Worker must change. The Worker becomes responsible for downstream write failures, retry logic, and schema compatibility in addition to its orchestration responsibilities.

### Option B: Shared storage with access controls

Extraction outputs and raw documents share the same Blob Storage account, separated by container or path. Access controls distinguish who can read what.

Trade-off: Simpler infrastructure. But the boundary between raw and governed content is a path convention, not a structural separation. A misconfigured access policy could expose raw, unreviewed content to a downstream consumer. The governance guarantee depends on correct configuration rather than architectural separation.

### Option C: Staging as a dedicated governed boundary (SELECTED)

ADLS Gen2 staging is a separate storage layer that only contains outputs that have completed the full UC2 processing pipeline: confidence evaluation passed or human review approved, PII classified, and written by the Worker. Downstream consumers read exclusively from staging. They do not access Blob Storage, Azure SQL, or any intermediate processing state.

Trade-off: An additional storage layer to provision and manage. But the boundary is structural. If data is in staging, it has been governed. Downstream consumers do not need to independently verify whether an output has been reviewed or tagged; the controlled write path to staging establishes that governance boundary.

---

## Decision

ADLS Gen2 staging is the governed downstream boundary for UC2.

Only outputs that have completed the full processing pipeline are written to staging. For structured extraction data, this means the output has passed confidence evaluation (either automatically or through human review approval), been classified for PII, and been enriched with PII metadata. For embeddings, this means the vectors were generated from accepted and governed content and written to staging as a downstream derivative (UC2-ADR-005).

The Worker is the only component that writes to staging (UC2-ADR-004). No processing-stage component, no external capability, and no downstream consumer writes to staging.

Downstream consumers read from staging only:

**Microsoft Fabric** reads structured extraction data from staging into the Relationship Intelligence Delta Table via Trusted Workspace Access, joining it with enterprise data flowing through the Medallion Architecture.

**Azure AI Search** indexes document chunks and embeddings from staging via the AI Search Indexer, governed by a Resource Instance Rule. The indexer reads from staging on schedule or on demand. It does not access Blob Storage or Azure SQL.

Staging contains two output categories, stored separately: structured extraction data (JSON with PII metadata) and document chunks with embeddings. Both are keyed by document reference and linked to the workflow record in Azure SQL, but staging itself does not contain workflow state.

---

## Consequences

### Positive

- The governance guarantee is structural. If an output is in staging, it has passed confidence evaluation, human review where required, and PII classification. Downstream consumers do not need to independently verify processing status.
- Downstream system availability does not affect the processing pipeline. If Fabric or AI Search is temporarily unavailable, staged outputs persist and are consumed when the downstream system recovers. The Worker's pipeline is not blocked by downstream failures.
- Raw documents (Blob), workflow state (Azure SQL), and governed outputs (staging) are structurally separated. Access controls, encryption posture, and retention policies can differ by store without path-level configuration complexity.

### Negative

- An additional storage layer adds infrastructure cost and operational surface. Staging must be provisioned, monitored, and included in backup and retention policies.
- There is a propagation delay between staging write and downstream consumption. Fabric and AI Search read on their own schedules. A document that has been written to staging is not immediately available in the Relationship Intelligence Semantic Model or the search index.

### Constraints introduced

- No downstream analytics or indexing consumer may read from Blob Storage, Azure SQL, or any intermediate processing state. Staging is the only read boundary for Fabric and Azure AI Search.
- Staging must not contain raw, unreviewed, or unclassified outputs. The Worker enforces this by writing only after the full processing sequence completes.
- Structured extraction data and embeddings are stored separately within staging. They share a document reference key but have independent write timing (UC2-ADR-005).
