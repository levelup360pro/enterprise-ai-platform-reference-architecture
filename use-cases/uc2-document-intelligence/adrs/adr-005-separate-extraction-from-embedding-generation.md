# UC2-ADR-005: Separate Extraction from Embedding Generation

**Status**: Draft  
**Date**: 14/03/2026  
**Decision Scope**: Whether extraction and embedding are combined or independent pipeline stages.  
**Depends on**: UC2-ADR-001 (Layered Hexagonal Architecture), UC2-ADR-004 (Worker as the Single Write Path)  
**Depended on by**: UC2-ADR-006 (Staging as the Governed Downstream Boundary)  

---

## Context

UC2 produces two distinct output types from each document: structured extraction data (classification, fields, confidence scores, PII metadata) and vector embeddings (document chunks with their embedding vectors for downstream search indexing).

Both outputs originate from the same source document. Both are written to staging. Both are consumed downstream. The question is whether extraction and embedding are performed together as a single step or as separate, sequential stages in the pipeline.

---

## Decision Drivers

- Extraction and embedding use different AI capabilities. Extraction calls Content Understanding for classification and structured field extraction. Embedding calls the Foundry embedding model for vector generation. They are different models with different failure modes, latencies, and cost profiles.
- Extraction must pass through confidence evaluation and potentially human review before the output is accepted. Embedding does not require confidence evaluation; it operates on accepted content.
- If combined, a failure in embedding generation would block or roll back an otherwise successful extraction. If separated, each stage fails independently.
- Downstream consumers differ. Fabric consumes structured extraction data. AI Search consumes embeddings. Their availability requirements and refresh cadences are independent.

---

## Considered Alternatives

### Option A: Combined extraction and embedding

A single pipeline stage calls Content Understanding for extraction and the embedding model for vector generation. Both outputs are produced together, evaluated together, and written to staging together.

Trade-off: Simpler sequencing. One stage, one write. But conflates two capabilities with different failure modes. An embedding failure forces the entire stage to retry, including extraction. Confidence evaluation would need to cover both extraction quality and embedding quality, which are unrelated concerns. Human review of low-confidence extractions would block embedding generation for documents that might not need review at all.

### Option B: Parallel extraction and embedding

Both stages run concurrently on document arrival. Extraction feeds through confidence evaluation and review. Embedding runs independently. Results are joined before staging write.

Trade-off: Faster for high-confidence documents. But creates a join dependency. If extraction enters human review, the embedding result must wait. If embedding fails, the join blocks staging even though extraction succeeded. Coordination complexity increases without clear benefit, since embedding depends on accepted content to be useful downstream.

### Option C: Sequential separation (SELECTED)

Extraction runs first, passes through confidence evaluation and human review, then PII classification. Embedding generation runs after accepted, PII-classified output has been written to staging. The two stages share no state other than the document reference and the accepted output.

Trade-off: Embedding cannot start until extraction is fully accepted. For documents that enter human review, this adds latency to embedding availability. But embedding operates on governed, accepted content. A failure in embedding does not affect the extraction output already in staging. Each stage can be retried independently.

---

## Decision

Extraction and embedding are separate, sequential pipeline stages.

Extraction runs first: classification, structured extraction, confidence evaluation, human review (if required), PII classification, and staging write. This produces the governed structured output.

Embedding runs second: the Worker retrieves accepted content, calls the Foundry embedding model, receives vectors, and writes document chunks with embeddings to staging. The search index client then makes these available to AI Search.

Extraction and embedding have independent workflow status progressions. A failure in embedding does not change the extraction status. A document can complete the extraction path and be available to Fabric without embeddings being available to AI Search. The two capabilities fail, retry, and complete independently.


---

## Consequences

### Positive

- Extraction and embedding fail independently. A transient embedding model failure does not block or roll back a successful extraction. The structured output is available in staging for Fabric consumption regardless of embedding status.
- Retry scope is narrow. If embedding fails, only embedding is retried. The extraction, confidence evaluation, human review, and PII classification steps are not repeated.
- Downstream consumers are decoupled. Fabric receives structured data from staging as soon as extraction completes. AI Search receives embeddings when embedding completes. Neither blocks the other.
- The separation aligns with the core domain model (UC2-ADR-001). Extraction and embedding have different responsibilities, different external service dependencies, and different downstream consumers. Combining them would conflate two independent concerns.

### Negative

- Embedding cannot begin until extraction is fully accepted and staged. For documents entering human review, embedding availability is delayed by the review cycle.
- Two sequential capabilities mean two independent progress states to manage and monitor. Operational dashboards must track both extraction and embedding progress per document.

### Constraints introduced

- Embedding generation must only operate on accepted, PII-classified, staged extraction output. It must not run on raw document content or unreviewed extraction results.
- A document may complete the extraction path without embeddings being available. Downstream consumers must handle this state.
- Extraction and embedding failures are tracked independently in Azure SQL. A failed embedding does not revert the extraction workflow state.

