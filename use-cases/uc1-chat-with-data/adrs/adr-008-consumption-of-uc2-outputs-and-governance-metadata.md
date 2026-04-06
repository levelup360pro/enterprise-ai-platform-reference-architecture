# UC1-ADR-008: Consumption of UC2 Outputs and Governance Metadata

**Status**: Draft
**Date**: 02/04/2026
**Decision Scope**: How UC1 consumes UC2 outputs while preserving staging,
lineage, review, and governance boundaries.
**Depends on**: UC2-ADR-005, UC2-ADR-006, UC2-ADR-007, UC1-ADR-002,
UC1-ADR-004
**Depended on by**: knowledge-layer design and downstream implementation
guidance

---

## Context

UC2 is designed as a governed document-processing pipeline that produces structured, reviewed, and classified outputs suitable for downstream consumption. Its pipeline processes enterprise documents through classification, field extraction, confidence evaluation, optional human review, PII detection, and embedding generation before writing results to governed staging storage (ADLS Gen2). These outputs are not raw data; they carry governance context including source lineage (which document, which page, which extraction run), structured interpretation (typed fields with confidence scores), downstream suitability signals (review state, PII classification), and provenance metadata that traces each extracted field back to its source evidence.

UC2's staging storage feeds two governed downstream stores through separate mechanisms:

1. **Fabric Gold-layer Delta tables**: A Fabric Data Pipeline reads the structured extraction output from staging, parses the JSON, transforms and validates it, and writes governed columns (client name, dates, parties, obligations, contract type, amendment references) alongside governance metadata columns (review state, confidence, PII classification, extraction timestamp, source lineage). These serve the analytical retrieval plane.
    
2. **Azure AI Search index**: UC2's embedding stage (UC2-ADR-005) reads accepted, PII-classified extraction output from staging, chunks the document content, generates embeddings, and writes structured chunk documents back to staging. Each chunk document contains the chunk text, the pre-computed embedding vector, and governance metadata inherited from the extraction output (parent document ID, source page numbers, extraction run ID, document type, review state, confidence score, PII classification, extraction timestamp, security filter identifiers). The AI Search indexer then pulls these chunk documents from staging on schedule or on-demand, parses the JSON, and maps all fields (content, vector, governance metadata) to the search index schema. These serve the evidence-retrieval plane.
    

UC1's knowledge layer needs to answer questions about the same enterprise documents that UC2 processes. Because UC2 already populates both downstream stores with governed, metadata-enriched content, UC1 does not need its own extraction, chunking, embedding, or indexing pipeline. UC1 queries the same stores that UC2 populates: the Fabric Gold-layer Delta tables for analytical queries and the Azure AI Search index for evidence-retrieval queries. UC1 is a read-only downstream consumer of UC2-owned stores.

This means the governance metadata that UC2 attaches to both stores is available to UC1 at query time without an intermediate transfer step. In the AI Search index, governance metadata is present as filterable and retrievable fields on each chunk document because UC2's embedding stage propagated it from the extraction output, and the indexer mapped it to the index schema. In Fabric, governance metadata is present as governed columns because the Fabric Data Pipeline wrote it from the structured extraction output.

Consuming UC2 outputs is not just about reading extracted text fields. If UC1 queries only the visible content and ignores the governance metadata fields, it loses the ability to trace answers back to their extraction provenance, to respect review-state signals (e.g., flagging answers derived from unreviewed extractions), and to react appropriately when UC2 outputs are corrected, superseded, or withdrawn. The governance metadata is not decorative; it carries operational meaning that affects how UC1 should present and qualify its answers.

---

## Decision Drivers

- UC2 already populates both downstream stores (Fabric Gold tables and AI Search index) with governed, metadata-enriched content. There is no architectural reason for UC1 to maintain separate stores or pipelines for the same content.
- UC2's embedding stage (UC2-ADR-005) already generates embeddings and propagates governance metadata to chunk documents. Re-embedding or re-indexing this content in UC1 would duplicate compute, bypass UC2's governance chain, and create divergent interpretations.
- Reusing governed outputs is more reliable than building a second extraction, embedding, or interpretation path in UC1.
- UC1 answers should preserve lineage and governance meaning; this metadata is already present in the stores UC2 populates as first-class filterable and retrievable fields.
- The architecture should avoid any design that lets UC1 read raw extraction internals, bypass UC2 staging, or create parallel indexes.

---

## Considered Alternatives

### Option A: UC1 reads raw source systems directly and ignores UC2 downstream outputs

Under this approach, UC1 would access the original source documents (contracts, policy documents, customer records) directly from blob storage or the document management system. UC1 would perform its own extraction, chunking, and embedding generation independently of UC2, creating a parallel interpretation pipeline with its own AI Search index. UC2's governed outputs would exist but would not be consumed by UC1. This approach maximises UC1's independence from UC2's release cycle and schema decisions, and avoids any coupling between the two use cases.

Trade-off: Maximum independence between use cases, but duplicates extraction, chunking, and embedding effort; bypasses UC2's confidence evaluation and human review controls; creates two divergent interpretations of the same source documents; and doubles AI Search index storage and Azure OpenAI embedding costs for no additional value. Answers from UC1 and outputs from UC2 could contradict each other because they are derived from separate, ungoverned extraction runs.

### Option B: UC1 builds a separate ingestion pipeline that reads UC2 outputs and writes to a UC1-owned AI Search index

Under this approach, UC1 would read UC2's governed downstream outputs from staging or from the Fabric Gold layer and run its own chunking, embedding, and indexing pipeline to populate a separate UC1-owned AI Search index. Governance metadata could optionally be carried through. This creates a UC1-controlled copy of the knowledge layer, decoupling UC1's index refresh timing from UC2's pipeline cadence and indexer schedule.

Trade-off: Gives UC1 independent control over index schema and refresh timing, but re-embeds content that UC2 has already embedded (duplicating Azure OpenAI embedding costs), introduces a synchronisation lag between UC2's authoritative index and UC1's copy, creates a second index that must be kept consistent with the first, and adds pipeline complexity with no retrieval-quality benefit since the source content is identical.

### Option C: UC1 queries UC2-owned downstream stores directly and respects governance metadata at retrieval and response time (SELECTED)

Under this approach, UC1 does not maintain its own ingestion, embedding, or indexing pipeline. UC1 queries the same AI Search index that UC2 populates for evidence-retrieval, and the same Fabric Gold-layer Delta tables that UC2 populates for analytical queries. UC2 owns the write path (extraction, embedding, staging, indexing); UC1 owns the read path (retrieval, response generation, governance-aware answer qualification).

The governance metadata that UC2 attaches to both stores is available to UC1 at query time as filterable, retrievable, or facetable fields (in AI Search) and as governed columns (in Fabric). UC1's retrieval adapter (UC1-ADR-004, CA-3) reads governance metadata alongside content when executing queries. UC1's response generation logic uses this metadata to qualify answers, include provenance in citations, flag unreviewed evidence, enforce PII response policies, and detect stale or withdrawn content.

Trade-off: Tightest integration with UC2's governed outputs, zero duplication of extraction, embedding, or indexing work, single source of truth for indexed content, but creates an explicit runtime dependency on UC2's index availability and refresh cadence. UC1 cannot serve evidence-retrieval answers if UC2's AI Search index is unavailable or stale.

---

## Decision

UC1 consumes UC2's governed downstream outputs by querying the stores that UC2 owns and populates, rather than building a separate ingestion, embedding, or indexing pipeline.

**Evidence-retrieval plane (Azure AI Search):** UC1 queries the AI Search index that is populated by the AI Search indexer pulling structured chunk documents from UC2's staging storage. Each chunk document contains chunk text, pre-computed embedding vectors, and governance metadata, all produced by UC2's governed pipeline. UC1 does not re-chunk, re-embed, or re-index this content. The retrieval adapter defined in UC1-ADR-004 (CA-3) queries this index using GA APIs (2025-09-01 stable), applying security filters, hybrid search, and semantic ranking. Governance metadata fields are retrieved alongside content and passed to the response generation layer.

**Analytical plane (Fabric Gold-layer Delta tables):** UC1's Fabric Data Agent queries the Gold-layer Delta tables that UC2 populates with structured extracted fields and governance metadata columns. UC1 does not maintain a separate analytical store.

**Cross-plane queries** (as defined in UC1-ADR-002) are orchestrated by the Foundry Agent Service (UC1-ADR-001), which invokes both retrieval planes and composes the results. In both planes, UC1 is a read-only downstream consumer of UC2-owned stores.

UC1 does not read UC2's internal workflow state, intermediate extraction artefacts, or raw staging data. The consumption boundary is UC2's published downstream stores: the AI Search index and the Fabric Gold-layer Delta tables. These stores are populated through governed mechanisms designed by UC2 (see UC2-ADR-005, UC2-ADR-006, UC2-ADR-007) to provide stable, governed artefacts suitable for downstream consumption.

---

## Consequences

### Positive

- Zero duplication of extraction, chunking, or embedding work. UC2's governed pipeline produces the indexed content once; UC1 consumes it directly. This eliminates redundant Azure OpenAI embedding costs, redundant AI Search index storage, and redundant pipeline infrastructure.
- Single source of truth for indexed document content. There is one AI Search index for the evidence-retrieval plane, populated by UC2's governed pipeline via the indexer. UC1 and any future use cases read from the same index, ensuring consistent answers.
- Full governance chain preserved from source document through UC2 extraction to UC1 conversational answer. Governance metadata is available at query time as first-class index fields without an intermediate transfer step, because UC2's embedding stage propagates it to each chunk document and the indexer maps it to the index schema.
- Reinforces UC2's architectural value as a governed producer: UC2's confidence evaluation, human review, PII detection, and embedding generation controls carry through to UC1's conversational answers automatically.
- Simpler UC1 architecture. UC1 does not need an ingestion pipeline, embedding generation, or index management for the evidence-retrieval plane. UC1's responsibility is retrieval, response generation, and governance-aware answer qualification.

### Negative

- UC1 has a runtime dependency on the AI Search index availability, which in turn depends on the AI Search indexer's schedule and the timeliness of UC2's embedding stage writing chunk documents to staging. If UC2's pipeline is delayed, or if the indexer has not yet pulled the latest staged outputs, UC1's evidence-retrieval answers may be stale. The failure mode must be explicit: UC1 should report that the knowledge layer is unavailable or stale rather than falling back to an ungoverned source.
- UC1 is coupled to UC2's AI Search index schema. If UC2 changes the index schema (field names, metadata structure, chunking strategy, embedding model), UC1's retrieval adapter must be reviewed and potentially updated. This dependency must be managed through a shared schema contract.
- UC1 cannot independently control index refresh timing. If UC1 needs fresher content than UC2's pipeline cadence and indexer schedule provide, the resolution is to adjust UC2's cadence or the indexer schedule, not to build a parallel UC1 pipeline.
- Index-level access control must be coordinated between UC2 (which writes security filter identifiers to chunk documents) and UC1 (which applies security filters at query time). The security filter values written during UC2's pipeline must align with UC1's query-time authorization context.

### Constraints introduced

- UC1 must not maintain a separate AI Search index for content that UC2 already indexes. One index, one governed write path, multiple read consumers.
- UC1 must not re-extract, re-chunk, re-embed, or re-index UC2 content. The extraction, embedding, and indexing are UC2's responsibility.
- UC1 must not read UC2's raw workflow state, intermediate extraction artefacts, or internal staging data. The consumption boundary is UC2's published downstream stores (AI Search index and Fabric Gold tables).
- UC1's retrieval adapter and response generation logic must read and respect UC2 governance metadata fields (lineage, review state, confidence, PII classification, supersession status) at query and response time. These fields are available as filterable and retrievable fields in the AI Search index because UC2's embedding stage propagates them and the indexer maps them to the schema.
- If UC2 changes the structure, naming, or semantics of any governance metadata field in the AI Search index schema or Fabric Delta table columns, UC1's retrieval adapter must be reviewed and updated before the change is deployed. This dependency is managed through the shared schema contract defined in follow-up 1.
- UC1 must not re-extract or re-interpret source documents that UC2 has already processed, as this would create divergent interpretations outside UC2's governance controls.

---

## Follow-ups

- Define a formal interface contract between UC2's published outputs (AI Search index schema, Fabric Delta table schema) and UC1's retrieval adapter. This contract should specify which fields UC1 depends on, their types, and the change management process when UC2 needs to evolve the schema. The governance metadata index schema, field-level consumption rules, and Edm type mappings belong in this spec, not in this ADR.    
- Define UC1's behaviour when the AI Search index is unavailable or stale. The default should be explicit failure rather than silent degradation to an ungoverned source.    
- Validate that both UC2's pipeline cadence (writing chunk documents to staging) and the AI Search indexer schedule (pulling from staging to the index) meet UC1's freshness requirements.    
- Validate that the security filter identifiers written to chunk documents by UC2's embedding stage align with UC1's query-time authorization context.    
- Define UC1's behaviour when UC2 marks outputs as superseded or withdrawn, including query-time filtering and logging of previously served evidence that is later withdrawn.