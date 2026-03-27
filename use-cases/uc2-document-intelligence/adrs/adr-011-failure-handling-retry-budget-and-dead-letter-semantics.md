# UC2-ADR-011: Failure Handling, Retry Budget, and Dead-Letter Semantics

**Status**: Draft  
**Date**: 24/03/2026  
**Decision Scope**: How UC2 handles processing failures, how retries are bounded, and how non-recoverable work is represented for operational recovery without contaminating governed staging.  
**Depends on**: UC2-ADR-004 (Worker as the Single Write Path), UC2-ADR-006 (Governed Staging Boundary), UC2-ADR-007 (Azure SQL as Workflow State Store), UC2-ADR-010 (Duplicate Handling and Reprocessing Semantics)  
**Depended on by**: None

---

## Context

UC2 processes documents through a multi-stage workflow that includes ingestion, classification, extraction, confidence evaluation, review routing, PII tagging, and governed staging. Failures can occur at any stage.

Some failures are transient. A dependent service may be temporarily unavailable. A network call may time out. A lock may expire. A downstream platform may reject a request because of temporary throttling. These failures may succeed on retry without any change to the document or workflow.

Some failures are persistent. A document may be corrupt. A file format may be unsupported. A required dependency may be misconfigured. A schema may be invalid for the document being processed. A source input may violate a contract assumption. These failures are unlikely to succeed through blind retry.

Without an explicit failure model, the pipeline has two failure modes of its own. First, it may retry indefinitely, consuming cost and hiding the fact that work has become operationally stuck. Second, it may fail fast and lose the distinction between transient and non-recoverable conditions, making recovery ad hoc and non-repeatable.

UC2 also has a governed staging boundary. A failed or uncertain processing run must not leak partial or untrusted output into staging. If failure handling is weak, governed staging stops being a governed boundary and becomes merely a storage location for mixed-quality results.

The question is how UC2 bounds retries, when it stops retrying, how it represents failed work, and how non-recoverable processing is surfaced for operational recovery without polluting governed outputs.

---

## Decision Drivers

- UC2 must distinguish transient failures from persistent failures and handle them differently.
- The Worker is the single write path for workflow state (UC2-ADR-004). Failure state and retry state must be explicit in workflow state.
- Governed staging (UC2-ADR-006) must contain only outputs that have completed the required workflow path. Failed or incomplete runs must not appear there.
- Operational recovery must be possible without losing lineage. A failed run must remain visible and reprocessable.
- Retry behaviour must be bounded. Unlimited retry hides failure, increases cost, and creates unpredictable backlog behaviour.

---

## Considered Alternatives

### Option A: Unbounded automatic retry

Any failed processing attempt is retried until it eventually succeeds.

Trade-off: Simple in principle. Transient issues may clear without operator intervention. But persistent failures never become explicit operational states. The system wastes resources retrying work that will not recover on its own. Backlogs become harder to reason about. There is no clean point at which a human is told that intervention is required.

### Option B: Fail fast with no retry

Any processing failure is treated as terminal. The workflow records failure and requires manual intervention for every failed document.

Trade-off: Operationally simple and explicit. But it treats transient faults as if they were persistent. Temporary outages, throttling, or short-lived dependency failures would all require manual recovery, increasing operational overhead and reducing resilience.

### Option C: Bounded retry with explicit terminal failure state (SELECTED)

UC2 retries failures that are considered recoverable, but only up to a defined retry budget. If the work still cannot complete, the run is moved to an explicit terminal failure state and surfaced for operational recovery. Failed runs remain outside governed staging. Recovery happens through explicit reprocessing or operator action rather than indefinite background retry.

Trade-off: More workflow state and operational design is required. Retry budget policy must be defined and maintained. But the system becomes predictable. Transient faults get a chance to recover automatically; persistent faults become visible, bounded, and recoverable in a controlled way.

---

## Decision

UC2 uses bounded retry with explicit terminal failure state.

A processing failure is not treated as automatically final, but neither is it retried indefinitely. Failures judged recoverable within the workflow are retried according to a defined retry budget. The retry budget is policy, not hardcoded by this ADR. It is defined separately and may vary by processing stage or dependency type.

If a processing run exhausts its retry budget without completing successfully, that run transitions to an explicit terminal failure state. Terminal failure means the workflow has stopped automatic recovery for that run. It does not mean the document is unrecoverable forever. It means further action requires explicit operational intervention, replay, or reprocessing.

Failure state is recorded in workflow state together with enough diagnostic context to support investigation and recovery. The exact diagnostic fields are defined separately from this ADR.

Where the messaging or orchestration substrate supports dead-letter semantics, terminally failed work is surfaced through that mechanism or an equivalent explicit operational holding state. The architecture requires a visible and queryable place for failed work to accumulate once automatic retry is exhausted. This ADR does not prescribe the exact platform implementation of that holding state. It prescribes the semantics.

Partial or failed processing results must not be written to governed staging. Governed staging receives outputs only from runs that have completed the required workflow path successfully. Failed runs, retrying runs, and incomplete runs remain outside the staging boundary.

Recovery from terminal failure is explicit. A failed run may be replayed, rerun, or superseded by a new processing run according to the reprocessing semantics defined elsewhere. The original failed run remains part of workflow history.

---

## Consequences

### Positive

- Transient failures can recover automatically without immediate operator intervention.
- Persistent failures become explicit and bounded rather than consuming indefinite retry.
- Workflow state becomes the authoritative record of retry count, terminal failure, and recovery lineage.
- Governed staging remains clean. Failed, partial, and uncertain outputs do not cross the staging boundary.
- Operational teams have a clear recovery model: investigate, correct, replay, or rerun.

### Negative

- Retry policy must be designed and maintained. Poor retry budgets can either over-retry persistent failures or under-retry transient ones.
- Terminal failure handling introduces operational overhead. Someone must own investigation and replay.
- Some failures are hard to classify cleanly as transient or persistent. Retry policy will need iteration based on observed production behaviour.
- Failure visibility requires dashboarding, alerting, or equivalent operational tooling outside the scope of this ADR.

### Constraints introduced

- Retry behaviour must be bounded. Unlimited retry is not permitted.
- Failure state and retry exhaustion must be recorded explicitly in workflow state.
- Failed, partial, or incomplete runs must not write governed outputs to staging.
- Terminally failed work must be surfaced through an explicit operational holding mechanism rather than disappearing into logs or silent retry loops.
- Recovery after terminal failure must occur through explicit replay, rerun, or operator action, not by hidden background continuation.

