# UC2: Document Intelligence

Transforms fragmented enterprise documents (contracts, amendments, NDAs, correspondence) into governed structured outputs for downstream analytics in Microsoft Fabric and indexed retrieval via Azure AI Search.

**Platform dependency:** This use case deploys on the shared [Enterprise AI Platform](../../platform/reference-architecture.md).

**Start here:** [reference-architecture.md](reference-architecture.md)

## UC2 ADRs

| ADR     | Title                                         | Decision Scope                                                                        | Status |
| ------- | --------------------------------------------- | ------------------------------------------------------------------------------------- | ------ |
| ADR-001 | Layered Hexagonal Architecture for UC2        | How UC2 separates orchestration, domain logic, infrastructure, and external resources | Draft  |
| ADR-002 | Post-Extraction PII Classification            | When and how PII is identified in UC2 extraction outputs                              | Draft  |
| ADR-003 | Confidence-Based Workflow Routing             | How extraction quality determines workflow progression                                | Draft  |
| ADR-004 | Worker as the Single Write Path               | Which component owns authoritative writes to workflow state and staging               | Draft  |
| ADR-005 | Separate Extraction from Embedding Generation | Whether extraction and embedding are combined or independent pipeline stages          | Draft  |
| ADR-006 | Staging as the Governed Downstream Boundary   | Where the boundary sits between UC2 processing and downstream consumers               | Draft  |
| ADR-007 | Azure SQL as the Workflow System of Record    | Where workflow state and operational truth are persisted                              | Draft  |
| ADR-008 | Schema-Defined Extraction by Document Type    | How extraction schemas are selected and managed across document types                 | Draft  |

## Diagrams

| View                         | Purpose                                                                                                                          | Path                                                                      |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| Platform Infrastructure View | Shared platform context inherited by UC2                                                                                         | [../../platform/diagrams/enterprise-ai-platform-reference-architecture-Infrastructure View.png](../../platform/diagrams/enterprise-ai-platform-reference-architecture-Infrastructure%20View.png) |
| UC2 Data Flow View           | Primary UC2 architecture view showing components, processing flow, control points, staging boundary, and downstream consumers    | [diagrams/uc2-document-intelligence-data-flow-view.png](diagrams/uc2-document-intelligence-data-flow-view.png) |
| UC2 Logical View             | Layered hexagonal architecture showing separation of orchestration, core domain, infrastructure adapters, and external resources | [diagrams/uc2-document-intelligence-logical-view.png](diagrams/uc2-document-intelligence-logical-view.png) |
| UC2 Workflow View            | End-to-end operational lifecycle with status transitions, human review interaction, and audit trail                               | [diagrams/uc2-document-intelligence-workflow-view.png](diagrams/uc2-document-intelligence-workflow-view.png) |

## Reading Order

1. **Reference architecture** for the full UC2 design, scope, workflow, components, and governance model.
2. **ADRs** in sequence (001 through 008) for the rationale behind specific architectural decisions.
3. **Diagrams** are embedded inline in the reference architecture. The files above are the editable sources.
