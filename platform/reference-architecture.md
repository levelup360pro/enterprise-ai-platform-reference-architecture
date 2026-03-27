# Enterprise AI Platform: Reference Architecture

**Status:** Draft  
**Date:** 13/03/2026 (last updated)  
**Repository:** `platform/reference-architecture.md`  

---

## Context

This reference architecture defines the shared platform for the enterprise AI workloads documented in this repository. The platform provides private networking, customer-managed encryption, identity-based access control, centralised observability, and AI safety controls as shared infrastructure. All use-case-specific workloads deploy on top of this platform and inherit its security, governance, and operational baseline.

The architecture targets organisations operating under regulatory obligations such as the EU AI Act, GDPR, and sector-specific frameworks in financial services, healthcare, public sector, and legal. It assumes a Microsoft Azure foundation with Microsoft Fabric as an external data platform.

This document describes the platform-level architecture only. Use-case-specific reference architectures build on this baseline and document the additional flows, controls, data handling, and operating constraints required for each use case.

---

## Scope

### In scope

- Shared platform runtime for AI workloads
- Enterprise ingress assumptions and frontend access path
- Private networking baseline for platform services
- Identity and authentication model at platform level
- Shared storage, messaging, search, AI, database, and monitoring components
- Fabric integration pattern for downstream analytics and data consumption
- Platform-level security, encryption, and observability posture
- Reusable baseline for use-case-specific reference architectures

### Out of scope

- Use-case-specific prompt design, schemas, orchestration logic, and evaluation logic
- Client-specific hybrid connectivity implementation details
- SOC, SIEM, and central security workspace design
- Detailed Microsoft Purview taxonomy, labelling, DLP, or retention policy design
- Low-level IaC modules, exact subnet sizing, or route table configuration
- Runbooks, operational procedures, and incident response playbooks
- API contracts, queue payload schemas, and database schema design

---

## Design Goals

The platform is designed to meet the following architectural goals:

- **Regulated-environment suitability.** Governance, traceability, and controllability are treated as architectural requirements, not post-build controls.
- **Private-by-default connectivity.** Platform-facing services are accessed privately wherever supported, with scoped exceptions only where required by documented Microsoft service patterns.
- **Identity-first access control.** Managed identities are the default for service-to-service access. Embedded secrets and shared keys are avoided.
- **Separation of concerns.** User access, synchronous API handling, asynchronous processing, operational state, staged outputs, retrieval, and analytics are separated into explicit architectural roles.
- **Reusable platform baseline.** Multiple AI use cases can deploy on the same foundation without redesigning the core runtime, networking, security, and observability model.
- **Production-first design.** The architecture is intended to survive operational, audit, and security scrutiny rather than optimise for rapid proof-of-concept delivery.

---

## Architecture

![Enterprise AI Platform Infrastructure View](diagrams/enterprise-ai-platform-reference-architecture-Infrastructure%20View.png)

The platform deploys into a single EU Azure region within an AI Landing Zone subscription. It assumes an existing Connectivity Subscription, following the Platform Landing Zone pattern, provides shared ingress, egress, and DNS services. The platform peers into this existing network infrastructure. Microsoft Fabric operates as a separate SaaS platform outside the workload network boundary.

This is a reference architecture view. It shows the major components, trust boundaries, and interaction patterns. It does not describe low-level implementation details such as IaC modules, exact subnet sizing, detailed RBAC assignments, or runbook procedures.

---

## Workflow

1. **Enterprise users** access the platform through approved private enterprise access patterns. The exact access method is client-specific and determined during deployment. Common examples include VPN and ExpressRoute. Alternative access models, such as Azure Virtual Desktop, are valid but are not represented as the primary ingress path in this view. At the platform boundary, user access resolves to HTTPS 443 / TLS 1.2+. Traffic enters through the Connectivity Subscription, routes through Azure Firewall in the Hub VNet, crosses a VNet peering connection into the Spoke VNet, and reaches the Frontend Container App.
    
2. **The Frontend Container App** receives user requests and forwards them to the API Container App over HTTPS 443 within the internal Container Apps Environment. Both apps run in the App Subnet of the Spoke VNet. The environment is configured as internal with no public endpoint.
    
3. **The API Container App** processes requests and delegates long-running or asynchronous tasks to the Worker Container App via Azure Service Bus (AMQP over TLS, TCP 5671). Service Bus acts as the message broker, decoupling request intake from processing. The Worker Container App consumes messages and executes asynchronous workload processing, AI inference orchestration, and background workflows. The exact orchestration, routing, and review semantics are defined in the relevant use-case architecture.
    
4. **All compute-layer services** communicate with PaaS services exclusively through private endpoints deployed in the Private Endpoints Subnet. Each private endpoint resolves via a linked Private DNS Zone, keeping all traffic on the Azure backbone. Private endpoint connections exist for Key Vault, AI Search, Staging Storage, Microsoft Foundry, Azure SQL Database, Service Bus, and the Azure Monitor Private Link Scope (AMPLS). Where a use case requires a platform-controlled document copy, Document Ingestion Storage is also accessed through a private endpoint.
    
5. **Microsoft Foundry** provides the AI services layer including model inference, content safety, and Content Understanding for document intelligence. Foundry guardrails enforce content filtering on model interactions. Safety, validation, and PII handling occur at workflow boundaries defined by the relevant use-case architecture rather than being assumed as a single fixed platform pattern.
    
6. **Azure AI Search** stores and queries vector and keyword indexes over the document corpus. The compute layer queries AI Search through its private endpoint. The AI Search indexer reads from Staging Storage via a resource instance rule on the storage firewall rather than a private endpoint. AI Search accesses Key Vault for CMK operations via a shared private link, a search-service-managed private endpoint that is distinct from both the customer-deployed private endpoint and the trusted-services firewall bypass.
    
7. **Microsoft Fabric** operates outside the workload network boundary as a SaaS platform. Fabric accesses Staging Storage via Trusted Workspace Access using its Workspace Identity. Data flows from external sources into Fabric via Data Factory connectors following the Medallion Architecture. The AI platform writes governed outputs to Staging Storage. Fabric consumes those outputs via Trusted Workspace Access into the Relationship Intelligence Delta Table, joining them with enterprise data flowing through the Medallion Architecture.
    
8. **Telemetry from the compute layer** flows to Application Insights and the Workload Log Analytics Workspace through the AMPLS private endpoint in private-only mode. PaaS services emit diagnostic logs to the same Log Analytics Workspace via Azure backbone diagnostic settings.
    
9. **Azure Firewall** inspects and controls all egress traffic. User-defined routes on compute subnets force outbound internet traffic through the firewall, which enforces FQDN-based allow lists. All traffic not explicitly allowed is denied.
    
10. **DNS resolution** for all privatelink.* domains is handled by Private DNS Zones linked to the Spoke VNet. The DNS Private Resolver in the Connectivity Subscription enables hybrid DNS resolution for on-premises or enterprise clients.
    

---

## Trust Boundaries

The platform has five trust boundaries that define where access control, monitoring, identity, and governance assumptions change:

**Enterprise ingress boundary.** Between enterprise users and the platform entry point. The access method is client-controlled. The platform assumes traffic arrives authenticated and routed through approved private network controls before reaching the Frontend Container App.

**Runtime boundary.** Around the Container Apps Environment. All three container apps (Frontend, API, Worker) run within an internal environment with no public endpoint. Inter-app communication stays within this boundary.

**Private endpoint boundary.** Between the workload network and Azure-managed PaaS services. All compute-to-PaaS communication crosses this boundary via private endpoints. This is the primary network isolation control.

**Managed service boundary.** Around Azure PaaS dependencies (Microsoft Foundry, AI Search, Storage, SQL, Service Bus, Key Vault, monitoring services). These services are managed by Azure. The platform controls access to them via private endpoints, managed identity, and RBAC but does not control their internal operation.

**Fabric boundary.** Between the Azure workload and Microsoft Fabric. Fabric is a SaaS platform outside the workload network boundary. It accesses platform storage through Trusted Workspace Access with its own identity. The platform produces governed outputs; Fabric consumes them for analytics and downstream use.

---

## Components

### Azure Container Apps

Azure Container Apps provides the compute layer. Three container apps (Frontend, API, Worker) run within a single internal Container Apps Environment integrated into the App Subnet. Container Apps was selected for its native VNet integration, scale-to-zero capability, and managed infrastructure without the operational overhead of AKS. The environment exposes no public endpoint.

### Microsoft Foundry

Microsoft Foundry is the unified AI services layer. It hosts model deployments, Content Safety, and Content Understanding (document extraction and classification). A single Foundry resource in the EU region covers all AI capabilities. Foundry accesses Key Vault for CMK via the trusted-services firewall bypass. See ADR-005 for encryption detail.

### Azure AI Search

Azure AI Search provides vector search, keyword search, hybrid search, and semantic ranking across the document corpus. AI Search is configured with CMK encryption. The indexer reads from Staging Storage via a resource instance rule. AI Search accesses Key Vault for CMK via a shared private link, which is a search-service-managed private endpoint requiring provisioning on the search service and approval on Key Vault. See ADR-003 for the full private endpoint inventory and ADR-005 for CMK configuration.

### Document Ingestion Storage (Azure Blob Storage)

Document Ingestion Storage is used where a use case requires a platform-controlled copy of source documents. Depending on the selected ingestion mode, documents may be retained persistently, held transiently for processing, or remain solely in the source system. Where used, the compute layer accesses this account via a private endpoint. CMK encryption is enabled. See ADR-005 for key management detail.

### Staging Storage (Azure Data Lake Storage Gen2)

Staging Storage serves as the integration point between the AI platform and Microsoft Fabric. The compute layer writes via a private endpoint. Fabric reads via Trusted Workspace Access. The AI Search indexer reads via a resource instance rule. This account has two non-private-endpoint access paths, both scoped and documented in ADR-003. CMK encryption is enabled.

### Operational Database (Azure SQL Database)

Azure SQL Database stores workflow state, processing metadata, human review queue state, and application configuration. The compute layer connects via private endpoint using TDS over TLS. Data at rest is encrypted via Transparent Data Encryption (TDE) using a customer-managed key from Azure Key Vault. See ADR-005 for encryption detail.

### Message Bus (Azure Service Bus)

Azure Service Bus provides asynchronous message dispatch between the API and Worker Container Apps. It supports queue-based load levelling, retry semantics, and dead-letter handling. CMK encryption is enabled.

### Azure Key Vault (Premium)

Azure Key Vault stores customer-managed encryption keys. Configured with soft-delete, purge protection, and RBAC access model. Public access is disabled. The compute layer accesses Key Vault via a private endpoint. Storage, Foundry, Service Bus, and Fabric access Key Vault for CMK via the trusted-services bypass. AI Search uses a shared private link. Azure SQL uses a first-party integration path. See ADR-003 for access path detail and ADR-005 for key management configuration.

### Azure Monitor, Application Insights, and Log Analytics

Application Insights provides APM. The Workload Log Analytics Workspace is the central log sink. AMPLS enforces private-only ingestion for compute-layer telemetry. PaaS services send diagnostics via Azure backbone. The Workload Log Analytics Workspace can be integrated with a central security workspace for SIEM consumption. See ADR-002 for observability decisions.

### Azure Firewall

Azure Firewall is the egress control point in the Connectivity Subscription. It inspects all outbound traffic and enforces FQDN-based rules. All compute subnets route outbound traffic through the firewall via UDRs.

### Azure DNS Private Resolver and Private DNS Zones

DNS Private Resolver and linked Private DNS Zones handle name resolution for all private endpoints. Each service type has a corresponding privatelink.* DNS zone linked to the Spoke VNet. The DNS Private Resolver enables hybrid clients to resolve private endpoint addresses.

### Azure DDoS Protection

Enabled at the VNet level for volumetric attack mitigation.

### Microsoft Defender for Cloud

Enabled for all services, providing continuous security assessment and threat detection.

### Microsoft Fabric

Microsoft Fabric is the enterprise data platform, operating as a SaaS service outside the workload network boundary. Fabric provides Data Factory, Lakehouses (Medallion Architecture), Spark notebooks, and semantic models. The Workspace Identity authenticates to Staging Storage via Trusted Workspace Access. Fabric CMK uses a version-less key from Key Vault accessed via the trusted-services bypass. See ADR-005 for encryption detail.

### Microsoft Purview

Microsoft Purview provides data governance integrated with Fabric, Storage, and Foundry for classification, sensitivity labels, and DLP. Detailed Purview design (label taxonomy, DLP rules, retention policies) is client-specific and treated as a separate workstream. The platform ensures integration points are available. See ADR-005 Section 11 for current scope.

### AI Gateway

The platform does not include an AI Gateway (Azure API Management as AI 
Gateway) between the application layer and Microsoft Foundry. The current 
scope is a single-workload platform where one application consumes one set 
of AI services. The multi-consumer governance problems that an AI Gateway 
addresses, such as cross-application token quotas, centralised policy 
enforcement across independent consumers, and per-team rate limiting, do 
not exist at this scope.

The architecture is designed so that adopting an AI Gateway later is 
additive. The application layer accesses Foundry models through a 
configurable endpoint abstraction. Introducing APIM as an AI Gateway 
would change endpoint configuration, identity assignment, and network 
routing without requiring changes to the processing pipeline or domain 
logic.

See ADR-001 (AI Platform Strategy) for the scaling triggers that would 
justify introducing an AI Gateway and the migration path.

---

## Alternatives

### Compute layer

Azure App Service or AKS can replace Container Apps. App Service is simpler for single-app deployments but less suited to the split runtime model. AKS provides full orchestration control but introduces significant operational overhead. Container Apps was selected as the middle ground.

### Operational database

Azure Cosmos DB can replace Azure SQL for workflow state. Cosmos DB offers flexible schema and global distribution but introduces a different consistency model and higher cost at low throughput. Azure SQL was selected for its relational model and familiarity in enterprise environments.

### Message dispatch

Azure Event Grid can replace Service Bus for event-driven routing. Event Grid is push-based and optimised for event distribution rather than queued work dispatch. Service Bus was selected for its queue semantics, dead-letter handling, and retry behaviour.

### API gateway

Azure API Management can be added in front of the Container Apps Environment for rate limiting, request transformation, and developer portal capabilities. It is not included in the baseline because the current access pattern (private enterprise users, no external API consumers) does not require it.

### AI model hosting

Self-hosted models on AKS or Container Apps can replace Foundry-hosted deployments for models not available in Foundry or scenarios requiring fine-tuned open-source models. This increases operational complexity. Foundry-hosted deployments are the default.

---

## Cross-Cutting Concerns

### Networking

The platform enforces a Private Link baseline with public access disabled on all PaaS services. Where Microsoft service integration patterns require cross-service access, the platform uses three firewall-level mechanisms: resource instance rules (Fabric and AI Search indexer to Staging Storage), shared private link (AI Search to Key Vault for CMK), and trusted-services bypass (Storage, Foundry, and Fabric to Key Vault for CMK). These are scoped firewall rules granting access to specific Microsoft service identities. They do not re-enable public access. Public access remains disabled on every service. Each exception is documented, scoped, and enforced by the storage or Key Vault firewall configuration. Azure Policy enforces EU-region-only deployment and blocks Global Standard Azure model deployments. See ADR-003 for the full network isolation design.

### Identity

All service-to-service authentication uses managed identities. No shared keys, connection strings, or API keys are used for platform service communication. Human access is authenticated through Microsoft Entra ID. Each service has a dedicated managed identity with minimum required RBAC roles. See ADR-004 for the identity model and per-service role assignments.

### Data Protection

Business-critical services use customer-managed keys stored in Key Vault Premium. TLS 1.2+ is enforced on all data-in-transit paths via Azure Policy. See ADR-005 for the CMK inventory, key rotation strategy, and deployment sequencing.

### Observability

Compute-layer telemetry flows through AMPLS in private-only mode to Application Insights and Log Analytics. PaaS services emit diagnostics via Azure backbone. The Workload Log Analytics Workspace can be integrated with a central security workspace for SIEM consumption. See ADR-006 for observability design decisions.

### AI Safety

Microsoft Foundry content safety filters are enabled on all model deployments, covering both input and output. Microsoft Foundry content safety filters are enabled on model deployments. Workflow-level controls such as routing, optional intermediate validation, human review, and PII handling are defined in the relevant use-case architecture. System prompts and workflow policies enforce behavioural boundaries: the AI flags findings, humans decide where required; outputs are expected to retain traceability to sources; and models must not present outputs as compliance determinations unless the use case explicitly defines and governs that behaviour. See ADR-001 for the AI safety framework and human oversight model.

### Data Governance

Microsoft Purview is integrated with Fabric, Storage, and Foundry for classification, sensitivity labels, and DLP. Detailed Purview configuration is client-specific and treated as a separate workstream. See ADR-005 Section 11 for integration scope.

---

## Assumptions and Constraints

### Assumptions

- Enterprise access to the platform is private and controlled by the client's network infrastructure
- Client-specific ingress implementation (VPN, ExpressRoute, AVD, or other) is outside the scope of this document
- An enterprise Azure estate exists around this workload (landing zones, management groups, policy baseline)
- Microsoft Fabric is available and licensed; Trusted Workspace Access requires F64+ capacity
- Use-case architectures inherit this platform baseline and extend it rather than redefining the shared core

### Constraints

- Durable operational state is stored in Azure SQL, not inferred from message queue state
- Service Bus is used for dispatch and coordination, not as a system of record
- Fabric is a consumption and analytics plane, not the runtime execution plane
- Some use cases may operate with persistent-copy, transient-copy, or no-copy ingestion patterns. The platform supports these patterns but does not prescribe one universally.
- Some Azure service integrations require access patterns (trusted-services bypass, shared private links, resource instance rules) that do not map to a pure private-endpoint-only model. These are documented exceptions, not workarounds. All three mechanisms operate over the Microsoft backbone network, not the public internet. Public access remains disabled on every service.
- Purview is an integration point in this architecture, not a fully designed workstream

---

## Deployment Guidance

Infrastructure-as-code is out of scope for this repository. The deployment sequence below documents the architectural constraints that any IaC implementation must respect:

1. Deploy foundational shared controls first (Key Vault, keys, network infrastructure)
2. Deploy identities and RBAC assignments second
3. Deploy CMK-dependent services third
4. Deploy workload-specific use cases on the platform baseline last

This sequencing matters where CMK-enabled services, private access dependencies, and identity relationships cannot be retrofitted after deployment. See ADR-005 for the CMK and encryption decisions that drive this sequencing.

---

## Extending the Platform

New use cases should build on this platform reference architecture when they share the same baseline assumptions for ingress and network posture, runtime model, identity model, data protection model, observability baseline, and Fabric integration pattern.

A new use-case-specific reference architecture is required when a use case introduces different autonomy boundaries, different data sensitivity or regulatory obligations, different service dependencies, different human review requirements, or materially different runtime or retrieval patterns. If a use case changes a platform assumption, the change should be captured in a dedicated ADR or architecture extension rather than silently drifting the platform baseline.

---

## Architecture Decision Records

| ADR     | Title                                            | Status   | Summary                                                                                                                            |
| ------- | ------------------------------------------------ | -------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| ADR-001 | Processing Paradigm                              | Accepted | Defines the typed executor model, workflow ownership boundaries, and human oversight baseline for the platform.                    |
| ADR-002 | Runtime Platform and EU Region Selection         | Accepted | Selects Azure Container Apps, Azure Service Bus, Azure SQL Database, and the primary EU deployment region.                         |
| ADR-003 | Network Isolation and Data Residency Enforcement | Accepted | Defines the Private Link baseline, scoped network exceptions, and Azure Policy region enforcement.                                 |
| ADR-004 | Identity, Authentication, and Authorisation      | Accepted | Defines Zero Trust identity boundaries, managed identities, Microsoft Entra ID usage, and least-privilege RBAC.                    |
| ADR-005 | Data Protection                                  | Accepted | CMK for business-critical services. Key Vault Premium configuration, auto-rotation strategy, TLS enforcement, Purview integration. |
| ADR-006 | Observability and Operational Model              | Planned  | Incident response, auditability, backup and restore, alerting, and operational runbooks. Scope defined; content pending.           |

---

## Use-Case Reference Architectures

Use-case-specific architectures build on this platform baseline. Each follows the same structure (context, diagram, workflow, components, considerations) and references this document for shared infrastructure.

| Use Case                   | Document                                                                             | Description                                                                                                                                                                                                                                                                                               |
| -------------------------- | ------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| UC1: Chat with Data        | Planned                                                                              | Natural language querying across structured data in Fabric and enterprise systems, using RAG with AI Search and Foundry models.                                                                                                                                                                           |
| UC2: Document Intelligence | `use-cases/uc2-document-intelligence/reference-architecture.md` | Extraction, classification, and unification of enterprise documents into governed structured outputs for downstream analytics and retrieval. One example is building a governed view of the contractual relationship with a given client from the underlying document estate and related enterprise data. |
| UC3: Contract Comparison   | Planned                                                                              | Automated delta identification between sent and returned contract versions.                                                                                                                                                                                                                               |
| UC4: Audit Preparation     | Planned                                                                              | Internal audit capability mimicking AI-powered external auditors, using RAG-based knowledge retrieval.                                                                                                                                                                                                    |
