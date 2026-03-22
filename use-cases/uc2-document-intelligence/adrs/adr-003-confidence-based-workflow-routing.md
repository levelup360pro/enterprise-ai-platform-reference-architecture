# UC2-ADR-003: Confidence-Based Workflow Routing

**Status**: Draft
**Date**: 13/03/2026
**Decision Scope**: How extraction quality determines workflow progression.
**Depends on**: UC2-ADR-001 (Layered Hexagonal Architecture)
**Depended on by**: UC2-ADR-007 (Azure SQL as the Workflow System of Record)

---

## Context

Content Understanding returns structured JSON with field-level confidence scores for every extracted value. These scores indicate how certain the model is about each extraction. A party name extracted from a clean, typed header will score differently from a jurisdiction inferred from a handwritten amendment on a 15-year-old scanned PDF.

UC2 cannot treat all extractions equally. High-confidence outputs should progress automatically. Low-confidence outputs need human review before they enter staging and become available to downstream consumers. Without an explicit routing mechanism, either every document requires manual review, which defeats the purpose of automated extraction, or every extraction is trusted regardless of quality, which puts ungoverned data into Fabric and AI Search.

The routing decision must be deterministic, auditable, and visible in the workflow state.

---

## Decision Drivers

- Extraction accuracy varies by document age, format, scan quality, and layout complexity. A single trust-or-reject model does not reflect this variation.
- Human review is expensive. It should be reserved for outputs where the model's confidence is genuinely low, not applied uniformly.
- Downstream consumers, including Fabric and Azure AI Search, must receive outputs that have either passed an automated quality gate or been explicitly approved by a reviewer. There is no third path.
- The workflow state in Azure SQL must record which path each document took and why, for audit and operational visibility.

---

## Considered Alternatives

### Option A: All outputs require human review

Every extraction is queued for manual review regardless of confidence. Reviewer approves or rejects before the document progresses.

Trade-off: Maximum oversight. But does not scale. At volume, the review queue becomes the bottleneck. Reviewers spend most of their time confirming outputs that the model already extracted with high confidence. Review fatigue degrades the quality of the reviews that actually matter.

### Option B: All outputs are trusted

Every extraction progresses to PII classification and staging without review. Confidence scores are logged but do not influence routing.

Trade-off: Maximum throughput. But ungoverned. A low-confidence extraction of a party name or a misclassified document type enters staging and propagates to Fabric and AI Search with no human checkpoint. The first time a downstream consumer acts on incorrect data, the pipeline's credibility is lost.

### Option C: Confidence-based routing with field-level thresholds (SELECTED)

Confidence evaluation examines field-level confidence scores against defined thresholds. If all required fields meet their thresholds, the document is marked AUTO_APPROVED and progresses to PII classification. If any required field falls below its threshold, the document is marked for review and routed to the human review queue. The reviewer sees the extraction with low-confidence fields flagged, corrects or approves, and the document resumes processing through the Worker.

Trade-off: Requires threshold definition and tuning per field and potentially per document type. Adds routing complexity. But reserves human effort for the documents that need it and lets high-confidence outputs flow without delay.

---

## Decision

UC2 uses confidence-based routing with field-level thresholds.

Confidence evaluation is a core domain capability (UC2-ADR-001). It receives the structured extraction output with field-level confidence scores and evaluates each required field against its defined threshold. The evaluation produces one of two outcomes: AUTO_APPROVED or REVIEW_REQUIRED.

**AUTO_APPROVED**: All required fields meet their confidence thresholds. The workflow proceeds to PII classification (UC2-ADR-002).

**REVIEW_REQUIRED**: One or more required fields fall below their confidence thresholds. The workflow records which fields triggered the flag and their scores. The document enters the human review queue. The reviewer sees the extraction with flagged fields highlighted and either approves (with or without corrections) or rejects. On approval, the workflow resumes and proceeds to PII classification. On rejection, the document exits the workflow.

Thresholds are configuration, not code. They are defined per field and can be scoped per document type. Initial thresholds are set conservatively (routing more documents to review) and tuned downward as extraction accuracy is validated over time.

Confidence evaluation does not call any external service. It operates on the extraction output already held by the orchestration layer. The orchestration layer persists the evaluation result as workflow state (UC2-ADR-007).

---

## Consequences

### Positive

- Human review effort is concentrated on documents where extraction quality is genuinely uncertain. High-confidence outputs progress without delay.
- Every document's routing decision is recorded in Azure SQL with the scores and thresholds that produced it. The decision is auditable and reproducible.
- Thresholds are tunable without code changes. As the team gains confidence in extraction accuracy for specific document types, thresholds can be adjusted to reduce unnecessary review volume.
- Downstream consumers only receive outputs that have passed either an automated quality gate or explicit human approval. There is no unvalidated path to staging.

### Negative

- Threshold definition requires judgment. Set too high, and the review queue fills with documents that did not need review. Set too low, and poor extractions reach staging. Initial calibration requires a representative sample of extraction outputs across document types.
- The human review loop adds latency for low-confidence documents. Documents in REVIEW_REQUIRED state do not progress until a reviewer acts. Reviewer availability and queue management become operational concerns.
- Field-level evaluation means confidence evaluation must understand the extraction schema for each document type. If a new document type is onboarded (UC2-ADR-008), its fields and thresholds must be registered before confidence evaluation works correctly for that type.

### Constraints introduced

- No extraction output may reach PII classification or staging without passing through the confidence evaluation.
- The confidence evaluation step produces two routing outcomes: AUTO_APPROVED or REVIEW_REQUIRED. Other workflow states, such as REVIEW_APPROVED, REJECTED, or FAILED, are handled outside the evaluation step.
- Thresholds must be defined for every required field in every supported document type schema. A required field without a defined threshold is treated as below threshold.
- Human review decisions must return through the Worker (UC2-ADR-004). The reviewer does not write to staging directly.
