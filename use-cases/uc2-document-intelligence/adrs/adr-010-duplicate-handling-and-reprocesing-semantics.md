# UC2-ADR-010: Duplicate Handling and Reprocessing Semantics

**Status**: Draft  
**Date**: 23/03/2026  
**Decision Scope**: How UC2 distinguishes duplicate ingestion from legitimate reprocessing, and how repeated processing of the same source document is represented through workflow state and governed staging.  
**Depends on**: UC2-ADR-004 (Worker as the Single Write Path), UC2-ADR-006 (Governed Staging Boundary), UC2-ADR-007 (Azure SQL as Workflow State Store), UC2-ADR-009 (Source-Aware Ingestion and Metadata Preservation)  
**Depended on by**: None

---

## Context

UC2 processes documents that may enter the pipeline more than once.

A document may be ingested twice accidentally because a connector re-reads content it has already processed, because the same document is uploaded through more than one path, or because an upstream retry republishes the ingestion event. A document may also be processed again intentionally because the extraction model has changed, the schema has changed, the source document has been updated, the confidence outcome requires remediation, or the business wants to rerun extraction after a control or policy change.

These cases are not the same.

An accidental duplicate should not create an unnecessary second operational result if the document and processing context are materially identical. An intentional reprocessing event must not be blocked simply because the document has been seen before. A changed version of the same source document must also be distinguishable from both a pure duplicate and a deliberate rerun of the same input.

Without an explicit decision, the pipeline has two failure modes. First, it may treat all repeated inputs as duplicates and suppress legitimate reprocessing. Second, it may treat all repeated inputs as new work and create conflicting downstream outputs for the same source document without a clear rule for which result is current.

The question is whether UC2 suppresses repeats at ingestion, always reprocesses and resolves later, or treats repeated processing as a first-class concept with explicit semantics in workflow state and governed staging.

---

## Decision Drivers

- UC2 must distinguish between accidental duplicate ingestion, source-document change, and intentional reprocessing.
- The Worker is the single write path for workflow state (UC2-ADR-004). Repeated processing must be represented explicitly in workflow state rather than left implicit in storage or messaging behaviour.
- Governed staging (UC2-ADR-006) must provide a clear current result for downstream consumers while preserving auditability of prior processing runs where required.
- Source-aware ingestion (UC2-ADR-009) preserves source reference and source metadata, which provides the basis for duplicate and reprocessing decisions but does not itself define the semantics.
- Reprocessing must remain possible for operational remediation, model improvement, schema evolution, and source-document changes.

---

## Considered Alternatives

### Option A: Reject all repeated documents at ingestion

The ingestion boundary determines whether a document has been seen before and suppresses any repeated input for the same source reference or content fingerprint.

Trade-off: Efficient and simple for accidental duplicates. But it prevents legitimate reprocessing unless the suppression state is manually overridden. It also couples duplicate semantics to the ingestion boundary, which cannot reliably distinguish all business cases. A source document may be the same physical file but require rerun because of a model or schema change. Blocking at ingestion is therefore too blunt.

### Option B: Always process every repeated input as new

Every ingestion event produces a new processing run regardless of whether the document has already been seen. Downstream consumers determine how to handle multiple results for the same source document.

Trade-off: Maximally flexible. Legitimate reprocessing is never blocked. But accidental duplicates create unnecessary cost, noisy workflow history, and conflicting staged outputs unless every downstream consumer implements its own resolution logic. This pushes an architectural concern outward and weakens control over the meaning of “current” versus “historical” results.

### Option C: Treat repeated processing as explicit workflow semantics (SELECTED)

UC2 distinguishes repeated inputs through workflow state. A repeated input may be classified as one of three cases: duplicate of an already processed input, new version of an existing source document, or intentional reprocessing of a previously seen input. The pipeline may suppress duplicate runs where no new value is created, while still allowing new versions and intentional reprocessing to create new processing runs. Governed staging maintains a clear current result while preserving processing lineage.

Trade-off: More state management and more explicit workflow logic. The pipeline must compare repeated inputs against prior state using the source reference, document identity, and other comparison signals defined outside this ADR. But it gives the architecture the control it needs: accidental duplicates can be suppressed, legitimate reruns remain possible, and downstream consumers receive a single current governed result with lineage preserved behind it.

---

## Decision

UC2 treats duplicate handling and reprocessing as explicit workflow semantics, not as a side effect of ingestion or storage behaviour.

A repeated input is not assumed to mean one thing. The architecture recognises three distinct cases:

- a duplicate input, where the same source document and materially equivalent processing context have already been processed
- a source-document change, where the source reference remains related but the input represents a new version or materially changed document
- an intentional reprocessing event, where the same input is processed again because the processing context has changed or a rerun has been requested

The exact comparison rules used to distinguish these cases are defined separately from this ADR. They may include source reference, source version information, document fingerprinting, processing version, rerun intent, or other comparison signals. This ADR does not prescribe the matching algorithm. It defines the required semantics.

Workflow state records repeated processing explicitly. A document identity may therefore have zero or more processing runs associated with it. Each run has its own status and lineage in workflow state. Duplicate suppression, where applied, is also recorded explicitly rather than inferred from absence of work.

Governed staging exposes a clear current result for downstream consumers. Where repeated processing creates a new valid governed result, that new result supersedes the prior current result. Prior runs remain part of processing history for audit and remediation purposes according to the retention and versioning rules defined elsewhere.

The architecture therefore allows legitimate reprocessing while preventing accidental duplicates from silently creating competing operational truths.

---

## Consequences

### Positive

- UC2 can distinguish accidental duplicate ingestion from legitimate reruns and source-document changes.
- Reprocessing remains possible for remediation, model changes, schema changes, source updates, and operational replay.
- Workflow state becomes the authoritative record of processing lineage. Repeated inputs are explicit states, not hidden behaviour.
- Governed staging presents a clear current result to downstream consumers while preserving lineage behind it.
- Downstream systems do not need to invent their own semantics for whether multiple results for the same source document are duplicates, versions, or reruns.

### Negative

- Duplicate handling is no longer a simple yes or no check. The architecture must support comparison logic and repeated-run lineage.
- A document identity can have multiple processing runs, which increases state-management complexity.
- The matching rules that distinguish duplicate input from changed input require careful design and validation. If they are weak, legitimate reruns may be suppressed or accidental duplicates may still slip through.
- Staging and downstream consumers require a clear notion of current versus historical result. That operational rule must be implemented consistently.

### Constraints introduced

- Repeated input must be represented explicitly in workflow state. It must not be left implicit in messaging behaviour, storage overwrite behaviour, or connector-specific logic.
- Duplicate suppression, where applied, must be an explicit outcome recorded by the pipeline rather than a silent drop.
- Governed staging must support a current-result view for downstream consumption even when multiple processing runs exist for the same document identity.
- The rules used to compare repeated input and determine its semantic meaning must be versionable and reviewable, because they affect operational truth.

