# UC2-ADR-009: Source-Aware Ingestion and Metadata Preservation

**Status**: Draft  
**Date**: 23/03/2026  
**Decision Scope**: How documents enter the UC2 processing pipeline from source systems, and how source metadata is preserved through workflow state and governed staging.  
**Depends on**: UC2-ADR-004 (Worker as the Single Write Path), UC2-ADR-006 (Governed Staging Boundary), UC2-ADR-007 (Azure SQL as Workflow State Store)  
**Depended on by**: UC2-ADR-010 (Duplicate Handling and Reprocessing Semantics)

---

## Context

UC2 processes documents that originate from multiple source systems: document management systems, including but not limited to SharePoint Online, CRM platforms such as Dynamics 365, network file shares, email archives, and direct user uploads.

Documents originate in source systems that differ in their APIs, authentication models, metadata structures, and governance controls. The ingestion boundary must standardise how documents and their source metadata enter the processing pipeline, regardless of where they originate. It must also accommodate different organisational requirements around document duplication. Some environments accept copies in platform-controlled storage. Others require that original documents remain solely in the source system.

Source systems often already hold metadata about a document that is useful to the pipeline. In SharePoint, that may include site, library, folder path, content type, item ID, and metadata columns such as client reference or document type. In Dynamics 365, that may include entity type, record ID, and attachment relationship context. In a document management system, that may include document class, version, access control metadata, and retention status. On a file share, the folder structure and naming convention may carry business meaning. If a document enters the processing pipeline without preserving that metadata, the pipeline starts with less information than the organisation already has.

The source relationship also matters after extraction. A user who finds a document through search, analytics, or a downstream operational process may need to navigate back to the original source document. If the source reference is not preserved, traceability is broken and the user must perform a manual search outside the system.

Different source systems also require different retrieval logic, authentication models, and metadata extraction approaches. The processing pipeline should not need to understand source-system-specific APIs or storage semantics. That concern belongs in the ingestion layer.

Document duplication at ingestion is a material concern. Source systems often already enforce their own governance, access control, retention, and lifecycle policies around the documents they hold. Copying those documents into separate platform-controlled storage creates a second copy that exists outside the source system's governance model. This introduces storage cost, retention complexity, and potential compliance exposure. Organisations that already govern their documents within the source system may not accept any duplication outside the source system, even temporarily during processing.

The question is how UC2 standardises document entry into the processing pipeline, preserves source metadata and source reference, and supports different organisational requirements around whether source documents are copied, copied temporarily, or left solely in the source system.

---

## Decision Drivers

- Documents originate from multiple source systems with different APIs, authentication models, and metadata structures. The processing pipeline must remain source-agnostic.
- Source metadata already held by the source system has value for classification hinting, downstream traceability, and operational context. Discarding it at ingestion is wasteful.
- The Worker is the single write path for workflow state (UC2-ADR-004). Ingestion must not introduce a parallel write path into workflow state or governed staging.
- Governed outputs written to staging (UC2-ADR-006) must retain enough context for downstream consumers to trace back to the original source document.
- New source systems will be added over time. The architecture must support onboarding a new source without modifying the processing pipeline.
- Source systems may already enforce governance, access control, and retention policies over their documents. The ingestion boundary should not undermine those controls by creating ungoverned copies by default.
- Some organisations will not accept persistent document duplication outside the source system. The architecture must accommodate that constraint without sacrificing processing reliability.

---

## Considered Alternatives

### Option A: Storage-triggered ingestion with attached metadata

Documents are copied into platform-controlled ingestion storage by source-specific mechanisms. Metadata is attached to the stored document in whatever form the storage platform supports. Processing starts from the presence of the document in storage.

Trade-off: Simple and storage-native. But storage-attached metadata is constrained in size, structure, and consistency. Rich source context is difficult to represent reliably. Different source systems would encode metadata differently, forcing the processing layer to interpret or normalise source-specific formats. This weakens the boundary between ingestion and processing and introduces source-system awareness into the core workflow. It also assumes persistent duplication of every ingested document, which may not be acceptable.

### Option B: Manifest-based ingestion

The ingestion process writes the document and a separate metadata manifest into platform-controlled storage. Processing begins from the manifest, which references the document and carries source context.

Trade-off: A manifest can represent rich structured metadata and avoids the limitations of storage-native metadata fields. But it introduces a two-object consistency problem. If the document and manifest are not kept in sync, the pipeline has an orphaned state that requires additional operational handling. The ingestion boundary remains storage-driven rather than contract-driven. Persistent duplication is still the default.

### Option C: Source-aware ingestion components with a standard ingestion contract and persistent ingestion copy

Each source system has a dedicated ingestion component responsible for retrieving the document, capturing source metadata, writing the document into platform-controlled ingestion storage, and passing a standard ingestion contract into the processing pipeline. The Worker begins from that contract rather than discovering documents directly from storage. The ingested document persists in platform-controlled storage.

Trade-off: More components to build and maintain. Each source system requires dedicated ingestion logic. But the processing pipeline remains source-agnostic. Source metadata is standardised through an explicit contract. The architectural boundary becomes clear: source-aware components own retrieval and metadata capture; processing components own classification, extraction, review, and staging. However, every ingested document exists in two places, the source system and ingestion storage, creating a persistent duplication concern. This requires a retention and lifecycle policy for ingestion storage aligned with the organisation's governance model.

### Option D: Source-aware ingestion components with a standard ingestion contract and transient ingestion copy

Same as Option C, but the copy in platform-controlled ingestion storage is transient. After the processing pipeline has successfully completed extraction and the governed output has been written to staging, the ingestion copy is deleted. The governed output in staging retains the source reference, the extracted fields, and the metadata. The original document remains only in the source system.

Trade-off: Avoids persistent duplication. The document exists in two places only during active processing. Once the workflow completes successfully, the ingestion copy is removed and the source system remains the single authoritative location for the original document. This respects the source system's governance model while still allowing the pipeline to operate from a controlled, source-agnostic storage location during processing. However, it introduces a deletion step that must be reliable and must only execute after confirmed successful processing. If deletion fails or is skipped, the system falls back operationally to the persistent-copy behaviour of Option C. Reprocessing or re-extraction requires re-retrieving the document from the source system, which introduces a dependency on source availability at reprocessing time.

### Option E: Source-aware ingestion components with a standard ingestion contract and no ingestion copy

Each source system has a dedicated ingestion component responsible for reading the document from the source, preparing the extraction input, capturing source metadata, and passing that extraction input together with a standard ingestion contract into the processing pipeline. The original document never leaves the source system. No copy is created in platform-controlled ingestion storage.

Trade-off: Eliminates document duplication entirely. The source system remains the sole location of the original document. This is the cleanest approach from a data governance perspective and is preferred by organisations that already govern their documents within the source system. However, it moves more responsibility into the ingestion component. The ingestion component must not only retrieve and describe the document but also prepare it for extraction. Reprocessing requires re-reading from the source system. If the source system is unavailable, reprocessing is blocked until it recovers. The pipeline also loses the ability to independently re-examine the original file, since it never holds a copy.

### Option F: Direct processing from source platforms without an ingestion boundary

Documents remain in their original systems and are accessed directly by processing components, without creating a platform-controlled ingestion copy and without a formal ingestion boundary.

Trade-off: Avoids document duplication entirely. But it weakens control over ingestion, repeatability, and auditability. Event-driven processing may be unavailable or inconsistent depending on the source platform. Source availability becomes a direct runtime dependency of the processing workflow. Processing components must understand source-specific APIs. This pattern is better suited to downstream access than to primary ingestion.

### Option G: Source-aware ingestion components with a standard ingestion contract and configurable ingestion mode (SELECTED)

Each source system type has a dedicated ingestion component responsible for retrieving or accessing the document, capturing source metadata, and submitting a standard ingestion contract to the processing pipeline. The handling of the document content at the ingestion boundary is not fixed. The architecture supports three configurable modes within the same contract and pipeline design: persistent copy (the document is written to platform-controlled storage and retained), transient copy (the document is written to platform-controlled storage and deleted after successful processing), and no copy (the document is read from the source and prepared for extraction without creating a platform-controlled copy). The ingestion mode is a deployment-level configuration decision. The processing pipeline consumes the same standard contract regardless of which mode was used.

Trade-off: This is the most flexible approach. It accommodates organisations with different data governance requirements within a single architectural design. The processing pipeline remains source-agnostic and mode-agnostic. However, it introduces the most design complexity at the ingestion boundary. Each ingestion component must support whichever mode is configured, or the mode must be fixed per source type. The ingestion contract must represent the extraction input consistently across modes where the underlying access mechanism differs (blob path, stream reference, structured payload). Contract design and versioning become more critical because the contract must serve all three modes without leaking mode-specific assumptions into the processing pipeline.

In practice, most organisations will select a single ingestion mode and apply it consistently across all source systems. The architecture supports multiple modes not because a single deployment is expected to mix them, but because different organisations have different data governance requirements. Providing the flexibility at the architectural level means the same pipeline design can be deployed in environments that accept document copies and environments that do not, without redesign.

---

## Decision

UC2 uses **source-aware ingestion components that deliver documents and source metadata into the processing pipeline through a standard ingestion contract**. The architecture supports multiple ingestion modes to accommodate different data governance requirements regarding document duplication.

Each ingestion component is responsible for one source type or ingestion path. It retrieves the document from the source, captures the available source metadata, and submits an ingestion message conforming to a standard contract. The specific handling of document content at ingestion depends on the ingestion mode selected for the deployment context.

The architecture supports three ingestion modes:

**Persistent copy mode** (Option C). The ingestion component writes the document into platform-controlled ingestion storage. The document persists in ingestion storage alongside its presence in the source system. A retention and lifecycle policy governs how long the ingestion copy is retained. This mode provides the highest operational independence from the source system and supports reprocessing without re-retrieval, at the cost of persistent duplication.

**Transient copy mode** (Option D). The ingestion component writes the document into platform-controlled ingestion storage. After the processing pipeline confirms successful completion and the governed output is written to staging, the ingestion copy is deleted. The document exists in two places only during active processing. This mode balances operational simplicity with the constraint of avoiding persistent duplication. Reprocessing requires re-retrieval from the source system.

**No copy mode** (Option E). The ingestion component reads the document from the source system and prepares an extraction input without writing a persistent or transient copy to platform-controlled storage. The original document never leaves the source system. This mode eliminates duplication entirely and is appropriate when the source system's governance model prohibits external copies. It increases ingestion-component complexity and creates a runtime dependency on source availability for both initial processing and reprocessing.

The ingestion mode is a deployment-level configuration decision, not a per-document decision. In practice, most organisations will select a single mode and apply it uniformly. The architecture defines all three modes so that the same pipeline design can serve different governance contexts without requiring structural changes. All three modes use the same standard ingestion contract. The processing pipeline does not need to know which mode was used. The contract provides what the pipeline needs: a way to identify the extraction input, the source reference, and the source metadata. The exact representation is defined separately from this ADR.

The ingestion contract contains, at minimum, the information required to uniquely identify the ingestion event, locate or access the extraction input, identify and reference the originating source system, preserve source metadata captured at ingestion, indicate the ingestion mode used, and support traceability through workflow state and staged output. The exact contract shape is defined and versioned separately from this ADR.

The processing pipeline begins from that ingestion contract. The Worker does not call source systems directly and does not interpret source-system-specific APIs. It treats the source reference, source metadata, and extraction input reference as data supplied by the ingestion boundary.

At workflow start, the Worker records the source reference and source metadata in workflow state together with the document's processing state. That metadata is then preserved through the pipeline as contextual information. It is not owned or reinterpreted by classification, extraction, confidence evaluation, PII tagging, or review.

When governed output is written to staging, it includes both the extracted result and sufficient source context to support downstream traceability back to the original document in the source system.

Manual uploads follow the same pattern. They are treated as a source type entering through the same ingestion boundary and standard contract.

---

## Consequences

### Positive

- The processing pipeline remains source-agnostic. Source retrieval and source metadata capture are isolated to the ingestion boundary.
- Source metadata is preserved from ingestion through to governed staging. Downstream consumers can trace outputs back to the original source document.
- Existing metadata from source systems is available from the start of the workflow. It can inform processing decisions without coupling the Worker to any particular source platform.
- The architecture supports new source systems additively. A new ingestion component can conform to the same contract without requiring changes to the Worker or downstream pipeline stages.
- The ingestion boundary becomes explicit. Source-aware concerns stay at the edge; document processing concerns stay in the core workflow.
- Document duplication is not imposed as a default. The architecture accommodates organisations that require no persistent copies outside the source system, organisations that accept transient copies during processing, and organisations that prefer persistent ingestion copies for operational resilience. The choice is a deployment decision, not an architectural constraint.

### Negative

- Each source system requires dedicated ingestion logic. This introduces additional implementation and maintenance effort.
- The ingestion contract becomes a shared dependency between ingestion components and the processing pipeline. Contract changes must be versioned and managed carefully.
- In persistent copy mode, platform-controlled ingestion storage contains a copy of source documents. This increases storage usage and introduces retention and lifecycle considerations.
- In transient copy mode, the deletion step must be reliable and must only execute after confirmed successful processing. Deletion failure must be detected and handled operationally.
- In no copy mode, the ingestion component carries increased responsibility for preparing the extraction input. Source system availability becomes a runtime dependency for both initial processing and reprocessing.
- Duplicate ingestion remains possible if a source-aware component re-reads previously seen documents or if the same document enters through multiple ingestion paths. Duplicate handling is not resolved by this decision and is addressed in UC2-ADR-010.

### Constraints introduced

- Documents are valid pipeline inputs only when they enter through the standard ingestion boundary and contract. A raw file placed directly into ingestion storage is not, by itself, a complete pipeline input.
- Source metadata is treated as preserved contextual data within the processing pipeline. Processing stages must not discard it or replace it with source-specific reinterpretation.
- New source systems must integrate through the ingestion boundary rather than by introducing source-specific logic into the Worker.
- The ingestion mode must be selected as a deployment-level decision and documented. The processing pipeline must not assume a specific ingestion mode.