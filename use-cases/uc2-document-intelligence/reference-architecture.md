# UC2: Document Intelligence - Reference Architecture

**Status:** Draft  
**Date:** 25/03/2026 (last updated)  
**Repository:** `use-cases/uc2-document-intelligence/reference-architecture.md`  
**Platform dependency:** This use case deploys on the shared Enterprise AI Platform. See the [Enterprise AI Platform reference architecture](../../platform/reference-architecture.md).  

---

> **Note:** This is a reference architecture for governed document intelligence on the Microsoft stack. It is not a product and it is not designed for a specific organisation.
> 
> The patterns, decision records, and component choices are intended to be adapted to a specific context: the source systems, governance requirements, document types, and operational constraints of the organisation using it inform the final design. It is there for teams to reference, test ideas against, or use parts of in their own context. 
> 
> This architecture is actively evolving. Diagrams, workflows, and component details reflect current design decisions and will be refined as the system is built and validated against production constraints. Structural changes will be captured in updated ADRs.

---

## Context

UC2 addresses a common enterprise problem in document-heavy organisations: business-critical information is spread across contracts, amendments, side letters, NDAs, notices, correspondence, and related document sets across multiple systems, with no governed way to extract, structure, review, and operationalise it.

The use case is not limited to a single document type. It is designed for document intelligence across relationship-relevant enterprise documents where the organisation needs more than storage and search. It needs structured outputs, confidence-based control, human review where required, source traceability back to the originating system, and reusable downstream artefacts for analytics and retrieval. One example is building a governed view of the contractual relationship with a given client from the underlying document estate and related enterprise data. But that is one application of a broader capability designed to support any document-intensive domain.

The primary outcome of UC2 is governed structured output derived from source documents. That output supports downstream use in Microsoft Fabric and Azure AI Search, but the core value of the use case is not chat or retrieval. It is the controlled transformation of fragmented documents into auditable, reusable, production-grade data.

Documents originate in source systems, including document management systems such as, but not limited to, SharePoint Online, CRM platforms such as Dynamics 365, network file shares, email archives, and direct user uploads. Those systems differ in their APIs, authentication models, metadata structures, and governance controls. The ingestion boundary must standardise how documents and their source metadata enter the processing pipeline without forcing the core workflow to understand source-specific APIs or storage semantics. It must also accommodate different organisational requirements around document duplication. Some environments accept copies in platform-controlled storage. Others require that original documents remain solely in the source system.

This document describes the use-case-specific architecture only. It inherits the runtime, network, identity, encryption, monitoring, and shared AI platform assumptions defined in the Enterprise AI Platform reference architecture.

---

## Scope

### In scope

- Document ingestion from approved enterprise sources through a standard ingestion boundary
- Event-driven or contract-driven dispatch of document processing, depending on the ingestion mode selected for the deployment
- Document classification across supported document types
- Structured extraction using defined schemas
- Policy-driven workflow routing using confidence and other control inputs
- Policy-enabled agentic validation for low-confidence or policy-selected outputs
- Human review for outputs where policy requires explicit reviewer intervention
- PII classification of accepted structured outputs
- Persistence of workflow state and audit trail
- Persistence of governed structured outputs to staging
- Embedding generation for downstream indexed outputs
- Downstream Azure AI Search indexing of curated outputs
- Fabric consumption of staged outputs for relationship-oriented data products

### Out of scope

- Ad hoc conversational querying over indexed content (UC1: Chat with Data)
- Contract comparison and redline delta analysis (UC3)
- Audit-assistant and audit-preparation workflows (UC4)
- Formal legal interpretation or legal sign-off
- Autonomous approval of extracted outputs without human oversight unless explicitly permitted by documented policy
- Client-specific network topology, landing zone implementation, or private endpoint design
- Infrastructure-as-code modules, deployment pipelines, and runbooks
- Detailed database, queue, storage, or agent contract design
- Detailed Microsoft Purview taxonomy, DLP rules, or retention policy design
- Consumer-specific downstream access-control implementation such as Fabric row-level security policy definitions

---

## Design Goals

- **Transform fragmented documents into governed structured outputs.** The architecture must turn document content into reusable data rather than stopping at storage or manual review.
- **Keep quality control explicit.** Confidence evaluation is part of the workflow, but routing decisions are policy-driven rather than threshold-driven in isolation. Outputs that need secondary scrutiny follow a governed control path before operationalisation.
- **Reduce review burden without weakening control.** Where policy enables it, low-confidence or policy-selected outputs can be enriched through agentic validation before manual decision, improving reviewer speed while preserving explicit decision authority.
- **Preserve durable operational truth.** Workflow state, review status, retries, failures, validation outcomes, and audit history are stored in Azure SQL, not inferred from queue state.
- **Preserve source traceability without imposing duplication by default.** The architecture must retain source reference and source metadata through workflow and staging while supporting deployment-specific choices around persistent copy, transient copy, or no-copy ingestion.
- **Separate extraction from downstream consumption.** Structured outputs, embeddings, indexing, and analytics are distinct concerns with explicit handoff points.
- **Support regulated operation.** Human review, auditability, PII-aware output classification, and policy-driven control routing are architectural controls, not documentation-only controls.
- **Enable reuse across analytics and retrieval.** The same governed staged outputs support both Fabric analytics and downstream search indexing without duplicating the pipeline.
- **Operate continuously.** The default operating model is event-driven processing, not one-off batch migration.

---

## Relationship to the Shared Platform

UC2 is not a standalone system. It is implemented on top of the Enterprise AI Platform and inherits its baseline controls and operating assumptions.

UC2 reuses the platform's:

- Azure Container Apps runtime model
- Azure SQL operational state store
- Azure Service Bus asynchronous dispatch model
- Ingestion boundary and staging storage patterns
- Microsoft Foundry AI service integration
- Azure AI Search integration model
- Identity-first access model using managed identities
- Private-by-default networking posture
- Data protection baseline, including CMK/MMK decisions
- Observability and audit baseline
- Microsoft Fabric integration model

This document does not restate those shared platform decisions. It explains how UC2 uses them.

UC2 supports multiple ingestion modes at the use-case layer. Depending on deployment requirements, source documents may be processed through persistent copy, transient copy, or no-copy ingestion patterns. The shared platform provides the storage, messaging, identity, and runtime primitives required to support these patterns. UC2 defines how they are used.

UC2 also supports multiple validation paths at the control layer. Depending on policy, outputs may proceed directly after confidence evaluation, enter agentic validation before final routing, or require explicit human review. The shared platform provides the orchestration, runtime, and audit primitives required to support these patterns. UC2 defines how they are applied for document intelligence.

UC2 uses the Foundry-hosted Language API for PII detection. This is not a separate service dependency. PII detection is accessed through the same Foundry resource, private endpoint, and managed identity used by Content Understanding and the embedding model. No additional infrastructure provisioning is required beyond the platform baseline.

Where UC2 introduces use-case-specific behaviour, such as policy-driven routing, optional agentic validation, and human review where required, PII classification after accepted extraction output, source-aware ingestion, and staged output reuse for both Fabric and search indexing, those behaviours are documented here.

For the shared infrastructure topology, see the [Enterprise AI Platform reference architecture](../../platform/reference-architecture.md).

---

## Architecture: Data Flow View

The Data Flow View is the primary architecture diagram for UC-2. It shows the major processing components, control points, staging outputs, and downstream consumers.

![UC2 Data Flow View](diagrams/uc2-document-intelligence-data-flow-view.png)

Documents enter UC2 through source-aware ingestion components connected to approved enterprise sources. The ingestion boundary captures source metadata, preserves source reference, and passes a standard ingestion contract into the processing pipeline. Depending on the ingestion mode selected for the deployment, the document may be copied into platform-controlled ingestion storage, copied temporarily for the duration of processing, or processed without a platform-controlled document copy.

The Worker consumes ingestion messages and orchestrates classification, extraction, confidence evaluation, policy-driven routing, agentic validation where enabled, human review where required, PII classification, staging writes, and embedding generation. Azure AI Search indexes curated outputs downstream. Microsoft Fabric consumes staged structured data for the relationship intelligence view. Azure AI Search is a downstream consumer of curated outputs, not the primary control plane for the use case.

---

## Workflow

The UC2 workflow is event-driven and orchestrated by the Worker Container App with Azure SQL as the durable source of operational truth.

A document enters UC2 from an approved source through a source-aware ingestion component. The document may be a contract, amendment, notice, correspondence, NDA, or another supported business document.

The ingestion component captures source metadata and source reference, prepares the extraction input according to the selected ingestion mode, and submits a processing message through the standard ingestion boundary. This decouples source-specific retrieval from document processing and allows retry and resumability. Where the selected ingestion mode requires a platform-controlled copy, the document is written to ingestion storage before processing continues.

The Worker Container App consumes the message, creates a workflow record in Azure SQL with status `RECEIVED`, records the source reference and source metadata, and retrieves or accesses the extraction input supplied by the ingestion boundary.

The Worker classifies the document using Content Understanding. The resulting document type is recorded in Azure SQL with status `CLASSIFIED`.

The Worker submits an extraction request to Content Understanding using the schema appropriate for the classified document type. The extraction result returns structured JSON, confidence scores, and grounded output references. Azure SQL is updated with status `EXTRACTED`.

The Worker then evaluates the extraction outcome against the configured routing policy. This evaluation uses confidence, document type, field-level policy, and any other control inputs required by the workflow. Azure SQL is updated with status `ROUTING_EVALUATED`, recording the routing basis.

From this point, the workflow follows one of three governed paths.

Where the routing policy permits direct acceptance, the workflow records status `VALIDATION_ACCEPTED` and proceeds without manual review.

Where the routing policy requires intermediate validation, Azure SQL is updated with status `VALIDATION_IN_PROGRESS` and the extraction result enters the agentic validation step. This step performs structured secondary analysis on the extraction result using the available grounding, confidence, and policy context. It may apply alternative extraction strategies, cross-field consistency checks, and domain plausibility checks. The resulting validation outcome, revised confidence where applicable, recommendation, and validation trace are recorded in workflow state.

After agentic validation, one of three outcomes applies. If the validation outcome satisfies policy for acceptance, the workflow records status `VALIDATION_ACCEPTED` and proceeds. If policy still requires a human decision, the workflow records status `REVIEW_REQUIRED` and pauses pending review. If the validation path rejects or escalates the item out of the governed processing path, the workflow records status `REJECTED` and the item exits the workflow unless it is resubmitted manually.

Where the routing policy requires human review directly, without invoking agentic validation, the workflow records status `REVIEW_REQUIRED` and pauses pending review.

For outputs requiring review, the reviewer enters through the Frontend and API Container Apps to access the review queue. Pending review items are read from Azure SQL. The review pack includes the original extraction result, source grounding, confidence, and, where applicable, the validation outcome, revised confidence, recommendation, and sufficient trace to support the decision.

The reviewer either approves with corrections or rejects. If approved, Azure SQL is updated to `REVIEW_APPROVED`, corrections are recorded, and a resume-processing message is queued. If rejected, Azure SQL is updated to `REJECTED` and the item exits the workflow unless resubmitted manually.

When processing resumes after review approval, the Worker reads the workflow record and retrieves the accepted or corrected structured output.

The Worker then calls Azure AI Language to classify PII-bearing fields in the accepted structured output. This does not redact the output. It identifies PII-bearing fields and produces classification metadata for downstream governance. Azure SQL is updated with status `PII_CLASSIFIED`.

The Worker writes governed structured output to Staging Storage. This output includes structured extraction results, PII classification metadata, preserved source context, and final workflow disposition. Azure SQL is updated with status `STAGED`.

Where the selected ingestion mode uses a transient document copy, cleanup occurs only after successful staging and must not run before the governed output is confirmed written. Where the selected ingestion mode is no-copy, the workflow retains source reference and lineage but does not rely on a platform-held original document after ingestion.

The Worker separately generates embeddings for document chunks using the Foundry embedding model. Chunks, metadata, and embeddings are written to Staging Storage. Azure SQL is updated with status `EMBEDDED`.

Azure AI Search indexes curated outputs downstream. The indexer reads staged content via a resource instance rule on its configured schedule.

Microsoft Fabric consumes staged structured outputs via Trusted Workspace Access into the Relationship Intelligence Delta Table. This joins with enterprise data that has already progressed through the Fabric medallion architecture to Gold. The resulting Relationship Intelligence Semantic Model provides the governed analytics view consumed by Power BI.

The workflow completes with Azure SQL updated to `COMPLETED`, preserving the full operational and audit trail. That audit trail includes extraction outcome, confidence, routing outcome, validation outcome where used, review decisions, PII classification, timestamps, source metadata, and lineage.

![UC2 Workflow View](diagrams/uc2-document-intelligence-workflow-view.png)

---

## Components

### Frontend Container App

Provides the operational user interface for job monitoring and human review. In UC2 it is not a retrieval or RAG interface. Its role is to expose job status, pending review items, failures, and user actions such as approval, correction, and rerun. The Frontend and API Container Apps exist to serve the UC2 document processing workflow. They are not a general-purpose application layer. Use cases requiring conversational or query interfaces, such as UC1, are expected to use platform-native delivery channels such as Copilot Studio or Fabric Data Agents rather than extending these components.

### API Container App

Provides the synchronous application interface required for operational interaction with the UC2 workflow. It retrieves workflow status, returns pending review items, accepts review decisions, and submits or reruns jobs through the message bus. It is not intended as a general-purpose consumption API for broader query or conversational use cases.

### Source-Aware Ingestion Components

Retrieves documents from approved source systems, captures source metadata and source reference, prepares the extraction input according to the selected ingestion mode, and submits work into the processing pipeline through the standard ingestion contract. These components isolate source-system-specific logic from the Worker.

### Worker Container App

Owns the orchestration of the asynchronous document processing workflow. It accesses the extraction input supplied by the ingestion boundary, calls classification and extraction services, evaluates extraction outcomes, applies routing policy, invokes agentic validation where enabled, and persists workflow state transitions, resumes approved workflows, coordinates PII classification and embeddings, writes outputs, and updates operational state in Azure SQL.

### Agentic Validation Node

Performs structured secondary validation for low-confidence or policy-selected outputs before final routing where invoked by policy. It may use multiple validation paths to challenge or refine the original extraction against source grounding, schema expectations, and domain plausibility rules. It produces a revised confidence assessment, recommendation, and traceable validation record. It supports human review; it does not replace it where policy requires explicit human approval.

### Content Understanding (Microsoft Foundry)

Performs document classification and structured extraction. It returns typed outputs, confidence signals, and grounded extraction results. In UC2 it is used for extraction, not for workflow routing or decision-making.

### Embedding Model (Microsoft Foundry)

Generates vector embeddings for document chunks as a downstream enrichment step. It is separate from extraction and does not replace the structured output path.

### PII Classifier (Azure AI Language)

Classifies PII-bearing fields in accepted structured outputs. Its role in UC2 is classification and metadata enrichment, not pre-extraction redaction. PII detection uses Azure AI Language, accessed through the Microsoft Foundry resource. It shares the same private endpoint, managed identity, and regional constraints as Content Understanding and the embedding model. No separate Azure AI Language resource is provisioned.

### Azure SQL Database

Stores workflow state, review state, source reference, source metadata, validation outcomes, timestamps, retries, failures, approvals, rejections, and audit trail. It is the durable operational truth of the use case.

### Azure Service Bus

Provides asynchronous dispatch and workflow resumption where the selected ingestion mode uses message-based decoupling. It coordinates movement of work, but it is not a system of record.

### Document Ingestion Storage (Azure Blob Storage)

Used when the selected ingestion mode requires a platform-controlled document copy. It may hold source documents persistently or transiently, depending on deployment choice. Documents stored here are not modified after ingestion.

### Staging Storage (ADLS Gen2)

Stores governed structured output and chunk-plus-embedding output. It is the shared handoff point for downstream consumers.

### Azure AI Search

Indexes curated document-derived outputs downstream. In UC2 it is a consumer of staged artefacts, not the primary control plane.

### Microsoft Fabric

Consumes governed structured outputs into analytics-oriented data structures and semantic models, including the Relationship Intelligence Delta Table and Relationship Intelligence Semantic Model.

The logical architecture separates UC2 into six layers: External Inputs, Ingestion Boundary, Orchestration Layer, Core Domain, Infrastructure Adapters, and External Cloud Resources. This view shows that source-specific ingestion, workflow orchestration, domain logic, infrastructure adapters, and cloud resources are distinct concerns. The ingestion boundary standardises entry into the workflow. The Worker and workflow coordinate execution. The core domain contains the logic for classification, extraction, confidence evaluation, policy-driven routing, agentic validation where enabled, PII classification, and embedding generation. Infrastructure adapters connect those capabilities to platform services.

![UC2 Logical View](diagrams/uc2-document-intelligence-logical-view.png)

---

## Control Boundaries and Governance

UC2 has five control boundaries that define where quality, oversight, and governance obligations sit.

### Ingestion boundary

Documents do not enter the workflow directly from arbitrary source systems or through unmanaged storage events. They enter through an approved ingestion boundary that captures source reference and source metadata, standardises the processing input, and enforces the permitted ingestion mode for the deployment. This is the boundary that separates source-specific retrieval concerns from governed document processing.

### Routing boundary

Extraction does not automatically become operational output. The extraction result is evaluated against routing policy, using confidence, document type, field-level policy, and any other configured control inputs required by the use case. This is the first major control gate. It determines whether an item can proceed directly, requires intermediate validation, or must pause for human review.

### Agentic validation boundary

Where policy enables it, outputs that require secondary scrutiny enter an explicit validation stage before final routing. This stage may challenge, refine, or downgrade the original result using multiple validation paths and domain-aware checks. It is a control-enhancing boundary, not an autonomous approval boundary by default. Its role is to enrich the evidence available to the workflow and, where permitted, support selective automatic acceptance.

### Human review boundary

Where policy requires manual approval, outputs cross into explicit human oversight. Reviewers approve, correct, or reject extraction results before the workflow can continue. They are presented with the original extraction, source grounding, confidence, and, where applicable, the validation outcome, revised confidence, recommendation, and sufficient reasoning context to support the decision. Human review remains the authority where policy mandates explicit sign-off.

### PII classification boundary

PII classification occurs after the output is accepted, not before extraction. Content Understanding requires the raw document or equivalent extraction input to extract parties, signatories, addresses, and obligations. Redacting PII before extraction would destroy the data the system is designed to extract. Instead, accepted outputs are enriched with PII classification metadata so downstream consumers such as Fabric RLS, AI Search field-level ACLs, and semantic model masking can enforce access boundaries appropriate to their context.

### Staging boundary

Staging Storage is the governed handoff point. Only accepted structured output and derived search artefacts are written there. Downstream consumers, including Fabric and Azure AI Search, consume from this point rather than directly from raw extraction steps, routing outcomes, or intermediate validation artefacts.

### EU AI Act

If outputs from this system are used in contexts that fall within EU AI Act high-risk categories, additional obligations may apply, including risk management, technical documentation, human oversight, and auditability. In UC2, the relevant architectural controls include explicit routing, optional intermediate validation, human review where required, and durable workflow state in Azure SQL. Enforcement deadline: August 2026.

### GDPR and data residency

All processing services are deployed in an EU Azure region. Foundry model deployments use DataZone Standard EU to ensure prompts and completions remain within EU boundaries. Storage, Foundry, AI Search, AI Language, and Fabric are all EU-region-locked. Azure Policy blocks Global Standard deployments. Documents containing personal data of EU-based parties require this residency guarantee.

---

## Data and State Model

UC2 deliberately separates raw content, workflow state, governed outputs, and downstream indexing artefacts.

### Raw documents

Raw document handling depends on the selected ingestion mode. In persistent-copy mode, source documents are retained in platform-controlled ingestion storage. In transient-copy mode, they are held there only for the duration required to complete processing successfully. In no-copy mode, the original document remains solely in the source system and the pipeline operates on the extraction input provided by the ingestion boundary.

### Workflow and review state

Stored in Azure SQL Database. This includes status progression, source reference, source metadata, document classification result, extraction result, confidence evaluation result, routing outcome, validation outcome where applicable, review-required flags, reviewer decisions, PII audit logs, rerun and retry history, timestamps, and lineage references.

Representative workflow states include `RECEIVED`, `CLASSIFIED`, `EXTRACTED`, `ROUTING_EVALUATED`, `VALIDATION_IN_PROGRESS`, `VALIDATION_ACCEPTED`, `REVIEW_REQUIRED`, `REVIEW_APPROVED`, `REJECTED`, `PII_CLASSIFIED`, `STAGED`, `EMBEDDED`, and `COMPLETED`.

These states are representative control and lifecycle markers for the reference architecture. They illustrate how the workflow distinguishes ingestion, extraction, routing, optional validation, review, downstream enrichment, and completion. They are not intended to prescribe a final database schema, enumeration contract, or implementation-specific state machine.

Where agentic validation is used, workflow state also retains sufficient evidence to explain the decision path. This includes the original extraction outcome, original confidence, validation invocation, validation outcome, revised confidence where produced, recommendation, and trace references sufficient to support audit and review.

### Governed structured outputs

Stored in Staging Storage. These include extracted structured data, preserved source context, final workflow disposition, and PII classification metadata.

### Chunk and embedding outputs

Also stored in Staging Storage. These support downstream indexing and retrieval scenarios.

### Indexed search artefacts

Stored in Azure AI Search. These are derived from curated outputs and are downstream consumers of the UC2 pipeline.

### Relationship-oriented analytics outputs

Materialised in Microsoft Fabric through the Relationship Intelligence Delta Table and downstream Relationship Intelligence Semantic Model.

---

## Assumptions and Constraints

### Assumptions

- Documents enter the processing pipeline through approved ingestion paths and a standard ingestion boundary.
- Supported document types evolve over time.
- Source document quality varies by age, format, and origin.
- Routing policy may distinguish between direct acceptance, intermediate validation, and human review depending on field, document type, and processing context.
- Downstream analytics and retrieval consumers use governed staged outputs rather than raw extraction responses or intermediate workflow artefacts.
- The use case runs inside the shared platform runtime and inherits its controls.

### Constraints

- Confidence is an input to routing, not by itself the final control decision.
- Outputs requiring secondary scrutiny must pass through configured routing controls before operationalisation.
- Agentic validation is optional and policy-enabled. It must not be assumed for all documents, fields, or workflows.
- Agentic validation recommendations are advisory unless policy explicitly permits automatic acceptance.
- Human review remains mandatory where policy requires explicit reviewer approval.
- Durable operational truth is in Azure SQL, not in Service Bus.
- The queue coordinates work but does not define final state.
- Source reference and source metadata must be preserved through workflow state and governed staging regardless of ingestion mode.
- The use case must not assume persistent document duplication unless the selected deployment mode explicitly permits it.
- Validation outcomes that influence routing must be auditable and traceable.
- PII classification quality depends on the accepted extraction output and the capabilities of the classification service.
- Linking outputs to canonical customer identifiers depends on the availability and quality of upstream CRM or master data.
- Fabric and Azure AI Search consume outputs through approved downstream patterns only.
- The use case does not define downstream consumer access-control policy implementation.

---

## Alternatives Considered

**Direct document-to-search pipeline without structured staging.** Rejected because it eliminates the governed structured output that Fabric and other downstream systems need. Search alone does not produce the relationship-oriented analytics view.

**Extraction without human review or equivalent governed routing.** Rejected because it would not survive regulated scrutiny where extracted data influences legal, financial, or commercial decisions.

**Queue-only workflow state.** Rejected because queues are transient dispatch mechanisms. Durable operational state requires a relational store with query, reporting, and audit capability.

**Collapsing extraction and embeddings into one opaque AI step.** Rejected because they serve different purposes, use different AI capabilities, and produce different output types. Keeping them separate preserves operational clarity and control.

**Persistent copy as the only ingestion model.** Rejected because some environments require that original documents remain solely in the source system or accept only transient duplication during processing.

**Direct routing from low-confidence extraction to human review only.** Rejected as the sole pattern because it gives reviewers limited secondary evidence and does not distinguish between recoverable and genuinely ambiguous cases.

---

## Extending the Use Case

UC2 can be extended without redesigning the shared platform.

Adding new document types requires new or refined extraction schemas and classification handling. Stronger customer or entity resolution can be introduced where upstream CRM or master data is available. Additional downstream consumers can be added to Staging Storage where the access pattern remains governed. Future retrieval-oriented use cases, including UC1, may consume indexed outputs produced by UC2. Future document-centric use cases such as comparison (UC3) and audit preparation (UC4) can reuse parts of the extraction, validation, review, staging, and source-aware ingestion model defined here.

A new use-case architecture is warranted when the autonomy boundary, consumer pattern, or regulatory handling materially changes.

---

## Document Status and Evolution

This reference architecture describes the current architectural position for UC2. It is intended to guide design, implementation, and review, but it is not a final product specification or immutable implementation contract.

As the use case is built and validated, material changes to workflow, controls, ingestion, or component responsibilities should be captured in decision records and then reflected here.