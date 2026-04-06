# UC2: Document Intelligence

Transforms fragmented enterprise documents (contracts, amendments, NDAs, correspondence) into governed structured outputs for downstream analytics in Microsoft Fabric and indexed retrieval via Azure AI Search.

**Platform dependency:** This use case deploys on the shared [Enterprise AI Platform](../../platform/reference-architecture.md).

**Start here:** [reference-architecture.md](reference-architecture.md)

> **Note:** This is a reference architecture for governed document intelligence on the Microsoft stack. It is not a product and it is not designed for a specific organisation.
> 
> The patterns, decision records, and component choices are intended to be adapted to a specific context: the source systems, governance requirements, document types, and operational constraints of the organisation using it inform the final design. It is there for teams to reference, test ideas against, or use parts of in their own context. 
> 
> This architecture is actively evolving. Diagrams, workflows, and component details reflect current design decisions and will be refined as the system is built and validated against production constraints. Structural changes will be captured in updated ADRs.
> 
## UC2 ADRs

| ADR     | Title                                                     | Decision Scope                                                                                                 | Status   |
| ------- | --------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- | -------- |
| ADR-001 | Layered Hexagonal Architecture for UC2                    | How UC2 separates orchestration, domain logic, infrastructure, and external resources                          | Accepted |
| ADR-002 | Post-Extraction PII Classification                        | When and how PII is identified in UC2 extraction outputs                                                       | Accepted |
| ADR-003 | Policy-Driven Workflow Routing                            | How extraction outcomes are routed using confidence and other control inputs                                   | Accepted |
| ADR-004 | Worker as the Single Write Path                           | Which component owns authoritative writes to workflow state and staging                                        | Accepted |
| ADR-005 | Separate Extraction from Embedding Generation             | Whether extraction and embedding are combined or independent pipeline stages                                   | Accepted |
| ADR-006 | Staging as the Governed Downstream Boundary               | Where the boundary sits between UC2 processing and downstream consumers                                        | Accepted |
| ADR-007 | Azure SQL as the Workflow System of Record                | Where workflow state and operational truth are persisted                                                       | Accepted |
| ADR-008 | Schema-Defined Extraction by Document Type                | How extraction schemas are selected and managed across document types                                          | Accepted |
| ADR-009 | Source-Aware Ingestion and Metadata Preservation          | How documents enter UC2 and how source reference and metadata are preserved across the workflow                | Accepted |
| ADR-010 | Duplicate Handling and Reprocessing Semantics             | How UC2 distinguishes repeated ingestion, source-document change, and intentional reprocessing                 | Draft    |
| ADR-011 | Failure Handling, Retry Budget, and Dead-Letter Semantics | How UC2 handles failures, bounds retries, and represents terminal failure for operational recovery             | Draft    |
| ADR-012 | Agentic Validation Between Routing and Human Review       | How UC2 uses optional agentic validation to enrich, resolve, or escalate policy-selected outputs before review | Accepted |

## Diagrams

| View                         | Purpose                                                                                                                          | Path                                                                                                                                                                                             |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Platform Infrastructure View | Shared platform context inherited by UC2                                                                                         | [../../platform/diagrams/enterprise-ai-platform-reference-architecture-Infrastructure View.png](../../platform/diagrams/enterprise-ai-platform-reference-architecture-Infrastructure%20View.png) |
| UC2 Data Flow View           | Primary UC2 architecture view showing components, processing flow, control points, staging boundary, and downstream consumers    | [diagrams/uc2-document-intelligence-data-flow-view.png](diagrams/uc2-document-intelligence-data-flow-view.png)                                                                                   |
| UC2 Logical View             | Layered hexagonal architecture showing separation of orchestration, core domain, infrastructure adapters, and external resources | [diagrams/uc2-document-intelligence-logical-view.png](diagrams/uc2-document-intelligence-logical-view.png)                                                                                       |
| UC2 Workflow View            | End-to-end operational lifecycle with status transitions, human review interaction, and audit trail                              | [diagrams/uc2-document-intelligence-workflow-view.png](diagrams/uc2-document-intelligence-workflow-view.png)                                                                                     |

## Reading Order

1. **Reference architecture** for the full UC2 design, scope, workflow, components, and governance model.
2. **ADRs** in sequence (001 through 008) for the rationale behind specific architectural decisions.
3. **Diagrams** are embedded inline in the reference architecture. The files above are the editable sources.
