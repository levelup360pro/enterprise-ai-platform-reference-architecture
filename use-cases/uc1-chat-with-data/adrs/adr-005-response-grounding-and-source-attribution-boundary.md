# UC1-ADR-005: Response Grounding and Source Attribution Boundary

**Status**: Draft  
**Date**: 01/04/2026  
**Decision Scope**: How UC1 decides an answer is sufficiently grounded and what source-attribution standard the architecture enforces.  
**Depends on**: UC1-ADR-001, UC1-ADR-002, UC1-ADR-004  
**Depended on by**: UC1-ADR-007, implementation guidance for prompts and response handling

---

## Context

UC1 is not a free-form creative assistant. It is an enterprise conversational system intended to answer questions over governed data: contracts, customer records, policy documents, and structured extractions produced by UC2. When a user asks a question, the system retrieves relevant evidence from Azure AI Search and Fabric, then generates a natural-language response grounded in that evidence. The boundary between what was retrieved and what was generated is not merely a prompt-engineering concern; it is an architectural control that determines whether the system's outputs can be trusted, audited, and investigated after the fact.

As of April 2026, there is an asymmetry in citation support across delivery channels. Some retrieval paths can produce citation-backed answers, for example, Azure AI Search answer synthesis returns document references alongside generated text. However, other channel publication paths currently strip or fail to surface citations. Published Foundry agents delivered through Teams or Microsoft 365 Copilot may not render source references even when the retrieval plane supplies them. This means the architecture cannot rely on the delivery channel to enforce grounding visibility. The grounding standard must be defined independently of the channel's rendering capability, and source attribution must be preserved in the operational layer even when the end-user interface cannot display it.

Without an explicit grounding boundary, the system risks two failure modes. First, when evidence is absent or weak, a model with strong prior knowledge may synthesize a plausible-sounding answer that has no basis in the enterprise corpus, creating false confidence in unverified information. Second, when evidence exists but the channel strips citations, there is no way for reviewers, auditors, or the users themselves to trace an answer back to its supporting sources. Both failure modes undermine the governance claims that justify UC1's existence as an enterprise tool rather than a general-purpose chatbot.

---

## Decision Drivers

- UC1 should not answer from model prior knowledge when enterprise evidence is absent or contradictory.
- Users and reviewers need a path back to supporting sources for high-value responses.
- Some channel paths suppress citations even when the retrieval plane can produce them.
- The architecture must behave safely when evidence is weak, missing, or conflicting.

---

## Considered Alternatives

### Option A: Best-effort grounding only

Under this approach, the agent would use retrieved evidence when available but fall back to model prior knowledge when retrieval returns no results or low-relevance results. The system would not distinguish between evidence-backed answers and synthesized answers in either the user-facing response or the operational audit trail. Citation references would be included when the retrieval plane provides them and the channel supports rendering, but their absence would not change the system's behaviour or confidence level. This is the default behaviour of most LLM-based assistants and requires the least implementation effort.

Trade-off: Simplest to implement and produces fewer abstentions, but creates false confidence when evidence is weak and makes it impossible to distinguish governed answers from ungrounded synthesis during audit or investigation.

### Option B: Hard evidence boundary (SELECTED)

Under this approach, evidence sufficiency is treated as an architectural control boundary rather than a prompt-level preference. The system evaluates whether retrieved evidence is present and relevant before generating an evidence-backed answer. If the evidence package is weak, contradictory, or missing, the system abstains, narrows the answer to what can be supported, or explicitly states the limitation to the user. Source attribution is preserved in the evidence package and the operational audit trail regardless of whether the delivery channel can render citations. The grounding standard is defined by the architecture, not by the channel.

Trade-off: More abstentions and constrained answers in borderline scenarios, and requires explicit implementation of a grounding validation step between retrieval and generation. But ensures every evidence-backed answer can be traced to its sources and prevents silent synthesis beyond the evidence.

### Option C: Channel-dictated grounding standard

Under this approach, the grounding standard would vary depending on the delivery channel's capabilities. Channels that support citation rendering (such as a custom web UI) would enforce full source attribution. Channels that strip citations (such as published agents in Teams) would accept a lower grounding standard, allowing the system to produce answers without traceable source references. The rationale would be that enforcing a standard the user cannot see provides no user-facing value.

Trade-off: Adapts gracefully to channel limitations and avoids user-visible abstentions on constrained channels. But means that governance and auditability vary by delivery path, making it impossible to guarantee a consistent evidence standard across all access points. Investigators and auditors would face different levels of traceability depending on which channel was used.

---

## Decision

UC1 adopts a hard grounding boundary.

Retrieval evidence must exist and meet a relevance threshold before the assistant produces an evidence-backed answer. The system does not fall back to model prior knowledge as a silent substitute for missing enterprise evidence. When the evidence package is weak, missing, or contradictory, the assistant either abstains from answering, narrows the response to the subset that can be supported, or explicitly communicates the limitation to the user. The specific abstention and fallback rules will be defined during implementation, but the architectural principle is that no answer should be presented as evidence-backed unless the evidence actually exists.

Source attribution is preserved in the evidence package and the operational audit trail regardless of the delivery channel's citation-rendering capability. If a user asks a question through a channel that cannot display citations, the answer is still generated with the same grounding standard, and the source references are still recorded in the operational store. This ensures that auditors and investigators can trace any answer back to its sources even when the user-facing channel could not display them.

The grounding standard is owned by the architecture, not by the channel or the prompt. Prompting contributes to enforcement but is not sufficient on its own — a retrieval validation step is required between the retrieval phase and the generation phase to evaluate evidence sufficiency before the model is invoked.

---

## Consequences

### Positive

- Every evidence-backed answer can be traced to specific retrieved sources in the operational audit trail, regardless of the delivery channel used.
- The system does not produce plausible-sounding answers from model prior knowledge when enterprise evidence is absent, reducing the risk of false confidence in unverified information.
- Auditors and investigators can query the operational store (UC1-ADR-007) to verify the evidence basis for any past response, even if the user received the answer through a citation-stripped channel.
- The grounding standard is consistent across all delivery channels, simplifying governance reasoning and compliance documentation.

### Negative

- Users will see more abstentions and constrained answers in scenarios where evidence is borderline, which may reduce perceived helpfulness compared to an unconstrained assistant.
- A retrieval validation step must be implemented between retrieval and generation, adding latency and implementation complexity to the response pipeline.
- Defining the evidence-sufficiency threshold requires judgment and tuning: too strict and the system abstains unnecessarily, too lenient and the grounding boundary loses its protective value.

### Constraints introduced

- Prompting alone is not sufficient to enforce grounding; a retrieval validation step must exist in the response pipeline between the retrieval phase and the generation phase.
- Channel rendering limitations do not lower the evidence standard. The same grounding rules apply regardless of whether the channel can display citations.
- Operational logging must preserve source references for every evidence-backed response, even when the user-facing channel does not render them.
- The abstention and fallback behavior must be explicit and testable, not implicit in the prompt wording.
- Citation metadata must be either (a) extracted from Foundry API annotations and rendered in Copilot Studio, or (b) included in the Foundry agent's response body via system prompt instruction. Specify which approach is selected after the citation validation proof.

---

## Follow-ups

- Define concrete grounding and fallback rules during implementation and testing.
- Reassess channel rollout readiness when citation support improves on published-agent paths.
