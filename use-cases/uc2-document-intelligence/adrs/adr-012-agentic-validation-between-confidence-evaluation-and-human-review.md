# UC2-ADR-012: Agentic Validation Between Confidence Evaluation and Human Review

**Status**: Draft  
**Date**: 27/03/2026  
**Decision Scope**: How UC2 handles low-confidence or policy-selected extraction outputs before final routing, and whether an intermediate agentic validation step is used to enrich, resolve, or escalate those cases before human review where required.  
**Depends on**: UC2-ADR-004 (Worker as the Single Write Path), UC2-ADR-006 (Governed Staging Boundary), UC2-ADR-007 (Azure SQL as Workflow State Store), UC2-ADR-009 (Source-Aware Ingestion and Metadata Preservation), UC2-ADR-010 (Duplicate Handling and Reprocessing Semantics), UC2-ADR-011 (Failure Handling, Retry Budget, and Dead-Letter Semantics)  
**Depended on by**: None

---

## Context

UC2 is designed as a governed document intelligence workflow for operationally sensitive and regulated document processing. The current control path evaluates extraction confidence and routes outputs according to workflow policy. Some outputs can proceed directly. Some require human review. Some low-confidence or policy-selected cases may benefit from an intermediate validation step before final routing.

A direct review-only model is simple and defensible, but it makes the reviewer the only secondary reasoning layer. At scale, this creates review bottlenecks and gives the reviewer limited support beyond the original extraction result, source grounding, and confidence score.

In practice, not all low-confidence cases are equal. Some can be clarified through structured secondary analysis. A field may benefit from a second extraction attempt using a different prompt strategy. It may be validated against other extracted fields for consistency. It may be challenged against domain constraints and plausibility rules. For example, a lease duration of 600 months may be implausible for the document type, or a jurisdiction value may be syntactically valid text but semantically nonsensical in context.

There is therefore value in introducing an intermediate validation stage between routing evaluation and human review for cases where policy enables it. In this stage, multiple agentic validation paths can challenge or refine the original result and produce a revised confidence assessment, a recommendation, and a traceable reasoning record. This does not remove human oversight by default. It enriches the review path and may support selective automated acceptance only where policy explicitly allows it.

---

## Decision Drivers

- Reduce manual review burden without weakening governance.
- Improve reviewer context and speed for low-confidence or policy-selected outputs.
- Distinguish recoverable low-confidence cases from genuinely ambiguous or invalid outputs.
- Preserve explicit human accountability where policy requires it.
- Keep routing policy configurable by field, document type, and processing context.
- Ensure revised outcomes remain auditable and traceable.
- Avoid opaque "AI reviewed the AI" behaviour with no defensible reasoning trail.
- Support graph-based or workflow-based implementation without making the ADR depend on one product choice.

---

## Considered Alternatives

### Option A: Route policy-selected cases directly to human review

This is the simplest governed model. It is operationally clear, but it places the full analytical burden on the reviewer and does not distinguish between recoverable and irreducibly ambiguous cases before manual intervention.

### Option B: Add deterministic rule-based validation only

This improves control for obvious cases through range checks, schema validation, and fixed domain rules. It is useful but limited where validation requires contextual re-extraction, cross-field interpretation, or competing hypotheses.

### Option C: Add a single secondary model pass before review

This adds one more attempt to improve the result, but it provides weak challenge diversity and does not create a meaningful consensus pattern. A second pass can still fail in the same way as the first.

### Option D: Add an intermediate agentic validation step before final routing (SELECTED)

Multiple validation paths examine the result using different strategies, then produce a consensus-based recommendation and revised confidence assessment. This adds complexity, but it creates a stronger evidence path and a better basis for both review and selective policy-driven automation.

---

## Decision

Adopt **Option D**.

Low-confidence or policy-selected extraction results may pass through an agentic validation step before final routing where policy enables that path for the relevant field, document type, or processing context.

The purpose of this step is to perform structured secondary analysis on selected outputs using multiple validation approaches, which may include:

- re-extraction using alternative prompt strategies
- cross-field consistency checking
- validation against domain constraints and plausibility rules
- comparison of competing interpretations
- production of a consensus-based recommendation
- production of a revised confidence assessment
- capture of a traceable reasoning record for downstream review

The output of this step is advisory unless policy explicitly permits automatic acceptance for the relevant field and document type.

After agentic validation, routing remains policy-driven. A result may:

- proceed to human review with enriched context
- be rejected or escalated
- be auto-accepted if the revised confidence and consensus satisfy configured policy rules

Human review remains the decision point where policy requires explicit human approval.

This ADR defines the control pattern. It does not mandate a specific agent framework, orchestration substrate, or consensus algorithm.

---

## Consequences

### Positive

- Reduces manual review effort for low-confidence or policy-selected cases that can be clarified algorithmically.
- Improves reviewer speed and decision quality by presenting richer context.
- Produces a stronger control story than confidence thresholds alone.
- Distinguishes extraction weakness from document ambiguity or source-quality problems.
- Supports selective automation without making automation universal.
- Produces richer audit evidence for why a field was accepted, rejected, or escalated.

### Negative

- Adds workflow complexity and another control stage to operate.
- Increases runtime and cost for items entering the validation path.
- Requires governance over when revised confidence can influence routing or acceptance.
- Risks over-trust if agentic recommendations are treated as decision authority rather than decision support.
- Requires careful design of reasoning capture so it remains useful rather than noisy.

### Constraints Introduced

- Agentic validation is not equivalent to final approval unless policy explicitly allows automatic acceptance for that case.
- Routing rules must be configurable by field, document type, and processing context.
- The workflow must preserve the original extraction result, original confidence, and source grounding alongside any revised assessment.
- Agentic validation outputs must be auditable and traceable.
- Consensus logic must be explicit and versioned as part of workflow behaviour.
- Governed staging must retain the final disposition and whether the result was human-approved, auto-accepted by policy, rejected, or escalated.
- This ADR does not prescribe a specific implementation technology, framework, or prompt structure.

