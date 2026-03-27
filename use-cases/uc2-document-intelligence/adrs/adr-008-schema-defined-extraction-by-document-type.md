# UC2-ADR-008: Schema-Defined Extraction by Document Type

**Status**: Draft
**Date**: 19/03/2026
**Decision Scope**: How extraction schemas are selected and managed across document types.
**Depends on**: UC2-ADR-001 (Layered Hexagonal Architecture), UC2-ADR-003 (Policy-Driven Workflow Routing), UC2-ADR-004 (Worker as the Single Write Path)
**Depended on by**: None

---

## Context

UC2 processes multiple document types: master leases, amendments, side letters, NDAs, termination notices, and correspondence. Each type contains different fields, different structures, and different information that matters to the business. A master lease contains parties, jurisdiction, commencement date, term, rental obligations, and termination clauses. An amendment contains the reference to the original contract, the specific clauses being modified, and the effective date of the change. An NDA contains parties, confidentiality scope, duration, and governing law.

Content Understanding performs structured extraction against a defined schema. The schema tells the model what fields to extract, what types they are, and where inference is permitted. The quality of extraction depends directly on how well the schema matches the document type being processed.

The question is whether UC2 uses a single generic schema for all documents or defines explicit schemas per document type, and how schema selection is managed in the pipeline.

---

## Decision Drivers

- Different document types contain fundamentally different fields. A single schema broad enough to cover all types would contain mostly empty fields for any given document, reducing extraction precision and making confidence evaluation unreliable.
- Confidence thresholds (UC2-ADR-003) are defined per field. If the field set changes per document type, thresholds must be scoped to the schema that produced the extraction. A generic schema makes threshold management ambiguous.
- Downstream consumers in Fabric expect consistent, typed data per document category. A Relationship Intelligence Delta Table with unpredictable field populations is difficult to query and unreliable for the Relationship Intelligence Semantic Model.
- New document types will be onboarded over time. The architecture must support adding a new type without modifying existing schemas or disrupting extraction for types already in production.

---

## Considered Alternatives

### Option A: Single generic schema

One schema covering all possible fields across all document types. Every document is extracted against the same schema. Fields that do not apply to a given type are returned as null or empty.

Trade-off: Simple to manage. One schema, one extraction call, one output shape. But precision degrades. The model attempts to extract fields that do not exist in the document, producing low-confidence nulls or false matches. Confidence evaluation becomes noisy because null fields with low confidence are expected rather than exceptional. Downstream consumers must handle sparse, unpredictable output. Adding a new document type means expanding the single schema, which risks affecting extraction quality for existing types.

### Option B: Schema per document type with classification-driven selection (SELECTED)

Each supported document type has its own extraction schema defining the fields, types, and inference rules relevant to that type. The DocumentClassifier determines the document type first. The Worker then selects the corresponding schema and passes it to Content Understanding for extraction. New document types are onboarded by adding a new schema and registering it with the classifier.

Trade-off: More schemas to define and maintain. Classification must be accurate for the correct schema to be selected; a misclassification produces extraction against the wrong schema. But each extraction is precise against its target type. Confidence thresholds are meaningful because the expected field set is defined explicitly for that document type, with required and optional fields distinguished. Downstream consumers receive consistent, fully populated output per type. New types are additive; existing schemas are unchanged.

### Option C: Dynamic schema inference

Content Understanding infers the schema from the document content without a predefined schema. The model determines what fields exist and extracts them.

Trade-off: Zero configuration per document type. But output shape is unpredictable. Confidence evaluation cannot define thresholds for fields it does not know in advance. Downstream consumers cannot rely on a stable contract for what the extraction will produce. PII classification (UC2-ADR-002) and staging governance (UC2-ADR-006) depend on knowing what fields exist in the output. Dynamic inference breaks that chain.

---

## Decision

UC2 uses explicit schemas per supported document type, selected by classification.

The pipeline operates in two stages. First, the Worker performs document classification through the DocumentClassifier to determine its type. Second, the Worker selects the extraction schema registered for that document type and passes it to Content Understanding for structured extraction.

Each schema defines the fields to extract, their types, which fields are required versus optional, and which fields permit inference, for example calculating an end date from a start date and duration. Each schema has a corresponding set of confidence thresholds registered with the ConfidenceEvaluator (UC2-ADR-003).

Schemas are configuration, not code. They are managed as versioned artefacts alongside their confidence thresholds. Adding a new document type requires three things: a new classification label registered with the DocumentClassifier, a new extraction schema, and a corresponding set of confidence thresholds. Onboarding a new document type is additive by design and should not require modification of existing document-type schemas or thresholds.

Documents classified as unsupported or unrecognised do not proceed to extraction. They are recorded in Azure SQL with an explicit non-processable classification state and routed to the review queue for manual triage.

---

## Consequences

### Positive

- Extraction precision is maximised per document type. The expected field set is explicit for the document type, with required and optional fields defined in advance. Confidence scores reflect genuine extraction quality, not the absence of irrelevant fields.
- Confidence thresholds are scoped to their schema. A threshold for "jurisdiction" in a master lease schema operates against a field that should be present, not a field that might not exist in an amendment.
- Downstream consumers receive a predictable output shape per document type. The Relationship Intelligence Delta Table in Fabric can define typed columns per document category with confidence that the extraction pipeline populates them consistently.
- Onboarding a new document type is additive. New schema, new thresholds, new classification label. Existing types are unaffected.

### Negative

- Each document type requires upfront schema definition. This is design work that must happen before extraction can run for that type. Document types with high format variability may require iteration to define a stable schema.
- Classification accuracy is a prerequisite for correct schema selection. A misclassified document will be extracted against the wrong schema, producing poor results. Classification quality must be monitored and classification errors must be catchable through confidence evaluation.
- The number of schemas grows with the number of supported document types. Schema versioning, threshold management, and regression testing scale linearly with the number of types.

### Constraints introduced

- Every supported document type must have a registered extraction schema and a corresponding set of confidence thresholds before it can be processed.
- Documents that cannot be classified against a known type must not proceed to extraction. They are recorded and routed for manual triage.
- Schema changes are versioned. A change to an existing schema must be evaluated for impact on confidence thresholds, downstream consumers, and staged output compatibility.
