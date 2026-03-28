# ADR-003: Network Isolation and Data Residency Enforcement

**Status**: Accepted  
**Date**: 10/03/2026  
**Decision Scope**: How all platform services communicate over private networks, how data flows between the processing layer and Microsoft Fabric, and how EU data residency is enforced at the network layer  
**Depends on**: ADR-001 (Processing Paradigm), ADR-002 (Runtime Platform and EU Region Selection)  
**Depended on by**: ADR-005 (Data Protection), ADR-006 (Identity, Authentication, and Authorisation), platform physical diagram, all use-case ADRs  

---
## Context

ADR-001 selected the processing paradigm (unified workflow with typed executors). ADR-002 selected the compute substrate (Azure Container Apps for the frontend, FastAPI API, and worker services), the reference EU region (Sweden Central), and the model deployment strategy (DataZone Standard EU by default). This ADR defines how all platform services communicate, how network boundaries are enforced, and how data residency controls are applied at the network layer.

The platform comprises services from two distinct ecosystems with fundamentally different networking models.

Azure PaaS services (Foundry, AI Search, Storage, Key Vault, Application Insights, etc.) follow the standard Azure private endpoint model: create a private endpoint per resource in the platform VNet, disable public access on the resource, configure a Private DNS Zone for name resolution. Each resource is independently addressable and independently secured.

Microsoft Fabric is a SaaS platform with its own networking model. Private link configuration operates at the tenant level or workspace level, not per-resource. Enabling private link on Fabric has broad consequences: it affects which Fabric features are available, how Spark compute is provisioned, whether on-premises data gateways function, and how OneLake endpoints resolve. The Fabric networking model is fundamentally different from standard Azure PaaS and requires a distinct architectural approach.

The processing layer is the orchestration layer. It calls Foundry services for AI inference and extraction, writes results to a staging store, and the extraction output must ultimately reach a Fabric Lakehouse for the relationship-oriented analytics view, AI Search indexing, Power BI semantic models, and downstream system consumption.

The critical architectural question is the direction of data flow into Fabric: does the platform push data into Fabric (requiring inbound private access to Fabric), or does Fabric pull data from the platform (using Fabric's own outbound connectivity to reach a staging store)?

---

## Decision Drivers

EU data residency: The platform is designed so that data is stored and processed in approved EU regions, with private connectivity over Microsoft-managed networks and region-locked service selection. Residency and processing controls should be aligned with Microsoft's service-specific data residency commitments and the client's regulatory interpretation. Minimal public exposure: All platform-controlled Azure services disable general public network access where supported and use private endpoints for VNet-connected access. Fabric access to staging storage uses Trusted Workspace Access through a resource instance rule rather than a Fabric-side private endpoint. Fabric feature preservation: The networking model must not restrict Fabric features needed for analytics, reporting, Copilot, or future capabilities. Operational simplicity: Minimise the number of networking primitives, additional infrastructure components, and Fabric-specific configuration while meeting security requirements. Client portability: The architecture must work across different client environments. Some clients will have existing Fabric tenants with their own private link policies. The architecture should impose minimal requirements on the client's Fabric tenant configuration. Standard patterns: Prefer networking patterns that every enterprise network team already understands over Fabric-specific networking constructs. Zero-copy where possible: Prefer data virtualisation (shortcuts, references) over data movement (copy activities, ETL) when the security and governance requirements can be met.

---

## Considered Alternatives: Fabric Data Flow Direction

Option A: Platform Pushes to Fabric (Workspace-Level Private Link) The processing layer writes extraction results directly to the Fabric Lakehouse via the OneLake DFS API (ADLS Gen2 compatible, Entra ID authenticated). This requires inbound private access to Fabric, achieved through workspace-level private link: a private link service is created for the workspace, and a private endpoint in the platform VNet connects to it.

Consequences: The processing layer reaches OneLake via the workspace-specific FQDN ({workspaceid}.z{xy}.dfs.fabric.microsoft.com), which resolves to a private IP through the workspace private endpoint.

Constraints introduced: Only supported on F SKU capacity, not premium (P SKU) or trial. Workspace FQDN must be used for all API calls (not the global onelake.dfs.fabric.microsoft.com endpoint). Default semantic models are not supported; lakehouses must be created after the workspace is configured to block public access. Deployment pipelines cannot connect to the workspace. Item sharing via shared links is not supported. OneLake Security is not currently supported. Workspace monitoring is not currently supported. The Fabric portal UI does not support enabling both inbound protection and outbound access protection simultaneously; the Network Communication Policy API must be used. One private link service per workspace (up to 100 private endpoints per workspace). DNS management requires workspace-specific Private DNS Zones.

Assessment: Introduces significant Fabric feature restrictions and operational complexity for a single benefit: direct writes from the processing layer to the Lakehouse. The workspace becomes a restricted environment where standard Fabric capabilities (semantic models, deployment pipelines, monitoring, item sharing) are constrained. For a platform targeting regulated environments where clients also need those Fabric capabilities for analytics and reporting, these restrictions are difficult to justify.

Option B: Fabric Pulls via Managed Private Endpoint The processing layer writes extraction results to a dedicated staging ADLS Gen2 Storage account with public access disabled and a private endpoint in the platform VNet. Fabric creates a managed private endpoint from the workspace to the staging Storage account and pulls data into the Lakehouse using a Spark Notebook.

Assessment: Managed private endpoints are GA for Fabric Data Engineering workloads (Notebooks, Lakehouses, Spark job definitions) and Eventstream. More critically, OneLake shortcuts do not support connections to ADLS Gen2 via managed private endpoints (documented limitation, active community feature request as of December 2025). This eliminates the zero-copy path. Every extraction result must be physically copied from the staging store into the Lakehouse. This is the strongest reason to reject this option for the current architecture.

Option C: Fabric Pulls via VNet Data Gateway A VNet Data Gateway is deployed in the platform VNet (or a peered VNet). The gateway has network connectivity to the dedicated staging ADLS Gen2 Storage account via private endpoint. Fabric Data Pipeline Copy Activity routes through the VNet Data Gateway to read from the staging Storage account and write to the Lakehouse.

Assessment: Viable. VNet Data Gateway support for Fabric Pipeline Copy Activity, Dataflow Gen2, and Copy Job is GA (September 2025). Supports a long list of data sources. The gateway is Microsoft-managed container infrastructure deployed in the customer's Azure subscription. However, it adds infrastructure (gateway nodes in the VNet), has vertical scale limits per node (~2 vCores, 8 GB RAM), incurs consumption costs billed to Fabric capacity, and still requires data movement (no zero-copy shortcut path).

Option D: Fabric Pulls via Trusted Workspace Access (SELECTED) The Fabric workspace gets a workspace identity (a Fabric-managed identity associated with an F SKU workspace). A resource instance rule on the dedicated staging ADLS Gen2 Storage account grants a network exception for the specific Fabric workspace, identified by workspace GUID and tenant ID. Microsoft documents that when public network access is set to Disabled, previously configured resource instance rules and exceptions remain effective by design. Fabric reaches the storage account through Microsoft-managed connectivity authorised by the resource instance rule, without requiring customer-managed VNet infrastructure on the Fabric side. Resource instance rules are the recommended mechanism over the broader 'Allow trusted Microsoft services' exception, which grants access to all qualifying Azure services across the tenant rather than a specific resource.

OneLake shortcuts in the Fabric Lakehouse reference the dedicated staging Storage account containers directly using workspace identity authentication. Fabric treats the shortcut as a folder in OneLake. Fabric workloads that consume those shortcuts access the data through standard Fabric item semantics: Spark reads the shortcut as a folder in OneLake, SQL analytics endpoint queries shortcut tables, pipelines and Dataflows Gen2 read from or write to storage through shortcuts, semantic models reference shortcut tables, and KQL databases can create shortcuts to the same storage. No data movement required for read access.

For Gold-layer transformation (joining extraction results with CRM data, applying business logic, building semantic models), a Spark Notebook or Data Pipeline reads from the shortcut and writes to native Lakehouse Delta tables.

Assessment: Simplest option. No additional networking infrastructure on the Fabric side (no managed VNet, no managed private endpoint, no VNet Data Gateway). Zero-copy access via OneLake shortcuts is GA today via Trusted Workspace Access. No Fabric private link configuration required. No workspace FQDN complexity. No Fabric-specific DNS zones. Coexists with the private endpoint in the platform VNet that the processing layer uses to write to the same dedicated staging Storage account. Both are independent access mechanisms on the ADLS Gen2 networking stack.

Trusted Workspace Access is explicitly described as the "good default choice for Azure Data Lake Gen2 with network restrictions" in Microsoft's field guidance on Fabric core patterns for accessing private data sources. It is GA, documented for regulated industries, and purpose-built for this exact scenario.

---

## Decision

Option D: Fabric pulls from a dedicated staging ADLS Gen2 Storage account via Trusted Workspace Access with OneLake shortcuts as the primary ingestion mechanism.

The processing layer writes extraction results (structured JSON, Parquet, or Delta format) to a dedicated staging ADLS Gen2 Storage account with public access disabled and a private endpoint in the platform VNet. The Fabric workspace identity is granted read access to the staging Storage account via RBAC, and a resource instance rule on the Storage account's firewall allows the Fabric workspace through the firewall. OneLake shortcuts in the Fabric Lakehouse provide zero-copy access to the staging data.

Why Trusted Workspace Access over Managed Private Endpoints Both achieve the same goal (Fabric accessing a private Storage account without requiring inbound private link to Fabric), but Trusted Workspace Access is superior for this architecture:

OneLake shortcuts are supported via Trusted Workspace Access (GA). They are not supported via managed private endpoints (documented limitation). This is the decisive factor: TWA enables zero-copy access today; MPE requires data to be physically copied into the Lakehouse.

TWA requires no additional Fabric-side networking infrastructure. No managed VNet creation, no managed private endpoint provisioning, no endpoint approval workflow (the resource instance rule is deployed via IaC on the Storage account side). MPE requires creating and approving a managed private endpoint from within the Fabric workspace.

TWA supports shortcuts, pipelines, semantic models, T-SQL COPY, and AzCopy for firewall-enabled ADLS Gen2 access. Fabric workloads then consume that data through OneLake shortcuts using standard Fabric item semantics. MPE supports Spark workloads; broader Fabric workload support through MPE is not documented at the same level.

TWA has no additional cost. MPE and VNet Data Gateway incur Fabric capacity consumption.

Why Azure Storage (ADLS Gen2) rather than Azure SQL Database as the staging store The processing layer extraction pipeline produces structured JSON output from Content Understanding and Document Intelligence. Writing JSON or Parquet files to a Storage account is operationally simpler than writing to SQL Database tables: no connection pooling, no schema migrations, no SQL driver dependencies, no database engine overhead. The Storage account is a file ingestion store, not a queryable data store. If the extraction output needs to be queryable by non-Fabric consumers before it reaches the Lakehouse, Azure SQL Database can be substituted (also a supported target for both TWA via shortcut and MPE). For this platform, the only consumer of staging data is Fabric, so Storage is simpler and cheaper.

Data flow architecture Write path (Platform to Staging): The processing layer writes extraction results to the dedicated staging ADLS Gen2 Storage account via private endpoint in the platform VNet. Standard Azure private endpoint pattern. DNS resolves .dfs.core.windows.net to a private IP via the privatelink.dfs.core.windows.net Private DNS Zone.

Read path (Fabric from Staging): OneLake shortcuts in the Fabric Lakehouse reference the staging Storage account containers. The Fabric workspace identity authenticates to the Storage account. The resource instance rule on the Storage account allows the Fabric workspace through the firewall. Fabric reaches the storage account through Microsoft-managed connectivity authorised by the resource instance rule, without requiring customer-managed VNet infrastructure on the Fabric side. Fabric workloads consume the shortcut data through OneLake; the TWA mechanism enables the shortcut connection, while each Fabric workload (Spark, SQL analytics endpoint, pipelines, Dataflows Gen2, semantic models, KQL database) reads through OneLake using standard item semantics.

Indexer read path (AI Search from Staging): The Azure AI Search indexer reads extraction results from the dedicated staging Storage account for indexing into AI Search. AI Search and the staging Storage account are both in Sweden Central (ADR-002). Same-region Azure services use private IPs that are not addressable via IP network rules, making IP firewall rules ineffective for this connection. A resource instance rule on the staging Storage account references the AI Search service's resource ID, granting the AI Search system-assigned managed identity access through the firewall. The trusted service exception is not enabled; the resource instance rule provides per-resource scoping. The AI Search system-assigned managed identity authenticates to Storage via Entra ID with Storage Blob Data Reader (RBAC defined in ADR-004). The indexer data source connection string uses the ResourceId= format, which signals AI Search to use its system-assigned managed identity. This is the only identity type supported for same-region indexer access to firewall-protected storage via resource instance rule. The resource instance rule is deployed via IaC on the staging Storage account alongside the Fabric TWA rule.

Transformation path (Staging to Gold): A Spark Notebook or Data Pipeline reads from the shortcut (Bronze/Silver), applies transformations (entity resolution against CRM master data, confidence threshold validation, business aggregation), and writes to native Lakehouse Delta tables (Gold). The Gold layer is the business-ready, query-optimised layer consumed by semantic models, Power BI, AI Search indexing, and downstream systems.

The dedicated staging Storage account has three independent access mechanisms operating simultaneously with public access disabled: the DFS private endpoint serves the processing layer (write path); the Fabric workspace resource instance rule serves Fabric (read path via TWA); the AI Search resource instance rule serves the indexer (read path for index population). No conflict between these mechanisms. Each is independently scoped: the private endpoint to the VNet, each resource instance rule to a specific service resource ID.

Trusted Workspace Access implementation requirements Workspace identity: Enable on the Fabric workspace (requires F SKU capacity in an EU region). RBAC: Grant the workspace identity Storage Blob Data Reader role on the specific staging containers, not account scope. Follows least-privilege principle. If write-back scenarios emerge, elevate to Storage Blob Data Contributor on the required containers only. Resource instance rule: Deploy via IaC on the dedicated staging Storage account, specifying the Fabric workspace ID and tenant ID. The subscription ID in the rule uses the fixed value 00000000-0000-0000-0000-000000000000 (this is correct per Microsoft documentation; it is a Fabric-specific convention, not an error). Deploy the rule in the same IaC template as the Storage account; it takes effect regardless of the publicNetworkAccess value at deployment time. Do not enable 'Allow trusted Microsoft services to access this storage account'; the resource instance rule provides per-workspace scoping that the trusted services checkbox does not. OneLake shortcuts: Create in the Fabric Lakehouse pointing to the staging Storage account containers. Use the DFS URL (https://.dfs.core.windows.net/) and workspace identity authentication. Validation: Confirm Spark Notebook can read through the shortcut (Spark reads the shortcut as a folder in OneLake; it does not connect directly to ADLS Gen2 via TWA). Confirm SQL analytics endpoint can query shortcut tables. Confirm Pipeline Copy Activity can read from the ADLS Gen2 connection with workspace identity authentication. Confirm semantic models can reference shortcut tables.

AI Search indexer access to staging storage Azure AI Search and the dedicated staging Storage account are both deployed in Sweden Central. When a search service and a storage account are in the same region, IP firewall rules are ineffective because traffic uses private IPs that are not included in the storage account's IP rule evaluation. For same-region AI Search indexer access to firewall-protected Storage, Microsoft supports system-assigned managed identity with either the trusted service exception or a resource instance rule. This architecture selects the resource instance rule to preserve per-resource network scoping.

Trusted service exception (rejected): Enabling "Allow trusted Microsoft services to access this storage account" on the dedicated staging Storage account would admit the AI Search system-assigned managed identity as a trusted service. This admits all Azure trusted services, not just AI Search. Any Azure service in the tenant with a system-assigned identity and appropriate RBAC could access the storage account through this exception.

Resource instance rule (selected): A resource instance rule on the dedicated staging Storage account explicitly references the AI Search service's resource ID. Only the specified AI Search instance is admitted through the firewall. This is consistent with the Fabric TWA resource instance rule pattern already used on this storage account.

Implementation:

System-assigned managed identity on AI Search: Required. This is the only identity type supported for same-region indexer access to firewall-protected storage via resource instance rule. User-assigned identities do not work with same-region firewall bypass mechanisms. The identity is created by enabling the system identity on the AI Search service.

Resource instance rule on staging Storage account: Deployed via IaC in the same networkAcls.resourceAccessRules array as the Fabric TWA rule. The rule specifies the AI Search service resource ID and tenant ID:

```json
{
  "networkAcls": {
    "resourceAccessRules": [
      {
        "tenantId": "{tenant-id}",
        "resourceId": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/{rg}/providers/Microsoft.Fabric/workspaces/{workspace-id}"
      },
      {
        "tenantId": "{tenant-id}",
        "resourceId": "/subscriptions/{subscription-id}/resourceGroups/{rg}/providers/Microsoft.Search/searchServices/{search-service-name}"
      }
    ],
    "bypass": "None",
    "virtualNetworkRules": [],
    "ipRules": [],
    "defaultAction": "Deny"
  }
}
```

The Fabric workspace rule uses the fixed subscription ID 00000000-0000-0000-0000-000000000000 (Fabric-specific convention, documented in the TWA implementation requirements above). The AI Search rule uses the actual subscription ID of the search service. bypass is set to None; the trusted service exception is not enabled.

RBAC: The AI Search system-assigned managed identity requires Storage Blob Data Reader on the dedicated staging Storage account. RBAC assignments are defined in ADR-004.

Indexer data source connection string: Uses the ResourceId= format:

ResourceId=/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/{name};

This signals AI Search to use its system-assigned managed identity for authentication. Combined with disableLocalAuth: true on AI Search and allowSharedKeyAccess: false on Storage (ADR-004), there is no fallback to key-based authentication.

Validation: Confirm indexer can connect to the dedicated staging Storage account and successfully index documents. Confirm that removing the resource instance rule causes the indexer to fail (negative test validates the rule is effective).

Trigger mechanism The processing layer orchestration writes extraction results to a known path in the dedicated staging Storage account (partitioned by batch ID and timestamp). Upon batch completion, the orchestration can trigger the Fabric transformation step via the Fabric REST API (Items - Run On Demand Job) or via an event-driven mechanism (Storage blob event triggering a Fabric Pipeline). The specific trigger is an implementation decision, not an architectural one. For the shortcut-based read path, no trigger is needed; data is accessible immediately after the processing layer writes it.

---

## Platform Networking Specification

The following decisions are pattern application based on the services selected in ADR-001 and ADR-002. They are documented here for completeness and to support the platform physical diagram.

VNet Topology Single VNet in the selected EU region (Sweden Central as reference). Three subnets:

Compute subnet: Delegated to Microsoft.App/environments for the internal Azure Container Apps environment. The frontend, FastAPI API, worker app, and optional jobs use this subnet for outbound access to private endpoints. A /24 is recommended as the starting point for workload-profile scale headroom; validate sizing per client throughput requirements.

Private endpoint subnet: Hosts all private endpoints for platform services. No delegation required. Size based on the number of endpoints (minimum /27).

Reserved subnet: Available for future expansion (jump box for debugging, VPN gateway for on-premises connectivity, additional compute). Not provisioned initially.

No hub-and-spoke topology required for the platform itself. If the client has an existing hub-and-spoke network with centralised firewall/DNS, the platform VNet peers to the hub. The ADR documents what the platform needs; the client's network team integrates it.

Private Endpoint Inventory

| Service                                   | Resource Type                            | Private Endpoint Sub-Resource | Private DNS Zone                                                                                                                                     | Notes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ----------------------------------------- | ---------------------------------------- | ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Foundry (AI services)                     | Microsoft.CognitiveServices/accounts     | account                       | privatelink.cognitiveservices.azure.com                                                                                                              | The primary AI account endpoint is secured through this private endpoint. It covers Content Understanding, Document Intelligence, Foundry Models, and Content Safety for the processing layer.<br>Foundry Agent Service may require additional networking configuration beyond the standard Cognitive Services private endpoint. The specific requirements depend on the agent hosting model selected and are assessed if agent-service capabilities are adopted. This does not affect the current platform networking specification. |
| Azure AI Search                           | Microsoft.Search/searchServices          | searchService                 | privatelink.search.windows.net                                                                                                                       |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Azure Storage (Blob) - Document Ingestion | Microsoft.Storage/storageAccounts        | blob                          | privatelink.blob.core.windows.net                                                                                                                    | Source documents are ingested here via blob endpoint                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Azure Storage (DFS) - Staging             | Microsoft.Storage/storageAccounts        | dfs                           | privatelink.dfs.core.windows.net                                                                                                                     | Extraction results staging; Fabric reads from here via Trusted Workspace Access, not via this private endpoint. TWA and the private endpoint are independent access paths on the same Storage account                                                                                                                                                                                                                                                                                                                                 |
| Azure Service Bus                         | Microsoft.ServiceBus/namespaces          | namespace                     | privatelink.servicebus.windows.net                                                                                                                   | Queue dispatch for worker processing and human-review workflows                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Azure SQL Database                        | Microsoft.Sql/servers                    | sqlServer                     | privatelink.database.windows.net                                                                                                                     | Default workflow state store for jobs, executions, retries, and approvals                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Azure Key Vault                           | Microsoft.KeyVault/vaults                | vault                         | privatelink.vaultcore.azure.net                                                                                                                      | Secrets and CMK for compute-layer services. AI Search CMK access uses a separate shared private link (see note below).                                                                                                                                                                                                                                                                                                                                                                                                                |
| Application Insights / Log Analytics      | Azure Monitor Private Link Scope (AMPLS) | Multiple (azuremonitor)       | privatelink.monitor.azure.com, privatelink.oms.opinsights.azure.com, privatelink.ods.opinsights.azure.com, privatelink.agentsvc.azure-automation.net | AMPLS with ingestion and query access modes set to Private Only                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |

Note on Cosmos DB: Azure Cosmos DB is not included in this inventory. ADR-002 now places workflow state in Azure SQL Database and work dispatch in Azure Service Bus, so there is no platform requirement for Cosmos DB in the current architecture. If the platform later adopts a managed agent runtime that requires Cosmos DB as a backing service, it should be added to this inventory explicitly at that time.

Note on Fabric: Fabric does not appear in this table. No private endpoint from the platform VNet to Fabric is required. Fabric accesses the dedicated staging Storage account via Trusted Workspace Access, which operates through Microsoft-managed connectivity using a resource instance rule on the Storage account's firewall. No networking infrastructure is needed in the platform VNet for Fabric connectivity.

Note on AI Search indexer access: AI Search does not use a private endpoint to access the dedicated staging Storage account. Same-region indexer-to-storage access is provided via a resource instance rule on the Storage account's firewall (described in the AI Search indexer access section above). The AI Search service has its own private endpoint in the private endpoint subnet for management and query operations, but the indexer's outbound connection to same-region storage uses the resource instance rule path, not a private endpoint.

Note on AI Search CMK access to Key Vault: AI Search accesses Key Vault for CMK key-unwrap operations through a shared private link, a search-service-managed private endpoint provisioned by AI Search from within Microsoft-managed infrastructure. This is not a customer-deployed private endpoint and is not included in the inventory above. It requires approval on the Key Vault side and is billed separately per Azure Private Link pricing. AI Search does not traverse the platform VNet private endpoint to Key Vault and does not require the trusted-services firewall bypass. RBAC (Key Vault Crypto Service Encryption User assigned to sami-search) remains the primary access control. Provisioning and approval workflow is tracked in ADR-005 follow-ups.

Note on the dedicated staging Storage account: The DFS private endpoint in this table serves the processing layer write path. The Fabric workspace accesses the same Storage account via Trusted Workspace Access (read path via OneLake shortcuts). The AI Search indexer accesses the same Storage account via a resource instance rule (read path for index population). These are three independent mechanisms. The private endpoint provides a private IP for the processing layer within the VNet. Each resource instance rule provides a firewall exception scoped to a specific service resource ID. All three operate simultaneously on the same Storage account with public access disabled.

All Private DNS Zones are linked to the platform VNet. If the client uses custom DNS (on-premises DNS, Azure DNS Private Resolver), conditional forwarding rules route privatelink.* zones to Azure DNS (168.63.129.16).

Storage account topology: The platform uses separate Storage Accounts for document ingestion and staging. The document ingestion account receives raw inbound files and is not exposed to Fabric Trusted Workspace Access or AI Search indexer resource instance rules. The dedicated staging account holds extraction output consumed by Fabric and AI Search and is the only Storage Account that carries the shared read boundary for the Fabric workspace identity, the AI Search system-assigned managed identity, and the processing-layer private endpoint write path. This separation provides clearer security boundaries, cleaner RBAC scoping, independent lifecycle and encryption policy decisions, and lower blast radius than a single account with multiple containers serving different trust zones. Consolidation into a single account is permitted only as an explicit client exception and requires ADR-004 and ADR-005 to define compensating controls in full.

Public Access Posture Where the service supports explicit public network access control, public access is disabled by default. Azure Monitor access is constrained through AMPLS with ingestion and query access modes set to Private Only. The Foundry resource, AI Search, Storage accounts, and Key Vault are configured with publicNetworkAccess: Disabled at deployment via IaC.

The 'Allow trusted Microsoft services to access this storage account' checkbox is not enabled on the dedicated staging Storage account. All non-private-endpoint access to the dedicated staging Storage account is controlled via resource instance rules scoped to specific service resource IDs: one for the Fabric workspace (TWA) and one for the AI Search service (indexer access). This enforces least-privilege at the network layer; the broader trusted service exception would admit any qualifying Azure service in the tenant.

Trusted service exceptions are used only where a documented first-party dependency requires them (for example, Foundry accessing its backing storage). They are not used on the dedicated staging Storage account.

The dedicated staging Storage account has publicNetworkAccess set to Disabled. Microsoft documents that when public network access is set to Disabled, previously configured resource instance rules and exceptions remain effective by design. The Fabric workspace and the AI Search service are the only non-private-endpoint paths to the Storage account.

---

## Data Residency Enforcement at Network Layer

ADR-002 established Azure Policy to deny Global Standard model deployments, forcing DataZone Standard EU or Regional Standard. This ADR extends data residency controls to the network layer:

All private endpoints are deployed in the selected EU region. All Private DNS Zones resolve to private IPs within the EU region VNet. The document ingestion account and the dedicated staging Storage account are deployed in the EU region with LRS or ZRS redundancy (no GRS to avoid cross-region replication). The AMPLS is configured with Private Only access modes, ensuring telemetry data ingestion and query traffic traverse the private path. The Fabric workspace capacity is in the EU region. Trusted Workspace Access traffic between Fabric and the dedicated staging Storage account uses Microsoft-managed connectivity.

The architecture is designed so that storage, compute, model processing, and telemetry remain in approved EU regions and use private Microsoft-managed connectivity wherever available. Legal residency obligations should be validated against the client's regulatory interpretation and Microsoft's service-specific data residency commitments.

Azure Policy is recommended to deny the creation of resources outside approved EU regions.

---

## Processing Layer VNet Integration

The internal Azure Container Apps environment routes all outbound traffic from the frontend, API, and worker services through the compute subnet to the platform private endpoints. The processing layer reaches all downstream services over private connectivity: Foundry for inference and extraction, AI Search for indexing, Service Bus for dispatch, Azure SQL Database for workflow state, the dedicated staging Storage account for writing extraction results, the document ingestion Storage account for reading source documents, and Key Vault for secrets.

Inbound traffic to the processing layer enters through the frontend container app and the FastAPI API app. The reference architecture uses an internal Container Apps environment by default; any external exposure is mediated deliberately through enterprise ingress such as Application Gateway or another approved reverse proxy with Entra ID controls. Pipeline triggers and background dispatch are routed through Service Bus and processed within the private network boundary.

---

## Future Migration Path: OneLake Shortcuts via Managed Private Endpoints

OneLake shortcuts do not yet support connections to ADLS Gen2 via managed private endpoints. This is a documented limitation with an active community feature request (December 2025). The current architecture uses Trusted Workspace Access for shortcut connectivity, which is GA and fully supported.

If managed private endpoint support for OneLake shortcuts becomes available, the architecture can migrate from TWA-based shortcuts to MPE-based shortcuts. The migration path:

Create a managed private endpoint from the Fabric workspace to the dedicated staging Storage account. Approve the managed private endpoint on the Storage account. Update the OneLake shortcut authentication from workspace identity (TWA) to managed private endpoint connection. Remove the resource instance rule from the dedicated staging Storage account. Optionally remove the workspace identity if no other TWA connections exist.

No data migration, no schema change, no pipeline modification, no Lakehouse restructuring. The shortcut points to the same Storage account containers; only the authentication and network path change.

The architecture does not depend on this migration happening. Trusted Workspace Access is the primary, GA, production-grade mechanism. The managed private endpoint path is documented as a future option for organisations that prefer MPE-based networking over resource instance rules. Cross-tenant scenarios are out of scope for this ADR. If shortcut support over managed private endpoints becomes available, cross-tenant viability would require separate validation.

---

## Consequences

Positive Zero-copy access to extraction results: OneLake shortcuts via Trusted Workspace Access provide in-place read access to staging data. No data movement required for Fabric workloads that consume OneLake shortcuts to query extraction results. Data is written once (by the processing layer to the dedicated staging Storage account) and read many times (by Fabric compute engines through the shortcut). This eliminates one entire data movement step compared to Options B and C.

Fabric feature preservation: The architecture avoids the documented restrictions introduced by workspace-level private links and preserves the standard workspace operating model. Semantic models, deployment pipelines, Copilot, Spark starter pools, workspace monitoring, item sharing, OneLake Security, Power BI export, and email subscriptions remain available because the workspace does not require inbound private link protection.

No additional Fabric-side networking infrastructure: No managed VNet, no managed private endpoint, no VNet Data Gateway, no gateway nodes. The only Fabric-side requirements are a workspace identity and an F SKU capacity, both of which are already platform prerequisites.

Standard Azure networking patterns for the platform VNet: Every platform service is secured with standard private endpoints. No Fabric-specific private link configuration. No workspace FQDN construction. No Fabric-specific DNS zones. Enterprise network teams understand this pattern immediately.

Network-isolated data path end to end: The processing layer writes to the dedicated staging Storage account via private endpoint within the VNet. Fabric reads from the dedicated staging Storage account via Trusted Workspace Access over Microsoft-managed connectivity. AI Search reads from the dedicated staging Storage account via a resource instance rule with system-assigned managed identity authentication. No general public data-plane access path is required for platform-controlled Azure resources. The dedicated staging Storage account has public access disabled; resource instance rules scoped to the Fabric workspace and the AI Search service are the only non-private-endpoint access paths.

Data residency at every layer: Compute (Azure Container Apps in an EU region), inference (DataZone Standard EU), messaging (Service Bus Premium in the EU region with private endpoint), workflow state (Azure SQL Database in the EU region), storage (all resources in the EU region with private endpoints), staging (dedicated Storage account in the EU region), telemetry (AMPLS Private Only in the EU region), Fabric (workspace on EU capacity, Trusted Workspace Access to EU Storage account). The architecture is designed so that all platform-controlled data remains in approved EU regions, supported by regional deployment controls and private connectivity.

Client-portable: The architecture requires a VNet, private endpoints, and a Fabric workspace with workspace identity on F SKU capacity. It does not require the client to change their tenant-level or workspace-level Fabric private link configuration. Works alongside existing Fabric private link policies or without them. No coordination with Fabric tenant administrators beyond enabling workspace identity.

Auditable network posture: Every platform service has public access disabled. The control story: "all platform-controlled Azure services are network-isolated via private endpoints; public access is disabled; the Fabric workspace accesses the dedicated staging store via Trusted Workspace Access with a resource instance rule scoped to a single workspace; the AI Search indexer accesses the dedicated staging store via a resource instance rule scoped to a single search service; data residency is enforced by region-locked deployment and private connectivity."

No additional cost: Trusted Workspace Access is free. No gateway consumption, no managed VNet overhead. The only costs are the dedicated staging Storage account (marginal) and the Fabric F SKU capacity (already required).

Clean future migration path: If OneLake shortcuts via managed private endpoints become supported, the transition is a configuration change, not an architecture change.

Negative Same-tenant requirement: Trusted Workspace Access requires the dedicated staging Storage account and the Fabric workspace to be in the same Entra tenant. For multi-tenant scenarios (e.g., a managed service provider running the platform in their tenant and the client's Fabric workspace in the client's tenant), TWA does not work. In that scenario, managed private endpoints (Option B) or VNet Data Gateway (Option C) would be required. This ADR assumes same-tenant deployment; multi-tenant scenarios are flagged for per-client evaluation.

Maximum 200 resource instance rules: The documented limit of 200 resource instance rules is not a constraint for this architecture. The platform uses two rules (Fabric workspace and AI Search service). If additional services require firewall-protected access to the dedicated staging Storage account, each requires its own rule.

Conditional Access interaction: If the client's Entra tenant has a Conditional Access policy for workload identities that includes all service principals, the Fabric workspace identity will be blocked by the policy. The workspace identity must be explicitly excluded from such policies. This requires coordination with the client's identity team.

Additional staging store: One additional Azure resource (dedicated staging Storage account) that would not be needed if the processing layer wrote directly to the Lakehouse (Option A). This adds marginal cost (Storage is inexpensive) and one more resource to deploy and monitor.

Two storage locations for extraction output: Extraction results exist in the dedicated staging Storage account and are referenced by the Lakehouse via shortcut. Gold-layer transformations produce native Delta tables in the Lakehouse. The staging store is not the system of record for analytics; the Gold layer is. A lifecycle management policy governs retention and tiering of staging data based on auditability and reprocessing requirements. Without this, staging data accumulates at hot-tier cost indefinitely.

---

## Risks and Mitigations

Risk: Trusted Workspace Access depends on resource instance rules, which are an ADLS Gen2 firewall feature. If Microsoft changes the firewall model or deprecates resource instance rules, the access path breaks. Mitigation: Resource instance rules are a foundational ADLS Gen2 capability used broadly across Azure services (not Fabric-specific). Low probability of removal. The managed private endpoint shortcut path is the long-term migration regardless. If TWA is deprecated, the MPE shortcut feature is likely to be GA by that point.

Risk: Workspace identity gets broad access to the Storage account if RBAC is set at account scope. Mitigation: Grant RBAC at container scope (Storage Blob Data Reader on the specific staging containers), not account scope. The document ingestion account is separate and is not accessible via the workspace identity.

Risk: Entra Conditional Access policies for workload identities block the Fabric workspace identity. Mitigation: Document in the client integration guide. Identify during pre-deployment assessment. The workspace identity must be excluded from blanket Conditional Access policies targeting all service principals.

Risk: Dedicated staging Storage account becomes a single point of failure for Fabric ingestion. If the Storage account is unavailable, Fabric cannot read extraction results. Mitigation: The Storage account is deployed with ZRS (zone-redundant storage) for high availability within the region. The processing layer orchestration retries writes to Storage. For Gold-layer transformation, the Spark Notebook or Pipeline retries reads. Both are idempotent. The shortcut remains valid; it will serve data again once the Storage account recovers.

Risk: OneLake shortcut performance for large-scale reads. Shortcuts add a level of indirection compared to native Lakehouse Delta tables. Mitigation: OneLake shortcuts are designed for production-scale access. SQL analytics endpoint and Direct Lake semantic models support reading through shortcuts. For the Gold layer (heavy analytical queries, semantic models), data is transformed into native Lakehouse Delta tables. Shortcuts serve the Bronze/Silver layer; Gold is native. Performance-sensitive workloads read from Gold, not from shortcuts.

Risk: AI Search resource instance rule requires a system-assigned managed identity on the search service. If the search service is deleted and recreated, the identity changes and RBAC assignments (ADR-004) must be reassigned. Mitigation: AI Search is a long-lived stateful service (contains indexes) that is not routinely recreated. The resource instance rule references the search service resource ID, not the identity's object ID, so the rule itself does not change if the identity is recycled. However, RBAC assignments reference the identity's principal ID and must be reassigned. Include in the deployment runbook.

---

## Network Topology Evolution

The current private endpoint topology connects the Container Apps environment workloads directly to the Foundry resource, AI Search, Key Vault, Service Bus, Azure SQL Database, and the two Storage Accounts. All model inference traffic flows over the API and worker apps to the Foundry private endpoint within the platform VNet.

If the platform scales beyond a single workload and introduces APIM as an AI Gateway (see ADR-001, Platform Scaling Boundary), the model-access path changes. APIM is an enterprise-wide shared service. It does not reside in the workload VNet or subscription. It is provisioned in its own VNet and subscription, potentially within the organisation's platform hub, managed by a central platform or networking team.

The workload-side changes are limited. The direct Container Apps workload-to-Foundry private path is decommissioned. A new private endpoint is created for the APIM instance in the hub VNet. Model inference traffic flows from the frontend, API, and worker services through this private path to APIM, which forwards to Foundry over its own private endpoint. The connectivity between the workload VNet and the APIM VNet uses VNet peering or Private Link service depending on the organisation's hub-spoke topology.

All other private endpoints within the workload VNet are unchanged. The Container Apps environment to AI Search, Key Vault, Service Bus, Azure SQL Database, and Storage paths remain private, and the AI Search to Storage (via resource-instance rule) and Fabric to Storage (via Trusted Workspace Access) paths are not affected. No subnet changes, address space changes, or VNet redesign are required in the workload landing zone.

The hub-side provisioning (APIM VNet integration, APIM to Foundry private endpoint, cross-VNet DNS resolution, APIM policy configuration, and the operational model for gateway management) is outside the scope of this ADR. This ADR governs the workload landing zone network. The enterprise hub network topology and APIM provisioning are the responsibility of the central platform team and would be documented in their own infrastructure decisions.

This section does not constitute a decision to adopt APIM. The triggers, justification, and migration path are documented in ADR-001. This section documents only the workload-side network implications so that the current topology can be validated against the potential future state.

---

## Follow-ups

IaC: Encode the full networking specification: VNet, subnets (compute subnet delegated to Microsoft.App/environments), private endpoints, DNS zones, AMPLS, public access disabled on all resources, dedicated staging Storage account with lifecycle management policy, document ingestion Storage account, resource instance rules for both the Fabric workspace and the AI Search service on the dedicated staging Storage account.

Fabric workspace configuration: Automate workspace identity enablement, RBAC assignment on the dedicated staging Storage account containers, and OneLake shortcut creation via the Microsoft Fabric Terraform provider (fabric_workspace with identity block, fabric_shortcut with adls_gen2 target) or the Fabric REST API. RBAC assignment is handled via the Azure IaC layer (Bicep or azurerm_role_assignment). No manual portal steps.

Deployment pipeline: Codify the deployment sequencing: Azure resources, VNet, and private endpoints first; dedicated staging Storage account with resource instance rules second; Fabric workspace, workspace identity, and RBAC third; OneLake shortcuts fourth; end-to-end validation fifth. The pipeline enforces ordering and dependencies. A companion document explains the rationale behind the sequencing for operators and auditors.

Client integration guide: Document what the platform requires from the client's environment: VNet (or VNet peering if hub-and-spoke), DNS forwarding if custom DNS, Fabric workspace with F SKU capacity in EU region, workspace identity enabled, Entra Conditional Access exclusion for the workspace identity if blanket workload identity policies exist, Storage Blob Data Reader RBAC on the staging containers for the workspace identity.

Staging retention policy: Define a lifecycle management policy on the dedicated staging Storage account. After Gold-layer transformation has completed and been validated, staging data can be moved to cool or archive tier or deleted. The retention period depends on the use case (default: 30 days for auditability, configurable per client).

Spike: End-to-end private data flow validation: Validate that the processing layer can write to the dedicated staging Storage account via private endpoint, that a Fabric Lakehouse OneLake shortcut can read from the dedicated staging Storage account via Trusted Workspace Access with public access disabled, that the AI Search indexer can read from the dedicated staging Storage account via resource instance rule with public access disabled, that a Spark Notebook can read through the shortcut (confirming Spark treats the shortcut as a OneLake folder and does not require a direct TWA or MPE connection to ADLS Gen2), that a Spark Notebook can write to native Lakehouse Delta tables, and that the SQL analytics endpoint can query shortcut-referenced tables.
