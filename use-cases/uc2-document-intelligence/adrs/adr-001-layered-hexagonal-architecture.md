# UC2-ADR-001: Layered Hexagonal Architecture for UC2

**Status**: Draft  
**Date**: 13/03/2026  
**Decision Scope**: How UC2 separates orchestration, domain logic, infrastructure, and external resources.  
**Depends on**: Enterprise AI Platform reference architecture, ADR-003 (Network Isolation and Data Residency Enforcement), ADR-004 (Identity, Authentication, and Authorisation)  
**Depended on by**: UC2-ADR-002, UC2-ADR-003, UC2-ADR-004, UC2-ADR-005, UC2-ADR-006, UC2-ADR-007, UC2-ADR-008  

---

## Context

UC2 processes enterprise documents through a multi-step pipeline involving classification, extraction, confidence evaluation, human review, PII classification, embedding generation, and governed persistence. Each step depends on a different external capability: Microsoft Foundry for extraction and embeddings, Azure AI Language (via Foundry) for PII detection, Azure SQL for workflow state, Azure Service Bus for asynchronous message coordination, and ADLS Gen2 for staging.

Without an explicit separation model, domain logic (what the system decides) becomes coupled to infrastructure (how the system connects) and to the agent framework used for workflow orchestration. That coupling makes it difficult to test extraction logic without calling Foundry, difficult to change a storage backend without rewriting workflow code, difficult to swap or upgrade the agent framework without rebuilding domain logic, and difficult to reason about where a failure originated when multiple concerns are interleaved in the same component.

The platform provides the runtime (Azure Container Apps, networking, identity, observability) but does not prescribe how application code is structured internally. UC2 needs its own internal architecture pattern.

---

## Decision Drivers

- UC2 domain logic (classification routing, confidence evaluation, PII classification rules) must be testable without external service dependencies.
- Infrastructure adapters (Foundry client, SQL client, queue client, storage client) must be replaceable without modifying domain logic.
- The agent framework used for workflow orchestration is an implementation choice, not a permanent commitment. Domain logic must not depend on framework-specific APIs, calling conventions, or state management patterns.
- The Worker orchestrates a multi-step workflow involving five distinct AI and data capabilities. The boundary between orchestration decisions and service integration must be explicit.
- The team reviewing this architecture needs to understand where each concern lives without reading implementation code.
- Platform ADRs govern how external services are accessed (private endpoints, managed identity, regional constraints). The application architecture must make those integration points visible and isolable.

---

## Considered Alternatives

### Option A: Layered hexagonal architecture

Five explicit layers: External Inputs, Orchestration, Core Domain, Infrastructure Adapters, External Cloud Resources. Core domain components define protocols (interfaces). Infrastructure adapters implement those protocols against specific Azure services. The orchestration layer (Worker, DocumentProcessingWorkflow, Workflow Integration Adapters) coordinates execution sequence without containing domain logic. The Workflow Integration Adapters absorb framework-specific calling conventions, isolating the core domain from the agent framework.

Trade-off: More components and indirection. Requires discipline to keep domain logic out of adapters and orchestration out of domain components.

### Option B: Vertical slice per pipeline stage

Each pipeline stage (classify, extract, evaluate, tag, embed) implemented as a self-contained module with its own service calls, persistence, and error handling. No shared domain layer.

Trade-off: Simpler per-stage reasoning. But cross-cutting concerns (confidence evaluation affecting multiple stages, PII classification depending on extraction output format, workflow state spanning all stages) would be duplicated or handled inconsistently. Testing requires live services for every slice.

### Option C: Framework-driven orchestration

Use an off-the-shelf workflow engine (Durable Functions, Temporal, or similar) as the primary structuring mechanism. Domain logic lives inside activity functions. Infrastructure is handled by the framework.

Trade-off: Reduces custom orchestration code. But introduces a framework dependency that governs the entire application shape. Portability decreases. Testing still requires the framework runtime or its emulator. The framework makes decisions about retry, state, and dispatch that the architecture should control explicitly, particularly given the human review loop and confidence-based routing.

---

## Decision

UC2 adopts a layered hexagonal architecture with five layers.

**External Inputs**: Document ingestion events, operations users, human reviewers. These are the entry points into the system. They do not contain logic.

**Orchestration Layer**: Frontend (job management UI), API layer (job orchestration), Worker  (document processing), workflow coordination, and framework wrappers that isolate the agent framework from the core domain. This layer owns execution sequence, message coordination, and workflow progression. It does not contain domain logic.

**Core Domain**: Classification routing, document extraction, confidence evaluation, PII classification, and embedding preparation. These capabilities contain domain logic and decision rules. They define protocols for the infrastructure they need but do not reference specific Azure services or the agent framework. They are testable in isolation.

**Infrastructure Adapters**: Clients and stores that connect the core domain and orchestration layer to external services. Each adapter implements a protocol defined by the core domain or orchestration layer. Each adapter owns a single external integration concern against a defined service boundary.

**External Cloud Resources**: Microsoft Foundry (Content Understanding, Embedding Model, PII Detection via Azure AI Language), Azure SQL Database, Azure Service Bus, Document Ingestion Storage (Blob), Staging Storage (ADLS Gen2), Azure AI Search. These are consumed through infrastructure adapters only. No orchestration or domain component references them directly.

---

## Consequences

### Positive

- The core domain is framework-agnostic. If the agent framework is replaced, the Workflow and Workflow Integration Adapters absorb the change. Domain logic (classification routing, confidence evaluation, PII classification rules, embedding preparation) and infrastructure adapters remain unchanged.
- Core domain components are testable without Azure service dependencies. Unit tests validate classification routing, confidence thresholds, PII classification rules, and embedding preparation logic against protocol stubs.
- Replacing an infrastructure dependency (e.g., moving from Azure AI Search to a different index, or changing the queue implementation) requires changing one adapter without modifying domain logic or orchestration.
- Each layer has a clear responsibility. Code review, incident triage, and onboarding can orient around the layer model.
- Platform-level controls (private endpoints, managed identity, regional constraints) are enforced at the adapter layer, making compliance boundaries visible in the architecture rather than scattered through application code.

### Negative

- More components than a flat or vertical-slice design. The initial codebase has more files, more interfaces, and more indirection.
- Requires discipline. If domain logic leaks into adapters or orchestration, the testability and replaceability benefits degrade. This is a team discipline problem, not a structural one.
- The Workflow Integration Adapters layer (framework wrappers between the orchestration workflow and core domain) adds a translation step. This exists because the workflow framework has its own calling conventions. If the framework changes, this layer absorbs the impact, but it is an additional abstraction to maintain.

### Constraints introduced

- No core domain component may import or reference an Azure SDK, HTTP client, or external service directly.
- No infrastructure adapter may contain business logic (confidence thresholds, PII classification rules, classification routing).
- All communication between core domain and infrastructure passes through defined protocols.
- The orchestration layer coordinates sequence. It does not evaluate, classify, tag, or transform document content.

---

## Follow-ups

- Validate layer boundaries during implementation by enforcing import restrictions (e.g., linting rules that prevent core domain modules from importing Azure SDK packages).
- Review the Workflow Integration Adapters layer after the first delivery cycle. If the workflow framework stabilises and the translation overhead is minimal, consider whether this layer can be thinned.
