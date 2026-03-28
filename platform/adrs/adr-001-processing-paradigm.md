# ADR-001: Processing Paradigm

**Status**: Accepted  
**Date**: 09/03/2026  
**Decision Scope**: Whether to use traditional deterministic software, fully autonomous multi-agent systems, or a hybrid approach as the fundamental processing paradigm for the enterprise AI platform  
**Depended on by**: ADR-002 (Runtime Platform and EU Region Selection), all use-case ADRs

---
## Platform Scaling Boundary

This architecture is scoped as a single-workload AI landing zone. One subscription, one set of AI services, one processing pipeline, one agent runtime, one set of consumers. Governance, identity, networking, and observability are designed for this scope.

The architecture does not include an AI Gateway (Azure API Management as AI Gateway). The current scope does not present the multi-consumer governance problems that an AI Gateway addresses: cross-application token quota management, centralised tool discovery and policy enforcement across multiple consumers, multi-team rate limiting, portfolio-wide prompt and completion auditing, or load balancing across multiple model deployments serving independent applications.

APIM AI Gateway becomes justified when the platform scales beyond a single workload. The triggers are: multiple AI applications sharing model infrastructure where independent token budgets are required; shared tool endpoints that need centralised discovery, authentication mediation, and policy enforcement across organisational boundaries; regulatory or cost-attribution requirements that demand per-consumer token metering at the gateway layer rather than the application layer; or multiple teams deploying agents that require consistent content safety policy enforcement without embedding it in each application.

The architecture is designed so that adopting an AI Gateway is additive, not destructive. The application layer accesses Foundry models through an application abstraction with a configurable endpoint rather than embedding gateway assumptions into workflow logic. If an AI Gateway is introduced later, the expected changes are primarily in endpoint configuration, identity assignment, network routing, and observability policy rather than in the processing paradigm defined by this ADR. The detailed identity, network, and telemetry implications of that change are defined in downstream ADRs rather than in this ADR.

---

## Context

This platform processes large volumes of complex documents (e.g., contracts, leases, amendments, side letters, audit evidence) across four capabilities: document intelligence, conversational analytics, contract intelligence, and audit automation. The target industries are EU-regulated, document-intensive sectors where organisations manage decades of unstructured data across fragmented systems, face regulatory compliance obligations (GDPR, phased EU AI Act obligations with most Annex III high-risk system requirements applying from 2 August 2026), and need AI systems that are governed, auditable, and human-supervised.

Before choosing a compute substrate, a region, or an SDK, there is a more fundamental question: **what kind of system are we building?**

The answer determines the technology stack, the governance model, the cost structure, the auditability story, and the risk profile. Every subsequent ADR flows from this choice.

### What the use cases actually require

**Document Intelligence** processes 20+ years of documents (e.g., contracts) scattered across multiple systems. Each document must be classified by type (lease, amendment, side letter, NDA, termination notice, up to 200 categories), extracted (parties, dates, obligations, jurisdictions, terms, etc...), validated (confidence thresholds, cross-reference verification against master data), resolved to a client entity, and persisted (structured output to Fabric Lakehouse and AI Search index). New documents arrive continuously. The classification and extraction steps require AI services that can handle format variation across over two decades of documents, infer values not explicitly written (e.g., end dates from start dates and durations, jurisdiction from party addresses), and provide grounded extraction with confidence estimates for extracted fields where enabled and supported by the analyzer configuration.

**Conversational Analytics** requires natural language queries across structured and unstructured data, with grounded answers and source citations. The intelligence is in the retrieval and reasoning, not in a fixed query path. Users ask questions that cannot be anticipated at design time. The system must decompose complex queries, retrieve from multiple sources, synthesise results, and cite evidence.

**Contract Intelligence** compares sent and returned contract versions. The system must extract both documents, identify structural differences deterministically (field-level diff on dates, amounts, party names), and then assess the semantic significance of changes (whether a clause modification increases liability exposure, whether unexpected language was added). The structural diff is a solved, deterministic problem. The semantic assessment is not.

**Audit Automation** mimics what AI-powered external auditors do: systematic coverage of compliance areas, gap identification against policies and regulatory requirements, structured findings with severity and remediation. This requires reasoning over a knowledge base of policies, prior findings, and regulations. The audit questions are not templated. The system must interpret ambiguous compliance language and apply it to specific operational contexts.

### The core tension

Some parts of these use cases are well-defined, repeatable, and must be 100% consistent: document routing, confidence threshold enforcement, stage sequencing, field-level comparison, persistence. These are deterministic problems with known solution paths.

Other parts are ambiguous, require multi-step reasoning, and vary per document: classifying a contract that does not match any standard template, inferring obligations from clause language, assessing the business significance of a contract change, interpreting a regulation against a specific operational context. These are problems where the path to the answer depends on the content.

A deterministic pipeline can call AI services without involving an agent. A function sends a request, receives a response, and deterministic code decides what to do with the result. That is a service call, not an agent decision. The distinction matters: an agent decides how to act on results, potentially choosing different paths based on intermediate findings. A service call within a deterministic pipeline does not. Most of this platform's processing falls into the service-call category. Agents earn their place only where the reasoning path cannot be defined in advance.

The architecture must accommodate deterministic logic, AI service calls, and agentic reasoning within a single unified processing model.

---

## Decision Drivers

- **Auditability in regulated environments**: Every AI output must have a confidence score where supported by the service, a source citation, and a traceable decision path. An auditor must be able to reconstruct why the system produced a specific output. "The agent decided" is not a control story.
- **Human oversight (EU AI Act alignment)**: For this platform, documents influencing legal or financial decisions are treated as requiring bounded autonomy and defined human oversight. Human-in-the-loop gates at defined points. The system flags. Humans decide.
- **Document variation**: 20+ years of documents across multiple jurisdictions, formats, and document management systems. Rule-based extraction breaks on format variation. Template matching breaks on negotiated contracts. The extraction and classification layer must handle ambiguity.
- **Semantic reasoning**: Contract comparison, audit gap identification, and cross-document validation require understanding meaning, not just matching patterns. Deterministic diff tells you a clause changed. It does not tell you why that change matters.
- **Strict stage ordering**: Documents must flow through classification, extraction, validation, entity resolution, and persistence in sequence. Quality gates must halt processing if confidence drops below threshold. Failed stages must retry without re-executing completed stages.
- **Cost efficiency**: LLM calls are expensive at scale. A batch of 500 documents generating 20,000+ API calls must not include redundant LLM invocations. If a decision can be made deterministically, it should be.
- **Governance as infrastructure**: Controls (access boundaries, logging, evaluation, human oversight, incident path, rollback) must be embedded in the processing model, not retrofitted as documentation.
- **Migration path**: The processing model must support future migration to managed compute (Azure AI Agent Service, Hosted Agents) without rewriting pipeline logic. Building on a framework that reduces hosting lock-in protects the investment as the platform matures.

## Considered Alternatives

### Option A: Fully Deterministic (Traditional ETL / Rule-Based Processing)

Traditional document processing pipeline: rule-based classification, template-based extraction, regex and lookup-based validation, deterministic entity resolution, direct persistence. No LLMs. No agents.

**What it handles well**: Stage sequencing (built-in), field-level comparison (exact match), persistence workflows, confidence threshold enforcement (if the extraction service provides scores), cost predictability (no LLM inference costs beyond AI service calls), full auditability (every step is deterministic and reproducible).

**Where it fails**:

Document classification across 200+ types with 20+ years of format variation. Rule-based classifiers require exhaustive rule definition per type. When a document does not match any rule, it falls through. Maintaining hundreds of classification rules across decades of format drift is operationally unsustainable.

Extraction of inferred values. Document Intelligence, used in a constrained extractive pattern, can only extract what is explicitly written. It cannot calculate an end date from a start date and duration. It cannot determine jurisdiction from party addresses. It cannot infer total obligations from individual line items. Content Understanding's generative extraction can. That is an AI capability, not a rule.

Semantic assessment. A deterministic diff tells you that Section 4.2 changed. It does not tell you that the change introduces unlimited liability. Contract comparison at the semantic level requires language understanding.

Cross-document reasoning. When an amendment references "Contract X dated Y" and the system must locate Contract X, validate the reference, and reconcile terms across both documents, the problem has moved beyond pattern matching into reasoning.

Ad-hoc querying. Conversational analytics and audit queries are open-ended by nature. Users ask questions that were not anticipated at design time. A deterministic system can only answer questions it was programmed to handle.

**Assessment**: Deterministic processing handles approximately 40-50% of the pipeline requirements well (stage control, persistence, field-level diff, threshold enforcement). It fails on the capabilities that differentiate AI-powered document processing from traditional document management: handling variation, inferring meaning, reasoning across documents, and answering novel questions. Choosing this approach means accepting that the most valuable capabilities are out of scope.

### Option B: Fully Autonomous Multi-Agent (Agents Control Everything)

Autonomous agents manage the entire pipeline. Agents decide their own routing, tool selection, stage sequencing, and quality assessment. The LLM plans and executes each step. No deterministic orchestration layer.

**What it handles well**: Ambiguous classification (agent reasons about document type), inferred extraction (agent uses Content Understanding with generative fields), semantic assessment (agent interprets changes in context), cross-document reasoning (agent queries and synthesises across documents), novel queries (agent decomposes and answers open-ended questions).

**Where it fails**:

Stage ordering is non-deterministic. If the agent decides to skip validation or reorder stages, the pipeline has no enforcement mechanism. In a regulated environment, "the agent chose to skip the quality gate" is an audit finding, not a feature.

Reproducibility. Same input can produce different outputs across runs. For document extraction feeding a single view of customer consumed by other systems, output consistency is a hard requirement. Downstream systems (CRM, finance, BI) depend on the extracted data being stable.

Cost. Every routing decision becomes an LLM call. If the agent reasons about which extraction model to use for each of 500 documents in a batch, that is 500 additional LLM calls that a deterministic router handles in milliseconds with a lookup table. At scale, this makes the pipeline economically unviable.

Auditability. Agent reasoning is probabilistic. Reconstructing why the agent classified a document as "side letter" rather than "amendment" requires inspecting the full prompt, context window, and model state. This is possible with logging infrastructure, but it is inherently more complex and less trustworthy than a deterministic decision tree.

Human oversight enforcement. The Agent Framework and Foundry Agent Service both provide infrastructure for human-in-the-loop: `wait_for_external_event` with durable timers, persistent threads, and checkpointing that survives scale-to-zero. The infrastructure capability exists. The problem is governance: in a fully autonomous model, the agent decides when to invoke a quality gate. If the agent reasons that confidence is sufficient and skips the human review step, the infrastructure does not prevent it. The enforcement mechanism is the agent's own judgment, which is probabilistic. In a regulated environment, the decision to pause for human review must be an orchestration-level enforcement, not an agent-level choice.

Failure recovery. The Agent Framework provides checkpointing and replay at the workflow layer whether the workload runs in Container Apps today or a managed agent runtime later. Persisting workflow state in the shared SQL store allows the API and worker services to resume from the last committed step rather than re-running an entire document batch. The infrastructure handles failure recovery. The issue for fully autonomous agents is reasoning reproducibility: if a checkpoint saves state mid-pipeline and the agent replays from that checkpoint, the next reasoning step may produce a different result because agent reasoning is probabilistic. The checkpoint preserves where the pipeline was. It does not guarantee that the agent will make the same decision it would have made before the failure. For a pipeline feeding a relationship-oriented analytics view consumed by downstream systems, this non-determinism in replay is a governance concern.

**Assessment**: Fully autonomous agents handle the ambiguous, reasoning-heavy parts of the pipeline well. The infrastructure (Agent Framework, Container Apps, and future managed agent services) provides checkpointing, replay, state persistence, and HITL capability. The failure is not infrastructure. It is governance. In a fully autonomous model, the agent controls stage ordering, quality gate invocation, and routing decisions. The infrastructure can support these controls, but cannot enforce them if the agent chooses not to use them. In a regulated environment, quality gates, human review thresholds, and stage ordering must be enforced by the orchestration layer, not left to agent judgment. An autonomous agent system where governance depends on the agent's own reasoning is not suitable for production in regulated industries.

### Option C: Unified Workflow with Typed Executors (SELECTED)

The Microsoft Agent Framework provides two primitives: **ChatAgent** (LLM-driven, conversational, dynamic reasoning) and **Workflow** (a directed graph of **Executors** connected by **Edges**, with the workflow engine controlling sequencing, conditional routing, parallelism, and checkpointing). An Executor is an individual processing node. It receives typed input, produces typed output, and the workflow engine enforces what happens next. The critical design property: an Executor does not require an LLM. It can contain pure deterministic code, a single AI service call, or a full ChatAgent. The workflow engine treats all three identically from an orchestration perspective.

This means the document processing pipelines are built as Agent Framework Workflows. Every stage is an Executor. The Executor type determines the intelligence at each stage. The Workflow engine controls all of it.

**Three Executor patterns** (these map to the SDK's execution model but are named here for architectural clarity; the framework provides Executors as a general-purpose primitive):

**Deterministic Executor**: Pure code. No AI service calls. No LLM. Handles ingestion triggers, confidence threshold validation, conditional routing, data transformation, persistence to Fabric Lakehouse, CRM sync, field-level diff, and quality gate enforcement. Input is predictable. Output is deterministic. These stages do not need intelligence; they need reliability.

**AI Service Executor**: Deterministic code that makes a single call to an AI service (Content Understanding, Document Intelligence, Microsoft Foundry models) and processes the response. The Executor sends a request with a defined schema, receives structured output (JSON with confidence scores where supported), and deterministic code decides what to do with it. The AI services themselves use probabilistic models internally, so extraction results may vary slightly across runs for identical inputs. From the orchestration layer's perspective, however, the behaviour is functionally bounded: same document in, same schema out, with confidence scores indicating extraction quality. The Executor does not reason about the response or choose alternative paths based on intermediate findings. It parses, validates, and routes according to code. That is the operational distinction from an Agent Executor, which does reason about intermediate results and may take different paths across runs. The AI service provides the intelligence. The Executor controls the interaction. No autonomous decision-making about what to do next. Classification via Content Understanding analysers with schema-defined classification fields, extraction via zero-shot schema, and single-call semantic summarisation (e.g., submitting a prompt asking "what is the business significance of these contract changes?" and processing the structured response) all fall into this category.

**Agent Executor**: A ChatAgent embedded within a Workflow node. The agent reasons dynamically: it decomposes a problem, decides which tools to call, processes intermediate results, and determines the next step based on what it finds. The reasoning path is not defined in advance. This is reserved for stages where the processing path depends on intermediate results and cannot be pre-defined in code. Complex audit queries that require multi-step reasoning across policies, regulations, and operational data. Conversational analytics where query decomposition, multi-source retrieval, and cross-document synthesis require dynamic decision-making. Ambiguous entity resolution where fuzzy matching across jurisdictions and naming conventions requires contextual reasoning that lookup tables cannot handle.

The Workflow provides:

**Type-safe messaging** between Executors, enforcing each stage's output schema at the boundary. A classification Executor outputs a typed document classification result. The next Executor's input type must match. Type-safe message handling moves many integration errors to the boundary of the stage rather than letting them leak downstream.

**Conditional Edges** for routing based on Executor output. Low confidence routes to human review. High confidence routes to persistence. Document type routes to the appropriate extraction schema. These are orchestration decisions, not agent decisions.

**Checkpointing and durability** so long-running pipelines survive failures and restarts. Durable execution resumes from persisted workflow state, reducing unnecessary repetition of successfully completed work and preserving orchestration progress across failures. A human-in-the-loop approval that takes hours does not lose pipeline state.

**External events** for human-in-the-loop gates. The Workflow pauses at a quality gate, raises an external event, waits for human approval with a durable timer for SLA enforcement, and resumes on approval. Provided approval and routing logic are implemented as workflow transitions rather than agent prompts, the agent cannot bypass these controls from within its bounded stage.

**Lifecycle events** for observability. Executor start, complete, and error events feed OpenTelemetry tracing. Executor lifecycle, routing behaviour, and workflow execution are observable through the framework's telemetry model. Raw inputs, outputs, and service-call details can be included where telemetry configuration and data-protection policy allow.

**Illustrative examples of Executor pattern selection:**

A stage that triggers on a blob event, validates file format, and routes to the next stage is a Deterministic Executor. No AI involved.

A stage that calls Content Understanding with a schema and parses the structured JSON response is an AI Service Executor. The service provides the intelligence. The Executor controls the interaction. No autonomous reasoning.

A stage that compares confidence scores against thresholds and routes low-confidence documents to human review is a Deterministic Executor. Code, not intelligence.

A stage that decomposes a complex audit query across policies, regulations, and operational data, retrieves from multiple sources based on intermediate findings, and synthesises structured results is an Agent Executor. The reasoning path depends on intermediate results and cannot be pre-defined.

The specific stages, their boundaries, and their Executor pattern assignments are defined per use case in the use-case ADRs. This ADR establishes the selection criteria: if the processing path is known in advance, use a Deterministic Executor. If intelligence is needed but the path is still defined in code, use an AI Service Executor. If the path depends on intermediate results and cannot be pre-defined, use an Agent Executor.

**Why build everything as Agent Framework Executors, even the deterministic stages?**

Lower migration friction. The Agent Framework SDK reduces hosting lock-in and should lower migration effort across future runtimes. Today, Workflows run on the compute substrate selected in ADR-002. As Foundry runtimes mature, especially once the required private-networking and production controls exist for the chosen deployment model, workflow logic may be moved from the current compute substrate to managed hosting with reduced rewrite effort. Deployment, identity, networking, and packaging changes are still likely, but the Executor logic and Workflow graph should transfer without rewriting. Deterministic code can run on any of these platforms regardless of framework choice; Hosted Agents accepts custom containerised code. But building deterministic stages outside the Agent Framework means maintaining two different patterns: framework Executors for agent stages, raw code for deterministic stages, with separate observability, separate checkpointing behaviour, and separate debugging approaches. Building all stages as Agent Framework Executors provides one pattern for building, one for observing, one for debugging.

Unified observability. All Executors (deterministic, AI service, agent) emit the same lifecycle events to the same OpenTelemetry pipeline. One monitoring surface. One tracing model. One set of alerts. No stitching together traces from two different systems.

Consistent boundary enforcement. Type-safe messaging and conditional edges apply uniformly across all Executor patterns. A deterministic Executor and an Agent Executor follow the same routing rules. The Workflow engine does not distinguish between them for orchestration purposes.

**Where agents operate under constraint**:

Agents do not control the pipeline. They execute within stages defined by the Workflow. Provided routing and approval logic are implemented as workflow transitions, an agent cannot skip a quality gate, reorder stages, or bypass human review from within its bounded stage. Within its bounded stage, an agent acts: it decides which tools to call, processes intermediate results, and determines the next step based on what it finds. That is why it is an agent and not a single service call. But its authority ends at the stage boundary. The Workflow enforces what happens next. This is the line of autonomy.

---

## Decision

**Unified Agent Framework Workflow with typed Executors.** All processing pipelines are built as Agent Framework Workflows. Every stage is an Executor. Deterministic Executors handle pipeline control, validation, persistence, and structural comparison. AI Service Executors make single calls to Content Understanding, Document Intelligence, or Azure OpenAI where intelligence is needed but the processing path is defined in code. Agent Executors (ChatAgent) handle the narrow cases where multi-step reasoning is required and the path depends on intermediate results. The Workflow engine controls sequencing, conditional routing, quality gates, human-in-the-loop checkpoints, retry policies, and checkpointing across all Executor patterns uniformly. The number of Workflows, their boundaries, and their triggers are use-case-level decisions documented in the use-case ADRs.

This is not "agents vs. pipelines." It is Workflows with three patterns of Executor. Deterministic where the path is known. AI service calls where intelligence is needed but the path is still defined. Agents where the path depends on intermediate results. The Workflow engine controls all of it.

---

## Consequences

### Positive

- **Auditable end-to-end**: Stage transitions are deterministic and logged by the Workflow engine. AI service calls within Executors are traced via OpenTelemetry. Agent reasoning within Agent Executors is observable through the same tracing pipeline. The design supports end-to-end trace correlation per document across the pipeline: which Executor, which pattern (deterministic / AI service / agent), which model (if applicable), which confidence score, which human approved.
- **Human oversight enforceable**: Quality gates are Workflow constructs (external events + durable timers), not agent decisions. Provided routing and approval are implemented as workflow transitions, agents cannot bypass what the Workflow engine controls.
- **Format variation handled**: AI-powered classification and extraction (via AI Service Executors calling Content Understanding) handle 20+ years of document variation that rule-based systems cannot. Zero-shot extraction and inferred fields reduce the need for hundreds of custom rules.
- **Semantic reasoning available where justified**: Contract comparison, audit queries, and conversational analytics benefit from AI service calls (single-call semantic summary) and agent reasoning (multi-step complex queries). The architecture is explicit about which stages use which Executor pattern and why.
- **Cost-controlled**: The majority of pipeline stages (ingestion, validation, persistence, structural comparison, routing) invoke no LLM. AI service calls are used only where intelligence is needed. Agentic reasoning is reserved for the narrow cases where the processing path depends on intermediate results. No redundant LLM calls for routing, sequencing, or persistence decisions. The intended cost profile is predominantly deterministic, with targeted AI spend concentrated in extraction, summarisation, retrieval, and bounded reasoning stages.
- **Reproducible persistence**: The output that feeds downstream systems (Fabric Lakehouse, AI Search index, CRM sync) comes from a Deterministic Executor. Extraction may vary slightly across runs (probabilistic AI service); persistence logic (schema mapping, upsert, deduplication) is deterministic.
- **EU AI Act alignment**: The architecture is designed to support key EU AI Act control expectations, including human oversight (Article 14, via quality gates as Workflow external events), traceability (Article 12, via unified OpenTelemetry across all Executor patterns), and operational resilience (Article 15, via Workflow retry policies and checkpointing). Formal compliance depends on use-case risk classification, governance documentation, and legal review.
- **Lower migration friction**: All pipelines are built as Agent Framework Workflows with typed Executors and Edges. The deliberate architectural decision to isolate business logic in Executors and keep the Workflow engine as a thin orchestration layer means that Executor logic and the Workflow graph are decoupled from the hosting substrate. When the target runtime changes (Container Apps today, potentially Hosted Agents or managed Foundry compute in the future), deployment configuration, identity bindings, networking, and packaging will need to change, but the Executor implementations and Workflow definitions should transfer without rewriting. This is a materially better migration posture than embedding logic directly in queue consumers, background jobs, or raw framework endpoints where business logic, orchestration, and hosting are coupled. Building deterministic stages outside the Agent Framework would mean maintaining two patterns (framework Executors for agent stages, raw code for deterministic stages) with separate observability, checkpointing, and debugging, and both would need independent migration paths. Building everything as Executors keeps one pattern, one migration path, and one set of interfaces to adapt. ADR-002 selects the specific compute substrate; this ADR ensures the processing paradigm does not lock to it.
- **Unified observability**: All Executor patterns emit the same lifecycle events. One OpenTelemetry pipeline. One monitoring surface. No stitching traces across different orchestration systems.

### Negative

- **Agent Framework SDK dependency**: All pipelines depend on the Agent Framework SDK, including stages that are pure deterministic code. If the SDK introduces breaking changes, even deterministic stages are affected. Mitigation: pin SDK version, isolate Executor logic behind interfaces and adapters following the same architectural pattern already proven in the platform prototype ([hexagonal architecture reference](https://github.com/levelup360pro/agentic-ai-examples/blob/main/marketing-team/reports/WEEK8.md)), upgrade in controlled cycles with dev validation before production.
- **Agent behaviour requires evaluation**: Agent Executors are probabilistic. The same complex audit query may produce different reasoning paths across runs. This requires evaluation pipelines, quality scoring, and drift detection for the Agent Executor stages. Deterministic and AI Service Executors do not have this problem because their processing paths are defined in code.
- **Governance infrastructure required from day one**: Quality gates, human review queues, confidence routing, audit trails, and HITL workflows must be built into the Workflow before agents process production data. This is front-loaded work. Skipping it and "retrofitting later" breaks the paradigm.
- **Clear boundary definition required per use case**: Each use case must define which Executor pattern is used at each stage and why. If the boundary is not explicit, the architecture drifts toward over-agenting (using Agent Executors where AI Service Executors suffice) or under-agenting (using deterministic code where AI service calls are needed). The line of autonomy must be documented per stage in each use-case ADR.
- **Over-engineering risk for simple stages**: Wrapping a threshold comparison or a file format check in an Agent Framework Executor adds framework overhead compared to a raw function call. The trade-off is justified by the migration path and unified observability, but it is overhead nonetheless.

---

## Risks and Mitigations

- **Agent quality drift**: AI model outputs can degrade over time as models are updated or data distributions shift. This affects AI Service Executors and Agent Executors equally. Mitigation: continuous evaluation pipeline monitoring extraction quality, classification accuracy, and reasoning relevance. Confidence thresholds in the Workflow catch degradation automatically by routing more documents to human review when scores drop.
- **Executor pattern boundary confusion**: Teams may use Agent Executors where AI Service Executors suffice (over-agenting), or deterministic code where AI service calls are needed (under-agenting). Mitigation: this ADR defines the selection criteria for each Executor pattern. Each use-case ADR applies those criteria to define the Executor pattern per stage with rationale. Code review checks that the Executor pattern matches the documented boundary. The test: if the processing path is defined in code (even if it includes an AI service call), it is a Deterministic or AI Service Executor, not an Agent Executor. If the processing path depends on intermediate results and cannot be pre-defined, it is an Agent Executor.
- **SDK breaking changes**: The Agent Framework SDK remains prerelease / Release Candidate at the time of this decision and must be version-pinned. Extension packages have introduced breaking changes between RC versions. Mitigation: pin to exact version, isolate Executor business logic behind protocol interfaces (hexagonal architecture), upgrade in controlled cycles. ADR-002 documents the specific version pinning strategy and upgrade process.
- **Over-engineering deterministic stages**: Wrapping simple deterministic logic in Agent Framework Executors adds overhead. Mitigation: Deterministic Executors should be thin. The business logic lives in service classes behind interfaces; the Executor is a lightweight wrapper that receives input, calls the service, and returns output. If the framework adds unacceptable latency to deterministic stages during the cold-start spike (ADR-002 Follow-up), re-evaluate whether the migration path justification still holds against the performance cost.

---

## Decision Matrix

|Requirement|Deterministic Only|Autonomous Agents|Unified Workflow (Selected)|
|---|---|---|---|
|Stage ordering enforcement|Yes|No|Yes (Workflow engine)|
|Quality gate enforcement|Yes|No|Yes (Workflow external events)|
|Human-in-the-loop enforcement|Yes (external events)|Possible, but weak if agent-controlled|Yes, orchestration-enforced|
|Durable recovery without unnecessary repetition|Yes (durable state)|Stateful recovery possible, reproducibility weaker|Yes (Workflow checkpointing with bounded agent stages)|
|200+ document type classification|No (rule explosion)|Yes|Yes (AI Service Executor)|
|Inferred value extraction|No|Yes|Yes (AI Service Executor)|
|Semantic contract comparison|No|Yes|Yes (AI Service Executor, single LLM call)|
|Cross-document reasoning|No|Yes|Yes (Agent Executor for complex cases)|
|Open-ended queries|No|Yes|Yes (Agent Executor for complex; AI Service Executor for simple)|
|Cost per document at scale|Lowest|Highest|Low-to-middle (agents only where justified)|
|Auditability|Full (deterministic)|Requires infrastructure|Full (unified OpenTelemetry across all Executor patterns)|
|Reproducibility|Full|Probabilistic|Deterministic persistence; probabilistic reasoning in bounded stages|
|EU AI Act alignment|Yes|Difficult|Yes|
|Migration to managed compute|No framework-assisted portability|Partial, depending on hosting model|Best portability of the three, subject to runtime differences|
