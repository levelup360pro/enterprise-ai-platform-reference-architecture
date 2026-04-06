# UC1: Chat with Data

Provides a governed conversational interface over two downstream planes that already exist or are planned on the platform: Microsoft Fabric for curated analytical truth and Azure AI Search for indexed, source-grounded enterprise knowledge. UC1 is the downstream conversational consumption layer for governed outputs produced by the shared platform and by UC2.

**Platform dependency:** This use case deploys on the shared [Enterprise AI Platform](../../platform/reference-architecture.md).

**Start here:** [reference-architecture.md](reference-architecture.md)

> **Note:** This is a reference architecture for a governed conversational system over enterprise data on the Microsoft stack. It is not a product and it is not designed for a specific organisation.
> 
> The patterns, decision records, and component choices are intended to be adapted to a specific context: the source systems, governance requirements, document types, retrieval patterns, and operational constraints of the organisation using it inform the final design. It is there for teams to reference, test ideas against, or use parts of it in their own context.
> 
> This architecture is actively evolving. Diagrams, workflows, and component details reflect current design decisions and will be refined as the system is built and validated against production constraints. Structural changes will be captured in updated ADRs.

## UC1 ADRs

| ADR     | Title                                               | Decision Scope                                                                                                                              | Status   |
| ------- | --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| ADR-001 | Conversational Orchestration Layer Selection        | How UC1 selects the primary orchestration layer and its relationship to delivery channels                                                   | Accepted |
| ADR-002 | Data Retrieval Pattern                              | Whether UC1 should retrieve through direct Fabric querying, Azure AI Search grounded retrieval, or a governed hybrid                        | Accepted |
| ADR-003 | Delivery Channel Selection                          | Which end-user channels UC1 should support now versus later, and which integration pattern connects Copilot Studio to Foundry Agent Service | Draft    |
| ADR-004 | Knowledge Layer Strategy                            | Whether Azure AI Search, Fabric semantic assets, Fabric IQ, or Foundry IQ should anchor the long-term knowledge layer                       | Accepted |
| ADR-005 | Response Grounding and Source Attribution Boundary  | How UC1 decides an answer is sufficiently grounded and what source attribution standard the architecture enforces                           | Draft    |
| ADR-006 | Identity Propagation and Permission-Aware Retrieval | How end-user authorization propagates through channels, agent tooling, Fabric, and Azure AI Search                                          | Draft    |
| ADR-007 | Operational State and Audit Trail                   | Where UC1 stores durable operational truth for conversations, retrieval decisions, and audit metadata                                       | Draft    |
| ADR-008 | Consumption of UC2 Outputs and Governance Metadata  | How UC1 consumes UC2 outputs while preserving staging, lineage, review, and governance boundaries                                           | Draft    |

## Diagrams

| View                              | Purpose                                                                                                          | Path                                                                                                                                                                                             |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Platform Infrastructure View      | Shared platform context inherited by UC2                                                                         | [../../platform/diagrams/enterprise-ai-platform-reference-architecture-Infrastructure View.png](../../platform/diagrams/enterprise-ai-platform-reference-architecture-Infrastructure%20View.png) |
| Logical View                      | Foundry agent orchestration, Azure AI Search knowledge layer, Fabric analytical adjunct, audit and observability | [diagrams/uc1-chat-with-data-logical-view.png](diagrams/uc1-chat-with-data-logical-view.png)                                                                                                     |
| Dual Output Data Flow             | UC2 pipeline through staging to Fabric Gold layer and AI Search index, with UC1 as read-only consumer of both    | [diagrams/uc1-chat-with-data-dual-output-data-flow.png](diagrams/uc1-chat-with-data-dual-output-data-flow.png)                                                                                   |
| Workflow View                     | Request lifecycle from user through channel, mode classification, retrieval, grounding validation, and audit     | [diagrams/uc1-chat-with-data-workflow-view.png](diagrams/uc1-chat-with-data-workflow-view.png)                                                                                                   |
| Governance and Control Boundaries | Channel, front-end, orchestration, retrieval, governed data, and audit boundaries                                | [diagrams/uc1-chat-with-data-governance-and-control-boundaries.png](diagrams/uc1-chat-with-data-governance-and-control-boundaries.png)                                                           |

## Reading Order

1. **Reference architecture** for the full UC1 design, scope, workflow, components, governance model, and deployment guidance.
2. **ADRs** in sequence (001 through 008) for the rationale behind specific architectural decisions.
3. **Diagrams** are embedded inline in the reference architecture as Mermaid diagrams.
