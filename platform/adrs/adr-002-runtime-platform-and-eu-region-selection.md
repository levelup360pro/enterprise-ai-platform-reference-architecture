# ADR-002: Runtime Platform and EU Region Selection

**Status**: Accepted  
**Date**: 09/03/2026  
**Decision Scope**: Front-end hosting, backend runtime, background processing pattern, queueing bias, workflow state store, and primary deployment region for the enterprise AI platform  
**Depends on**: ADR-001 (Processing Paradigm)  

---

## Architecture Principle: Production on GA, Validate on Preview

Production deployments use only GA services and GA API versions. Preview features (Content Understanding Pro Mode, multi-file input, reasoning; Agent Framework extensions; Hosted Agents) are used in the development environment to validate future capabilities. Every architecture option that depends on a preview feature includes a documented GA fallback. When a preview feature reaches GA, it is adopted without redesign because the abstraction layer already accounts for it.

---

## Context

ADR-001 establishes that this platform uses Agent Framework Workflows with typed Executors: deterministic code, AI service calls, and agentic reasoning applied per stage based on the processing requirement. The Workflow engine controls sequencing, quality gates, and human-in-the-loop checkpoints across all Executor types. The runtime architecture still needs an explicit deployment shape that is easy to operate, auditable in a regulated environment, and aligned with the private-networking posture defined across the platform.

This platform provides four capabilities for EU-regulated, document-intensive industries: document intelligence, conversational analytics, contract intelligence, and audit automation. The runtime must expose explicit HTTP APIs for internal callers, run background processing for document batches and review workflows, and keep workflow truth in durable platform services rather than in container memory.

The corrected platform recommendation is to treat Azure Container Apps as the primary application boundary. The frontend UI, FastAPI backend, and long-running worker services are part of the same regulated application surface. Queue dispatch uses Azure Service Bus, and workflow state is stored in Azure SQL Database by default. PostgreSQL Flexible Server remains a valid exception when the solution has a deliberate JSONB-heavy or portability-driven reason.

Sweden Central remains the reference region for this repository because it combines broad GA service availability in the EU with access to the preview Content Understanding capabilities used for development validation. Production deployments remain portable to other EU regions where the required GA services are available.

---

## Decision Drivers

- **Explicit application boundary**: The frontend, backend API, and background workers are part of one regulated internal application surface with shared identity, networking, diagnostics, and deployment controls.
- **Deterministic orchestration without hidden runtime magic**: The platform needs explicit request/response APIs, explicit workflow state, queue-driven background execution, and auditable job status.
- **EU data residency with private networking**: All data at rest and in transit must remain within the EU. Internal ingress, private endpoints, private DNS, and managed identities are required.
- **Operational symmetry**: The hosting model should minimise special cases between frontend, backend, and worker services.
- **Enterprise messaging semantics**: Dead-lettering, duplicate detection, RBAC, private endpoints, and predictable delivery controls are preferred over the most lightweight queue option.
- **State clarity**: Queue state and database state must be the workflow truth, not runtime-local memory or opaque platform checkpoints.
- **Production maturity**: GA-level platform components first. Preview-only compute models remain migration targets rather than production dependencies.
- **Migration path**: The processing model must remain portable to Hosted Agents or other managed runtimes when those options meet the platform's private-networking and governance requirements.

---

## Considered Alternatives

### Frontend Hosting

#### Option A: Azure Static Web Apps

Static Web Apps supports private endpoints, Microsoft Entra ID authentication, and route-based authorization. It remains a valid option for genuinely static sites with a simple boundary. It is rejected for this platform because the UI is part of the same regulated application boundary as the API and background services. Using a different hosting model for the UI adds another ingress, another deployment model, and another operational special case without a compensating benefit.

#### Option B: Frontend container on Azure Container Apps (SELECTED)

The frontend runs as its own container app in the same internal Container Apps environment as the API and worker services. This gives the platform one compute boundary, one networking model, one identity model, one logging surface, and one revision strategy across the user-facing and processing components.

### Backend and Background Runtime

#### Option A: Azure Durable Functions on Flex Consumption

Durable Functions remains a viable platform for event-driven or orchestration-heavy workloads. Private networking is supported on the right hosting plan, and the Functions programming model is still defensible when the team explicitly wants it. The Azure Storage backend for Durable Functions supports private endpoints and remains a valid option where Durable orchestration semantics are required within a private-networked design. It is no longer the preferred choice here because this platform benefits more from an explicit application service boundary than from a function-first hosting model. The architectural preference is for a web-queue-worker pattern with a FastAPI API boundary, Azure Service Bus for dispatch, and auditable workflow state in a relational database rather than in framework-managed orchestration storage. The corrected recommendation is not that Functions is incapable; it is that Functions is less aligned with the operational shape the platform now needs.

#### Option B: FastAPI API plus worker services on Azure Container Apps (SELECTED)

The backend runs as a FastAPI container app. Background execution runs as a separate worker container app reading from Azure Service Bus. Azure Container Apps jobs are used only where a workload is clearly finite-duration and isolated per execution. This option keeps the HTTP surface explicit, separates API and worker responsibilities cleanly, supports internal-only ingress, and aligns naturally with the web-queue-worker pattern Microsoft continues to recommend for this class of workload.

#### Option C: Azure Functions on Azure Container Apps

Functions hosted on Container Apps narrows some of the gap between the two models, but it still preserves the function runtime abstraction while adding container operational overhead. For this platform, if the team is going to accept the operational model of containers, it is cleaner to run the API and worker code directly rather than layering the Functions runtime on top.

#### Option D: Foundry Agent Service Hosted Agents (preview)

Hosted Agents remains the most interesting managed-compute migration target because it can run the same Agent Framework code in containers. It stays rejected for production because private networking support is not yet available at the required maturity level.

#### Option E: Durable Functions with Durable Task Scheduler

Durable Task Scheduler is the managed backend for Durable Functions, replacing Azure Storage with a purpose-built scheduler service that communicates with the application over gRPC. It offers better performance characteristics, a built-in dashboard, and removes the need for a separate storage account for orchestration state. It is rejected for this platform because, as validated in March 2026, the service does not provide the private-networking posture required by this architecture. There is no private endpoint support, no private-only inbound connectivity model, and no documented private DNS pattern equivalent to the services selected elsewhere in the platform. The scheduler endpoint resolves publicly. In a platform where internal-only networking and private connectivity are explicit design principles, this is a blocking constraint. TLS, Microsoft Entra authentication, and RBAC are useful controls, but they do not satisfy the platform’s requirement for private-only service exposure. If private endpoint support reaches GA, this option should be re-evaluated against the selected Container Apps worker pattern.

### Queueing and Workflow State

#### Option A: Azure Storage Queues plus storage-backed state

Storage Queues is simpler and cheaper, but the platform gives up richer delivery semantics, dead-lettering, duplicate detection, sessions, and the stronger enterprise governance posture that Service Bus provides.

#### Option B: Azure Service Bus plus Azure SQL Database (SELECTED)

Service Bus handles work dispatch. Azure SQL Database stores workflow state, job executions, message lineage, retries, and audit-oriented operational data. This keeps dispatch and state explicit, queryable, and independently observable.

#### Option C: Azure Service Bus plus PostgreSQL Flexible Server

PostgreSQL Flexible Server is still a valid alternative when the platform intentionally models agent or workflow payloads as JSONB-first entities or when portability outside Azure is an explicit design driver. It is not the default recommendation for this repository.

### EU Region Selection

#### Sweden Central (SELECTED)

Broad GA service and model availability across EU regions. Frequently receives early EU availability for new AI models and capabilities. Azure Container Apps workload profiles, Azure Service Bus Premium, Azure SQL Database, AI Search, Content Understanding, Document Intelligence, Foundry Projects, and Fabric are all available. Sweden Central is also the EU region used for validating future Content Understanding capabilities during development.

#### North Europe (Ireland)

Viable production region if a client landing zone requires it. The trade-off is narrower AI model availability and no access to the Content Understanding preview capabilities used for development validation. As of March 2026, Azure AI Search in North Europe is flagged by Microsoft as capacity-constrained, which may prevent creation of new search services.

#### West Europe (Netherlands)

Also viable for production with the same trade-off profile as North Europe. As of March 2026, Azure Container Apps provisioning in West Europe is experiencing ongoing capacity failures. Validate service and capacity availability per engagement.

---

## Decision

**Frontend**: A dedicated frontend container on Azure Container Apps. The frontend shares the same internal Container Apps environment as the backend unless there is a later blast-radius reason to split environments.

**Backend API**: FastAPI on Azure Container Apps with managed identity, internal ingress by default, and private access to downstream services.

**Background processing**: Azure Service Bus for work dispatch. A worker container app consumes queue messages continuously. Azure Container Apps jobs are introduced only for isolated finite-duration tasks where a per-execution boundary adds value.

**Workflow state store**: Azure SQL Database is the default operational state store for jobs, executions, retries, approvals, and audit-oriented workflow history. PostgreSQL Flexible Server remains the exception path when there is an explicit JSONB-heavy or portability-driven reason.

**Region**: Sweden Central as the reference architecture region. Production deployments stay portable to other EU regions where the required GA services are available.

**Model deployments**: DataZone Standard EU as the default deployment type. Standard (regional) deployments remain appropriate where lower and more predictable latency is required.

**Data residency enforcement**: Azure Policy denies Global Standard model deployments. Only DataZone Standard and Standard (regional) are permitted.

**ACA baseline**: Use an internal Azure Container Apps environment with workload profiles rather than a legacy consumption-only environment. This keeps private endpoints, network control, and enterprise traffic management aligned with the platform's security posture.

---

## Data Ingestion Boundary

The pipeline processes documents from Azure Blob Storage. Content Understanding and Document Intelligence accept input via URL (blob storage with managed identity or SAS) or inline base64 content. Neither service connects to on-premises storage directly. Documents must land in Azure before the pipeline can process them. For clients with on-premises document repositories, ingestion options are evaluated per engagement:

**Fabric Data Factory with On-Premises Data Gateway**: Recommended for continuous ingestion. Supports File, Folder, SharePoint, SQL Server, SFTP, and other on-prem sources via a gateway agent installed on the client's network. Copies documents into a Fabric Lakehouse or Blob Storage on a schedule or event trigger.

**AzCopy or Azure Storage Explorer over ExpressRoute/VPN**: Suitable for one-off historical migration or low-frequency batch transfers. No gateway agent required. No built-in scheduling.

**Custom automation (scripts, Logic Apps, Power Automate)**: For clients with specific workflow requirements around document approval or staging before upload.

The choice depends on the client's existing connectivity, the volume and frequency of new documents, and whether they need an auditable, scheduled ingestion pipeline or a one-time migration. This is scoped during engagement discovery, not pre-decided in the reference architecture.

---

## Consequences

### Positive

- **One coherent application boundary**: Frontend, API, and workers share the same compute platform, identity model, logging surface, and private-networking posture.
- **Explicit operational model**: FastAPI exposes the application boundary clearly. Service Bus and Azure SQL make queue state and workflow state queryable and auditable.
- **Enterprise-ready messaging**: Service Bus provides dead-lettering, duplicate detection, sessions, RBAC, and private endpoints without additional middleware.
- **Cleaner background processing story**: A continuously running worker app is the default. ACA jobs are available when a task is clearly finite and isolated.
- **SQL-first operational data model**: Azure SQL aligns well with jobs, executions, retries, approvals, audit trails, and reporting. PostgreSQL remains available by exception rather than by default.
- **Private-network-ready compute**: Container Apps environments support internal ingress, managed identity, private connectivity to downstream services, private DNS, and common traffic management patterns.
- **Migration headroom**: Agent Framework Workflows remain portable. The workflow graph stays decoupled from the hosting substrate even though the runtime bias is now Container Apps rather than Durable Functions.

### Negative

- **More explicit operations**: The platform team must own container builds, revisions, health probes, and worker lifecycle management directly.
- **API and worker split is deliberate complexity**: The separation improves clarity, but it means more deployable units than a single monolithic process.
- **SQL schema ownership becomes a design responsibility**: The platform team must model workflow state intentionally rather than delegating that concern to a framework-specific backend.
- **Container Apps jobs are not the default for every workload**: Deciding between a continuously running worker and a job requires discipline and a clear workload-shape decision.
- **Sweden Central is still a reference choice, not a universal default**: Client landing-zone constraints may still drive production deployments to another EU region.

---

## Risks and Mitigations

- **Container Apps workload sprawl**: Frontend, API, worker, and optional jobs can grow into too many independently managed services. **Mitigation**: keep the split intentional and limited to genuine trust or workload boundaries; share one internal environment and one observability model.
- **Worker backlog growth**: A queue-based design can hide failures if backlog, dead-letter volume, or retry storms are not monitored. **Mitigation**: instrument Service Bus queue depth, dead-letter counts, processing latency, and retry rates in Application Insights and Azure Monitor alerts.
- **Database coupling**: If workflow state is allowed to drift into ungoverned semi-structured payloads, the SQL-first bias loses its value. **Mitigation**: keep the operational schema relational by default; use PostgreSQL only when a conscious JSONB-first decision is documented.
- **Preview migration targets remain immature**: Hosted Agents and other managed runtimes may eventually become simpler than Container Apps. **Mitigation**: keep executor logic decoupled from hosting and review migration triggers quarterly.
- **Single-region deployment**: A regional outage in the selected region delays processing. **Mitigation**: this is acceptable for the current batch-oriented platform scope; revisit active-passive multi-region only if client SLAs require it.

---

## Service Availability Matrix (Sweden Central)

This matrix documents service availability for the reference architecture region as of March 2026. If a client engagement deploys to a different EU region, validate service availability against the relevant Microsoft Learn service documentation.

|Service|Status|Deployment Type|Notes|
|---|---|---|---|
|Azure Container Apps workload profiles|GA|Internal environment|Required baseline for the regulated application boundary and network controls.|
|Azure Service Bus Premium|GA|Private endpoint capable|Preferred enterprise queueing primitive for regulated workflows.|
|Azure SQL Database|GA|Single database / elastic pool|Default operational state store for workflow state and audit-oriented execution history.|
|Azure Database for PostgreSQL Flexible Server|GA|Optional exception path|Use only when JSONB-heavy modelling or portability requirements justify it.|
|Content Understanding (GA)|GA (2025-11-01 API)|N/A|Full GA feature set.|
|Content Understanding (Preview)|Preview (2025-05-01-preview)|N/A|Used only for development validation. Sweden Central remains the EU preview region of interest.|
|Document Intelligence v4.0|GA|N/A|Prebuilt and custom extraction capabilities.|
|Foundry Models (GPT-4o / GPT-4.1 / embeddings)|GA|DataZone Standard, Standard|Deployment selection governed by data-residency posture.|
|Azure AI Search|GA|N/A|Hybrid retrieval, semantic ranker, and indexing.|
|Foundry Projects|GA|N/A|Required for managed model and agent integration.|
|Fabric (all workloads)|GA|N/A|Lakehouse, Spark, Data Factory, and downstream analytics.|
|Entra Agent ID|Preview|N/A|Watch item only; not a production dependency.|
|Key Vault|GA|N/A|Secrets and CMK integration.|
|Azure Policy|GA|N/A|Data residency enforcement.|
|Application Insights|GA|N/A|OpenTelemetry-compatible observability backend.|

---

## Additional Data Residency Controls

All Blob Storage accounts use private endpoints with public access disabled. Azure Service Bus, Azure SQL Database, AI Search, and Key Vault are reached over private connectivity. The Container Apps environment is internal by default, with private DNS treated as a first-class design concern rather than an implementation afterthought. Application Insights and the Log Analytics workspace remain in the selected EU region.

---

## Data Residency Enforcement

Azure Policy to block Global Standard model deployments:

```json
{
    "if": {
        "field": "Microsoft.CognitiveServices/accounts/deployments/sku.name",
        "equals": "GlobalStandard"
    },
    "then": {
        "effect": "Deny"
    }
}
```

This policy is illustrative; test against current Azure Policy schema before deployment. It forces DataZone Standard or Standard (regional) deployments only. DataZone Standard EU ensures inferencing data stays within the EU data zone.

---

## Follow-ups

**Spike: Container Apps runtime baseline** Validate startup latency, scaling behaviour, and managed-identity access for the frontend, FastAPI API, and worker container apps in an internal workload-profiles environment.

**Spike: Service Bus worker pattern** Validate queue consumption, dead-letter handling, duplicate detection, and idempotent retry behaviour for the document-processing worker.

**Spike: Workflow state schema** Model the Azure SQL workflow schema for jobs, executions, approvals, retries, and audit lineage. Confirm the reporting queries needed by operations and audit stakeholders.

**Spike: PostgreSQL exception criteria** Document the conditions that would justify choosing PostgreSQL Flexible Server instead of Azure SQL Database for a client deployment.

**IaC templates** Define Azure Container Apps environment resources, internal ingress, private endpoints, Service Bus namespace and queues, Azure SQL Database, VNet and DNS zones, AI services, Key Vault, Application Insights, and Azure Policy assignments.

**Observability setup** Configure OpenTelemetry instrumentation, Container Apps diagnostics, queue metrics, SQL workflow-state telemetry, and end-to-end correlation IDs across API, worker, and downstream AI service calls.

**Migration trigger criteria** Review Hosted Agents and other managed runtime options quarterly. Migration is evaluated only when GA status and private-networking support meet the platform's security posture.