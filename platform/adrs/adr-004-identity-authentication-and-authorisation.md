# ADR-004: Identity, Authentication, and Authorisation

**Status**: Accepted  
**Date**: 10/03/2026  
**Decision Scope**: Identity primitives, authentication model, authorisation strategy, and key management for all platform services  
**Depends on**: ADR-001 (Processing Paradigm), ADR-002 (Runtime Platform and EU Region Selection), ADR-003 (Network Isolation and Data Residency Enforcement)  
**Depended on by**: ADR-005 (Data Protection), ADR-006 (Observability and Operational Model)  

---

## Context

ADR-003 established the network isolation model. ADR-004 established the identity and RBAC model. This ADR addresses the next layer: encryption of data at rest using customer-managed keys where supported, and enforcement of TLS for data in transit.

Every major platform service either stores data directly or processes protected data with persisted state, encrypted artefacts, or telemetry implications. Extraction results in Storage, indexed content in AI Search, model interactions and project artefacts in Foundry, lakehouse and warehouse data in Fabric, workflow state in Azure SQL Database, transient work dispatch in Azure Service Bus, and telemetry in Log Analytics. The question is whether Microsoft-managed encryption (the default for all Azure services) is sufficient, or whether customer-managed keys are required, and if so, how key management, rotation, and access are handled across the platform.

---

## Decision Drivers

**Regulatory defensibility.** Regulated environments (financial services, healthcare, public sector) may require demonstrable control over encryption keys, not just that data is encrypted. The ability to show that encryption keys are customer-owned, rotatable, and revocable is a distinct audit requirement from "data is encrypted at rest."

**Zero-touch key management.** Key rotation should not require manual intervention. Auto-rotation through Key Vault with versionless key URIs is the target operating model. Manual rotation introduces operational risk and audit exposure.

**Blast radius containment.** Key revocation should be able to render specific data stores inaccessible without affecting the entire platform. Per-service key separation supports this, though all keys reside in a shared vault which remains a dependency concentration point. The revocation granularity and dependency concentration coexist; per-service keys mitigate but do not eliminate the shared-vault dependency.

**Operational simplicity.** CMK adds complexity (Key Vault dependency, identity configuration, rotation monitoring, deployment sequencing). The architecture should not apply CMK where it adds significant complexity without proportionate security benefit.

**Cost awareness.** Some CMK configurations require higher-tier resources. The architecture documents where CMK introduces cost thresholds so the decision is informed.

---

## Considered Alternatives

**Option A: Microsoft-managed keys everywhere (MMK-only).** Accept the Azure default. Every service encrypts at rest with Microsoft-managed keys. No Key Vault dependency for encryption. Simplest operationally. No rotation overhead. The trade-off: the client cannot demonstrate independent control over encryption keys, which may not satisfy audit requirements in regulated environments.

**Option B: CMK everywhere (maximum control).** Every service that supports CMK is configured with customer-managed keys. Maximum control and auditability. The trade-off: CMK for Log Analytics requires a dedicated cluster with a minimum 100 GB/day commitment tier, which is unlikely to be cost-justified for a single-workload platform. The operational overhead of managing CMK for transient messaging services and lower-sensitivity operational stores is disproportionate to the security benefit.

**Option C: CMK for business data stores, MMK for operational services (SELECTED).** Apply CMK to supported services storing business or regulated data where the control benefit is proportionate and operationally supportable. Use MMK for operational or transient services, or where CMK is unavailable or economically disproportionate. This creates a defensible control story ("all business data is encrypted with customer-controlled keys") without requiring infrastructure that cannot be justified for the workload.

---

## Decision

Apply CMK to supported services storing business or regulated data where the control benefit is proportionate and operationally supportable. Use MMK for operational or transient services, or where CMK is unavailable or economically disproportionate.

The per-service inventory in section 6 documents the specific classification of each platform service under this decision.

---

## Key Vault Configuration

**Tier**: Azure Key Vault Premium. HSM-backed keys. FIPS 140-3 Level 3 (upgraded from FIPS 140-2 Level 2 in June 2025; both Key Vault Premium and Managed HSM now share the same FIPS level).

**Region**: EU region, co-located with platform resources.

**Required settings**: Soft-delete enabled. Purge protection enabled. Azure RBAC access model (not vault access policies).

**Network access**: Private endpoint in the platform VNet is the baseline access path (per ADR-003). "Allow trusted Microsoft services to bypass this firewall" is enabled as a controlled exception to the private-endpoint-only posture. This exception is required because Storage, Foundry, and Fabric CMK operations are documented by Microsoft as requiring first-party service access to Key Vault rather than traversing the platform VNet data path. Azure SQL Database also uses a first-party service integration path to Key Vault for TDE with CMK, but the exact dependency on the selected firewall posture remains an implementation validation item documented in Open Item 7. AI Search follows a different pattern through a search-service-managed shared private link and does not rely on the trusted-services bypass. This is not a general relaxation of network isolation; it is a specific, bounded exception driven by documented CMK access requirements. RBAC remains the primary access control; the bypass alone grants no data-plane access without the corresponding Key Vault role assignment. This exception must be reviewed if Azure service CMK access patterns change to support private-endpoint-only connectivity.

ADR-003 does not currently address Key Vault firewall bypass settings. This ADR formally introduces the trusted-services bypass as a controlled exception to ADR-003's private-endpoint baseline. ADR-003 should be annotated to acknowledge this exception and its scope.

**Managed HSM alternative**: Managed HSM provides single-tenant HSM isolation and a customer-managed security domain. Both Managed HSM and Key Vault Premium are FIPS 140-3 Level 3; the differentiation is operational control and single-tenant isolation, not cryptographic assurance level. Managed HSM is the appropriate choice only when the client has explicit single-tenant HSM regulatory requirements or key sovereignty obligations that go beyond key control. For mid-market regulated platforms, Key Vault Premium is the default.

---

## Per-Service CMK Inventory

### 6.1 Azure Storage (ADLS Gen2): CMK

**Encryption scope**: All data at rest in the storage account, including blob data (extraction results, document uploads, intermediate processing outputs).

**Configuration**: CMK enabled at the storage account level. Versionless key URI configured. Zero-touch rotation is documented by Microsoft: when configured with a versionless key identifier, Azure Storage automatically detects new key versions after Key Vault auto-rotation within 24 hours.

**Identity**: Dedicated user-assigned managed identity (`uami-storage-cmk`). Microsoft documentation states that when configuring CMK at the time a new storage account is created, a user-assigned managed identity is required; system-assigned managed identity is only supported when configuring CMK on an existing storage account. Since IaC pipelines provision new storage accounts with CMK enabled at creation time, a user-assigned managed identity is the only supported option. This is not an architectural preference; it is a platform constraint. The identity's sole purpose is accessing the encryption key; it has no other role assignments.

**RBAC**: `uami-storage-cmk` requires Key Vault Crypto Service Encryption User on the Key Vault. This role assignment is documented in ADR-004 section 4.2.1.

**Key Vault access path**: Azure Storage is listed as a trusted Microsoft service. Storage CMK operations are documented by Microsoft as requiring the "Allow trusted Microsoft services to bypass this firewall" setting on Key Vault. This reinforces the requirement documented in section 5.

**Key Vault region**: Same tenant required. Same region recommended for latency but not mandatory.

**Constraints**: None that affect the platform architecture. CMK for Storage is mature and well-documented.

### 6.2 Azure AI Search: CMK

**Encryption scope**: Per-object. CMK applies to indexes, synonym lists, indexers, data sources, and skillsets individually at creation time. Sensitive content within these objects (connection strings, keys, user inputs, references to external resources) is encrypted. All content within indexes and synonym lists is encrypted.

**Configuration**: CMK specified in the object creation call (REST API, SDK, or Azure portal). Versionless key URI supported and explicitly recommended by Microsoft documentation. Auto-rotation works: AI Search picks up the new key version; the key is cached for up to 60 minutes.

**Identity**: `sami-search` (the system-assigned managed identity of the AI Search service, defined in ADR-004).

**RBAC**: `sami-search` requires Key Vault Crypto Service Encryption User on the Key Vault. This role assignment is documented in ADR-004 section 4.2.

**Key Vault access path:** AI Search accesses Key Vault for CMK operations through a shared private link: a search-service-managed private endpoint provisioned by AI Search and approved on the Key Vault side. This is distinct from the platform VNet private endpoint used by compute-layer services. AI Search does not traverse the customer VNet private endpoint and does not require the trusted-services firewall bypass. RBAC remains the primary authorisation control; the shared private link provides network reachability, not data-plane permission. Shared private links are a premium feature with per-link usage-based pricing; cost is tracked under the AI Search resource. Provisioning and approval workflow is validated during the IaC spike (see Follow-ups).

**Deployment sequencing constraint**: Per Microsoft documentation, CMK encryption is irreversible and applies per object at creation time. Objects created without CMK cannot be retroactively encrypted; they must be deleted and recreated with CMK specified in the creation call. The Key Vault key must exist and the `sami-search` identity must have Key Vault Crypto Service Encryption User before the first index is created. IaC pipelines must enforce this sequencing: Key Vault key provisioned, RBAC assignment confirmed, then index creation with CMK configuration.

**Performance**: CMK adds latency to indexing and query operations due to encrypt/decrypt overhead. This should be factored into SLA expectations.

**Tier requirement**: Billable tier (Basic or higher). Free tier does not support CMK.

**Policy enforcement**: Azure Policy can enforce CMK compliance. The AuditIfNotExists policy (built-in) flags non-compliant objects. The Deny policy blocks creation of search services without `SearchEncryptionWithCmk` set to `Enabled`. Policy enforcement requires the Management REST API; the Azure portal and CLI do not expose the `encryptionWithCmk.enforcement` property natively.

### 6.3 Azure AI Foundry (Cognitive Services): CMK

**Encryption scope**: Data at rest stored in the Foundry resource's associated storage, including project artefacts, uploaded files, and evaluation data. Double encryption is provided automatically when CMK is configured (Microsoft-managed layer plus customer-managed layer).

**Configuration**: CMK enabled at the Foundry resource level via the Encryption blade or at resource creation time. Key Vault URL and key name specified in the resource configuration.

**Identity**: System-assigned managed identity of the Foundry resource is preferred for CMK because the CMK identity lifecycle should be coupled to the Foundry resource. If the Foundry resource is deleted, the CMK identity is automatically deprovisioned, eliminating orphaned role assignments. A user-assigned managed identity is also supported. ADR-004 section 4.2.1 documents the RBAC assignment.

**RBAC**: The Foundry CMK identity requires Key Vault Crypto User on the Key Vault per the current Foundry CMK documentation (March 2026, `learn.microsoft.com/en-us/azure/foundry/concepts/encryption-keys-portal`). Note: the older Foundry-classic/OpenAI encryption documentation (`learn.microsoft.com/en-us/azure/foundry-classic/openai/encrypt-data-at-rest`) specifies Key Vault Crypto Service Encryption User. Key Vault Crypto User is broader (includes encrypt/decrypt and sign/verify in addition to wrap/unwrap). The current implementation baseline follows the newer Foundry documentation and the Azure Verified Module Terraform pattern, both of which specify Key Vault Crypto User. This is treated as a documented provisional broadening of least privilege pending validation of whether Key Vault Crypto Service Encryption User is sufficient for Foundry CMK operations. Open Item 5 tracks validation of the minimum required role.

**Key Vault access path**: Based on current documentation (March 2026), the Foundry CMK documentation requires "Allow trusted Microsoft services to bypass this firewall" on the Key Vault in both supported configurations: (a) private link endpoint with trusted services enabled, described as the recommended configuration, and (b) trusted services enabled without a private endpoint. The trusted-services bypass is a controlled exception to the private-endpoint baseline, documented in section 5.

**Key Vault region**: Must be in the same Azure region as the Foundry resource. This is a hard requirement, unlike Storage where same-region is recommended but not mandatory.

**Auto-rotation**: The Foundry CMK documentation does not explicitly confirm versionless key URI support for auto-rotation. The documentation states: "Rotate keys regularly. Update the Foundry resource to reference the latest key version." This implies a manual rotation pattern. The underlying Cognitive Services platform may support versionless keys implicitly, but this is not documented. **Open Item 1**: Validate during the implementation spike whether Foundry CMK supports versionless key URIs for zero-touch auto-rotation. Until validated, the operational model must include a manual key version update step after each Key Vault auto-rotation event.

**Constraint**: Projects can be updated from Microsoft-managed keys to CMK, but cannot be reverted. Project CMKs can only be updated to keys in the same Key Vault.

### 6.4 Microsoft Fabric: CMK

**Encryption scope**: Workspace-level. Envelope encryption: Microsoft-managed DEK encrypts data; customer-managed KEK in Key Vault encrypts the DEK. Supported items: Lakehouse, Warehouse, Notebook, Environment, Spark Job Definition, API for GraphQL, ML model, Experiment, Pipeline, Dataflow, Mirrored Database, SQL Database (preview), Industry solutions. Unsupported items cannot be created in a CMK-enabled workspace.

**Configuration**: Enabled per workspace in Fabric portal settings. Requires tenant-level "Apply customer-managed keys" setting to be enabled by a Fabric administrator. Versionless key URI is mandatory; Fabric only supports versionless keys. Fabric checks Key Vault daily for a new key version and uses the latest version available. After creating a new key version, wait 24 hours before disabling the older version to avoid an access gap.

**Identity**: Fabric Platform CMK service principal. This is the tenant-local enterprise application instance of a Microsoft-managed multi-tenant application. The platform team provisions this service principal in the tenant's Entra ID and grants it the Key Vault role, but does not control the upstream application lifecycle managed by Microsoft. The identity pattern is: a Cloud Application Administrator provisions the tenant-local service principal for the Fabric Platform CMK app, then a Key Vault Administrator grants it the required role.

**RBAC**: The Fabric Platform CMK service principal requires Key Vault Crypto Service Encryption User on the Key Vault.

**Key Vault access path**: Fabric is a SaaS platform. It is not VNet-injected. The Fabric Platform CMK service principal accesses Key Vault from Microsoft-managed infrastructure, not from the platform VNet. It cannot traverse the Key Vault private endpoint. The Fabric CMK documentation explicitly states: "You can also use Azure Key Vaults for which the firewall setting is enabled. When you disable public access to the Key Vault, you can choose the option to 'Allow Trusted Microsoft Services to bypass this firewall.'" This bypass is mandatory for Fabric CMK. Microsoft Power Platform is on the Key Vault trusted services list; the bypass combined with the RBAC assignment is the control.

**Key Vault requirements**: Soft-delete and purge protection enabled. RSA or RSA-HSM key type. Supported key sizes: 2048, 3072, 4096 bit. Fabric documentation should be rechecked if a specific item type introduces tighter limits.

**Data not protected by CMK**: Fabric CMK does not cover all data in a CMK-enabled workspace. Unprotected categories include Lakehouse structural metadata, Spark cluster temporary data, job logs, environment libraries and session customisations, Pipeline and Copy job metadata, ML model and experiment metadata, and Warehouse Object Explorer queries and backend cache. CMK-enabled workspaces must be treated as partial-coverage controls and assessed against the client's data classification requirements.

**Revocation**: Revoking the key blocks read and write access to the workspace within 60 minutes. SQL Database in Fabric requires manual revalidation to restore access after key reinstatement.

**Deployment sequencing constraint**: A CMK-enabled workspace cannot contain unsupported items. If unsupported items are needed, they must be created in a separate non-CMK workspace. This workspace segmentation must be planned before any Fabric items are provisioned.

**Tier requirement**: F SKU required. Trial capacities do not support CMK. CMK cannot be enabled for workspaces with BYOK enabled.

### 6.5 Azure SQL Database: CMK

**Encryption scope**: Database files, transaction log files, backups, and persisted workflow-state records including jobs, executions, retries, approval checkpoints, and audit-oriented operational data.

**Configuration**: Transparent Data Encryption with a customer-managed protector in Key Vault. The database uses Entra authentication for application access and a versionless key URI for the TDE protector where supported by the deployment tooling and Azure SQL integration path.

**Identity**: The Azure SQL logical server identity accesses the TDE protector in Key Vault. Server managed identity is the default identity pattern. A dedicated user-assigned managed identity may be used where separation from the server identity is explicitly required by deployment policy or operating model. Application identities do not manage encryption keys directly; they authenticate separately for database access.

**RBAC**: The Azure SQL CMK identity requires the documented Key Vault crypto role for the TDE protector. Database access for the API and worker identities is granted through Entra-authenticated contained users and schema-scoped database permissions, governed by ADR-004.

**Key Vault access path**: Azure SQL TDE with CMK is a first-party service integration path to Key Vault. The logical server identity accesses the TDE protector as part of the Azure SQL encryption control plane, not as an application data-plane call traversing the workload VNet. This path is treated as part of the controlled Key Vault trusted-services exception described in section 5 and must be validated during implementation against the final Key Vault firewall posture.

**Auto-rotation**: Versionless key support and end-to-end zero-touch rotation behaviour for the Azure SQL TDE protector must be validated in the chosen deployment tooling path if not already proven by the implementation spike. If tooling or service behaviour requires explicit key version updates, the platform must treat SQL as a documented exception to the zero-touch target in the same way as Foundry. Until that validation is complete, Azure SQL should be treated as expected to support the target model but not yet closed as proven in this architecture.

**Rationale**: Workflow state is durable, queryable, and part of the platform's audit story. This is no longer opaque scheduler state hidden behind a platform service. Applying CMK to the SQL store aligns with the decision to treat workflow state as a first-class regulated data asset.

### 6.6 Azure Service Bus: MMK

**Encryption scope**: Namespace-level encryption at rest using Microsoft-managed keys.

**Decision**: Microsoft-managed keys.

**Rationale**: Azure Service Bus carries transient work dispatch, coordination messages, and review routing events. It is not the durable system of record for workflow execution, approvals, or operational evidence; Azure SQL Database holds that role. Applying CMK to Service Bus would add Key Vault dependency, identity configuration, and rotation overhead to a transient coordination plane without materially improving the protection posture of the authoritative data store.

**Boundary condition**: This decision assumes queue and topic payloads remain operationally minimal and do not become a shadow copy of business data, extraction output, or human-review evidence. If queue payloads begin carrying sensitive extracted content, approval evidence, or regulated business records beyond what is necessary for dispatch and correlation, the MMK decision must be revisited and Service Bus escalated to explicit CMK assessment.

**Tier note**: Azure Service Bus Premium supports customer-managed keys. The platform is not rejecting CMK because the service lacks support; it is rejecting CMK because the security benefit is disproportionate for the current message role in the architecture.

### 6.7 Log Analytics Workspace: MMK

**Rationale (layered)**:

First, CMK for Log Analytics requires a dedicated cluster. The minimum commitment tier is 100 GB/day. For a single-workload AI platform, daily telemetry volume is unlikely to approach 100 GB/day. The dedicated cluster cost floor is not justified for the expected workload.

Second, telemetry data is operational data. However, telemetry must be treated as capable of carrying sensitive content unless explicitly constrained by application logging policy. LLM-based systems are particularly vulnerable to accidental prompt and completion content appearing in trace payloads, custom dimensions, or exception details. The sensitivity of telemetry is therefore a function of what the application logs, not the telemetry infrastructure.

Third, if a client's data classification policy classifies telemetry as sensitive, some regulated environments do, particularly where trace payloads might capture input or output content, the escalation path is: provision a dedicated Log Analytics cluster and enable CMK on the workspace. This is additive; it does not require redesigning the architecture. The cost and volume thresholds should be documented in the client engagement so the decision is revisited if telemetry volume grows or classification changes.

**Code-level governance constraint**: The frontend, API, worker, and job code must not log PII, business data, or extraction content in trace payloads, custom dimensions, or custom metrics. If they do, the telemetry becomes business data and the MMK decision must be revisited. This constraint feeds into ADR-006 (Observability) as a binding rule on what the application is permitted to emit in telemetry. ADR-006 must document this as a governance control with an enforcement mechanism, code review checklist, automated scanning for PII patterns in telemetry output, or both.

---

## Key Auto-Rotation Strategy

**Target operating model**: Zero-touch rotation via Key Vault auto-rotation policy with versionless key URIs, for services that support it. Current exception: Foundry, where versionless key support is not documented and requires validation. Azure SQL follows the target model only after the implementation path confirms versionless TDE protector handling in the chosen tooling and service integration flow. Until Azure SQL is validated, the platform's rotation model is zero-touch for Storage, AI Search, and Fabric, with a monitored manual or automated step for Foundry and a validation-dependent posture for Azure SQL.

**Key Vault rotation policy**: Configured per key. Default rotation interval: 180 days, configurable per client requirement. Key expiry: 2 years. Event Grid notification: 30 days before expiry. Notification enables operational alerting if auto-rotation fails or if manual intervention is needed for services that do not fully support versionless keys.

**Per-service auto-rotation behaviour**:

Azure Storage: Versionless key URI supported. Auto-detects new key version within 24 hours. Zero-touch. Documented by Microsoft.

Azure AI Search: Versionless key URI supported and recommended. Picks up new key version; cached key expires within 60 minutes. When rotating, all encrypted objects must be updated to reference the new key before the old key version is deleted or disabled. Failure to maintain this sequence renders affected objects permanently unusable. This is an irreversibility constraint, not an operational preference.

Microsoft Fabric: Versionless key URI is mandatory. Fabric does not support versioned keys. Checks Key Vault daily for new version. Wait 24 hours after creating a new key version before disabling the old version. Zero-touch. Documented by Microsoft.

Azure AI Foundry: Versionless key URI support not confirmed in documentation. The documented rotation pattern is manual: "Update the Foundry resource to reference the latest key version." **Open Item 1**. Until validated, the operational model must include monitoring for Key Vault rotation events, via Event Grid, and a manual or automated step to update the Foundry resource's key version reference after each rotation.

Azure SQL Database: Versionless key handling and zero-touch rotation behaviour for the TDE protector must be validated in the chosen implementation path. If validated, Azure SQL joins the zero-touch model. If not, it becomes a managed exception with explicit update procedure after key rotation.

---

## RBAC Assignments for CMK

All CMK identity definitions and Key Vault role assignments are documented in ADR-004 section 4.2.1. The following table is a summary cross-reference; ADR-004 is the authoritative source for identity definitions and RBAC assignments. ADR-005 documents the rationale for why each identity needs Key Vault access and what encryption capability it enables.

|Identity|Role|Key Vault Access Path|ADR-004 Reference|ADR-005 Rationale|
|---|---|---|---|---|
|uami-storage-cmk|Key Vault Crypto Service Encryption User|Trusted services bypass|Section 4.2.1|Section 6.1|
|sami-search|Key Vault Crypto Service Encryption User|Shared private link to Key Vault (search-service-managed PE), no trusted-services bypass|Section 4.2|Section 6.2|
|Foundry CMK identity, Foundry resource SAMI preferred|Key Vault Crypto User, provisional; see section 6.3|Trusted services bypass|Section 4.2.1|Section 6.3|
|Fabric Platform CMK service principal|Key Vault Crypto Service Encryption User|Trusted services bypass, Fabric is SaaS and not VNet-injected|Section 4.2.1|Section 6.4|
|Azure SQL Database server managed identity or uami-sql-cmk|Key Vault Crypto Service Encryption User|First-party Azure SQL TDE integration path to Key Vault, dependency on trusted-services bypass subject to Open Item 7|Section 4.2.1|Section 6.5|

The Key Vault access path column documents how each identity reaches Key Vault, which is an ADR-005 concern (encryption configuration) that depends on ADR-003 (network isolation) and ADR-004 (identity). The trusted services bypass requirement and its cross-ADR dependency with ADR-003 are documented in sections 5 and 12.

Note on Foundry role: The current Foundry CMK documentation (March 2026) specifies Key Vault Crypto User. The older Foundry-classic documentation specifies Key Vault Crypto Service Encryption User. Key Vault Crypto User is the broader role. The current baseline follows the newer documentation as a provisional assignment. This is an explicit least-privilege exception pending validation that the narrower role, Key Vault Crypto Service Encryption User, is sufficient. Open Item 5 tracks this validation. See section 6.3 for detail.

---

## Deployment Sequencing Constraints

The following sequencing dependencies must be enforced in IaC pipelines. Violating them results in either failed deployments or unrecoverable states, AI Search objects created without CMK cannot be retroactively encrypted and must be recreated.

**Phase 1: Key Vault and keys.** Key Vault Premium provisioned in EU region. Soft-delete, purge protection enabled. Private endpoint created. "Allow trusted Microsoft services" enabled. Auto-rotation policy configured. RSA keys created, per-service keys recommended for independent revocation; see Open Item 3.

**Phase 2: Identity and RBAC.** All CMK identities provisioned, `uami-storage-cmk`, `sami-search`, Foundry CMK identity, Fabric Platform CMK service principal, Azure SQL CMK identity if SQL CMK is enabled through server identity or dedicated UAMI. Key Vault role assignments confirmed for each identity. Role assignments must be active before any CMK-dependent resource is provisioned.

**Phase 3: CMK-dependent resources.** Storage account created with CMK configuration referencing versionless key URI and `uami-storage-cmk`. AI Search service created on billable tier. CMK enforcement set via Management REST API if using Deny policy. Indexes, indexers, skillsets, data sources created with CMK specified in the creation call. Foundry resource created or updated with CMK configuration. Fabric tenant setting enabled, workspace CMK enabled referencing versionless key URI. Azure SQL logical server configured with TDE protector in Key Vault before database protection posture is considered complete.

**Critical constraint, AI Search**: No index creation before Phase 1 and Phase 2 are complete. Per Microsoft documentation, CMK encryption is irreversible and per-object at creation time. An object created without CMK cannot be retroactively encrypted; it must be deleted and recreated.

**Critical constraint, Fabric**: No workspace items created before CMK is enabled on the workspace. Unsupported items must be created in a separate non-CMK workspace.

**Critical constraint, Azure SQL Database**: The Key Vault key and SQL CMK identity access must be in place before the TDE protector is set. Application runtime identities and contained database users are separate from the CMK path and must not be used as a substitute for the server-side encryption identity.

---

## TLS Enforcement

All Azure PaaS services in the platform stack enforce TLS 1.2 as the minimum by default. No service requires additional configuration to achieve this baseline. This is not an architectural decision; it is a documented platform invariant.

Azure Policy to deny creation of Storage accounts with minimum TLS version below 1.2: Policy ID `fe83a0eb-a853-422d-aff8-513bc498900d`. Deployed in audit mode initially, transitioning to deny after validation, aligned with ADR-004 Open Item 3.

---

## Data Classification and Governance (Microsoft Purview)

Encryption is one data protection control. Data classification determines which controls apply to which data. Microsoft Purview is the selected platform for data classification, governance, and information protection across the platform estate.

ADR-005 does not specify Purview configuration, label taxonomy, classification rules, or DLP policies. Those are scoped to a separate design activity that depends on the client's data classification policy, regulatory requirements, and existing Purview estate.

**Integration points between Purview and the platform:**

Microsoft Fabric workspaces, Lakehouse and Warehouse, can be scanned and classified by Purview. Sensitivity labels in Fabric control labelling governance, inheritance, downstream export restrictions, label change permissions. They do not natively configure or enforce data access controls such as row-level security or column-level masking, which are separate Fabric security mechanisms. Classification metadata from Purview can inform governance decisions but does not directly drive data-access enforcement at the query level within Fabric.

Azure Storage, ADLS Gen2, can be scanned and classified by Purview. Purview Protection Policies, public preview, provide a mechanism to restrict access to labelled blobs based on sensitivity classification. However, this feature is in public preview, requires M365 E5 licensing, and current preview documentation states that Protection Policies do not work when CMK is enabled on the storage account. Since this architecture applies CMK to the staging storage account, section 6.1, Purview Protection Policies cannot be used on the same account. If Protection Policies reach GA and the CMK limitation persists, the architecture must choose between CMK encryption control and Purview-driven access enforcement on the same storage account, or separate business data storage across accounts. This conflict is tracked as Open Item 4. Classification results from scanning can still inform governance decisions independently of Protection Policies.

Azure AI Search has a public preview integration for query-time sensitivity label enforcement, 2025-11-01-preview API. When enabled, AI Search evaluates Purview sensitivity labels at query time and filters results based on user permissions. This requires the preview API version and mandates RBAC-based query authentication, API keys are restricted to schema operations only when this feature is enabled. The platform should not take a production dependency on preview features, ADR-004 Decision Driver 7. This integration point should be monitored for GA readiness. Until GA, it does not change the current architecture but represents a future enforcement path for document-level access control in RAG scenarios.

Azure AI Foundry model interactions, prompts and completions, are not classified by Purview. Classification of AI input and output content is an application-level concern, not a platform-level Purview integration.

**Design intent**: Data classification informs protection decisions. CMK, this ADR, is one enforcement mechanism. Access control, ADR-004, is another. Purview provides the classification layer that identifies what data exists, where it resides, and what sensitivity level it carries. The platform architecture accommodates Purview integration; the detailed Purview design is a separate workstream.

Purview integration points are represented in the platform infrastructure diagram, to be produced separately.

---

## Cross-ADR Dependencies

**ADR-003, Network Isolation**: Key Vault must be configured with private endpoint plus "Allow trusted Microsoft services to bypass this firewall" enabled. ADR-003 does not currently address Key Vault firewall bypass settings. This ADR formally introduces the trusted-services bypass as a controlled exception to ADR-003's private-endpoint baseline. The bypass is required for CMK operations by Storage, Foundry, and Fabric. Azure SQL Database uses a first-party integration path to Key Vault for TDE with CMK, and the exact dependency on the selected firewall posture remains subject to Open Item 7. AI Search follows a different pattern through a search-service-managed shared private link and does not rely on the trusted-services bypass. ADR-003 should be annotated to acknowledge this exception and its scope.

**ADR-004, Identity and RBAC**: All CMK identity definitions and Key Vault role assignments are documented in ADR-004 section 4.2.1. ADR-005 references those assignments and documents the rationale for why each identity needs Key Vault access. The Fabric Platform CMK service principal, Foundry CMK identity, and Azure SQL CMK identity rows exist in ADR-004 section 4.2.1; ADR-005 owns the explanation of why they are required and how they function. The Foundry CMK role discrepancy, Key Vault Crypto User vs Key Vault Crypto Service Encryption User, is documented in section 6.3 and reflected in ADR-004.

**ADR-006, Observability**: The Log Analytics MMK decision depends on a code-level governance constraint: the Container Apps workloads must not log PII, business data, or extraction content in telemetry. ADR-006 must document this as a binding rule with an enforcement mechanism. If this constraint is violated, the MMK decision for Log Analytics must be revisited.

---

## Open Items

|ID|Description|Owner|Target|
|---|---|---|---|
|1|Validate whether Azure AI Foundry CMK supports versionless key URIs for zero-touch auto-rotation. If not, implement Event Grid monitoring for Key Vault rotation events with automated or manual Foundry resource update.|Platform team|Implementation spike|
|2|Policy enforcement transition: move TLS minimum version policy and CMK compliance policies from audit to deny mode after validation period. Aligned with ADR-004 Open Item 3.|Platform team|Post-deployment validation|
|3|Per-service vs shared Key Vault keys: determine whether each CMK-consuming service should use a dedicated key, independent revocation and cleaner audit trail, or a shared key, simpler management. Recommendation: per-service keys. Confirm during implementation.|Platform team|Implementation spike|
|4|Purview Protection Policies for Azure Storage are incompatible with CMK-enabled storage accounts, current preview limitation. If Protection Policies reach GA and the CMK limitation persists, evaluate whether to separate business data storage, CMK and no Purview enforcement, from classification-driven access control, Purview enforcement and MMK. Monitor for resolution.|Platform team|Ongoing|
|5|Validate minimum required Key Vault role for Foundry CMK. Current documentation specifies Key Vault Crypto User, broader; older documentation specifies Key Vault Crypto Service Encryption User, narrower. The broader role is assigned as a provisional baseline. If the narrower role is sufficient, prefer it and update ADR-004 section 4.2.1 and ADR-005 section 6.3.|Platform team|Implementation spike|
|6|Validate Azure SQL Database TDE protector rotation behaviour with versionless key identifiers in the chosen deployment tooling path. If explicit version updates are required, document the runbook and treat Azure SQL as a managed exception to the zero-touch rotation target.|Platform team|Implementation spike|
|7|Confirm whether Azure SQL Database CMK path under the selected Key Vault firewall posture relies on the trusted-services bypass, equivalent first-party service access semantics, or a tooling-specific exception path. Update ADR-003 and ADR-005 wording if implementation evidence narrows the statement.|Platform team|Implementation spike|

---

## Follow-ups

**IaC**: Encode the full CMK specification per service: Key Vault Premium with auto-rotation policy, RSA keys per CMK-consuming service, versionless key URIs, RBAC assignments for all CMK identities, ADR-004 section 4.2.1, Storage account CMK configuration with `uami-storage-cmk`, AI Search CMK enforcement via Management REST API, Foundry resource CMK configuration, Fabric tenant setting and workspace CMK enablement, and Azure SQL TDE protector configuration. Deployment sequencing, section 9, must be codified in the pipeline: Key Vault and keys first, identity and RBAC second, CMK-dependent resources third.

**Policy enforcement transition**: Deploy Azure Policies in audit mode, TLS minimum version policy `fe83a0eb-a853-422d-aff8-513bc498900d`, AI Search CMK AuditIfNotExists. Validate no deployment or runtime operations depend on prohibited configurations. Transition to deny after validation period. Aligned with ADR-004 Open Item 3.

**ADR-006 telemetry governance constraint**: ADR-006 must document the binding rule that the Container Apps workload code must not log PII, business data, or extraction content in telemetry payloads. ADR-006 must define the enforcement mechanism, code review checklist, automated PII scanning, periodic telemetry audit. This constraint is a direct dependency of the Log Analytics MMK decision in section 6.7.

**Service Bus payload boundary rules**: Document Service Bus payload boundary rules in application design guidelines. Define what constitutes permissible dispatch content (job ID, correlation ID, message type, timestamp) versus prohibited content (extraction output, approval evidence, business records). Enforce through code review and periodic payload audit. This mirrors the Log Analytics telemetry governance constraint pattern and keeps the two MMK decisions symmetrical in their enforcement approach.

**Spike: Foundry CMK versionless key validation**: Test whether configuring Foundry CMK with a versionless key URI results in automatic pickup of new key versions after Key Vault rotation. If not, implement Event Grid subscription for key rotation events with an automated step to update the Foundry resource's key version reference. This spike resolves Open Item 1 and determines whether the Foundry rotation risk in section 15 can be closed.

**Spike: Azure SQL CMK rotation validation**: Test whether Azure SQL Database TDE with customer-managed protector follows zero-touch rotation with versionless key identifiers in the chosen provisioning path. If not, define the operational update procedure and alerting model. This spike resolves Open Item 6.

**Spike: End-to-end CMK validation**: Validate that all CMK-configured services can encrypt and decrypt data successfully with the platform Key Vault configuration, private endpoint, trusted services bypass, RBAC. Test key revocation and reinstatement for each service. Confirm AI Search CMK latency impact is within acceptable SLA bounds. Confirm Fabric CMK revocation takes effect within the documented 60-minute window.

**Shared Private Link provisioning and approval workflow.** Validate the end-to-end sequence during the IaC spike: (a) PUT on the search service sharedPrivateLinkResources to create a private link to Key Vault (subresource: vault), (b) approve the pending private endpoint connection on the Key Vault side, (c) confirm the link transitions to Approved/Succeeded state, (d) verify AI Search can unwrap the CMK via the shared private link with Key Vault public access disabled. Document the provisioning sequence in IaC templates and include the approval step in the deployment pipeline. Reference: [https://learn.microsoft.com/en-us/azure/search/search-indexer-howto-access-private](https://learn.microsoft.com/en-us/azure/search/search-indexer-howto-access-private).

---

## Risks

|Risk|Likelihood|Impact|Mitigation|
|---|---|---|---|
|Key Vault unavailability renders all CMK-encrypted data inaccessible after service-specific cache windows expire. All CMK keys reside in a single vault, creating a shared dependency concentration point.|Low|High|Dependency managed through layered controls: service-specific key caches, 60 minutes for AI Search, daily for Fabric, provide short-term resilience; soft-delete and purge protection prevent accidental key destruction; Key Vault diagnostic logging enables monitoring of access failures. Key Vault provides zone and paired-region replication, but this does not eliminate the shared-vault dependency for the workload. If vault-level isolation per service is required, multiple Key Vaults can be deployed at additional operational cost.|
|Fabric CMK does not encrypt all workspace data. Unprotected categories include Lakehouse structural metadata, Spark cluster temporary data, job logs, environment libraries and session customisations, Pipeline and Copy job metadata, ML model and experiment metadata, and Warehouse Object Explorer queries and backend cache.|Medium|Medium|CMK-enabled workspaces must be treated as partial-coverage controls. If classification policy requires CMK coverage for metadata or operational artefacts, the gap must be accepted as a documented residual risk or mitigated through complementary controls, restricting metadata content and limiting log retention.|
|AI Search objects created without CMK cannot be retroactively encrypted; they must be deleted and recreated. Key loss, purge, or role assignment removal renders CMK-encrypted objects permanently unusable.|Low|High|Purge protection on Key Vault. IaC-enforced deployment sequencing ensures CMK is configured before first index creation. Documented key recovery procedures. Key Vault diagnostic logging monitors access patterns.|
|Foundry CMK auto-rotation is not documented as zero-touch. Manual key version update may be required after each Key Vault rotation event. Missed update means rotation policy is not fully enforced.|Medium|Medium|Event Grid notification on key rotation events triggers operational alert. Operational runbook for manual update. Validation during implementation spike, Open Item 1. If versionless key support is confirmed, this risk is eliminated.|
|Azure SQL CMK zero-touch rotation behaviour is not yet validated in the chosen implementation path. A tooling-specific gap could require explicit TDE protector update after key rotation.|Medium|Medium|Tracked as Open Item 6. Validate during implementation spike. If explicit update is required, document runbook and automate where possible.|
|Trusted services bypass on Key Vault grants access to all services on Microsoft's trusted services list, not only Storage, Fabric, Foundry, and Azure SQL Database. Services such as Azure Data Factory, Databricks, SQL Database, and Machine Learning are also on the list.|Low|Medium|RBAC is the primary control; the bypass alone grants no data-plane access. Only identities with the appropriate Key Vault role can perform wrap or unwrap operations. Regular audit of Key Vault role assignments ensures no unintended access.|
|Log Analytics MMK decision depends on the Container Apps workloads not logging PII or business data in telemetry. LLM-based systems are particularly vulnerable to accidental prompt and completion content appearing in trace payloads.|Medium|High|ADR-006 must enforce this as a binding rule with an enforcement mechanism. Automated scanning for PII patterns in telemetry output, code review checklists, and periodic telemetry audits reduce the risk. If the constraint is violated, the MMK decision must be revisited and the dedicated cluster escalation path activated.|
|CMK adds operational cost: AI Search encrypt or decrypt latency on every operation, Fabric CMK requires F SKU, Log Analytics CMK escalation requires dedicated cluster at 100 GB/day minimum.|Low|Medium|Costs documented in client engagements so pricing reflects the full data protection posture. Per-service CMK cost impact assessed during implementation.|
|Purview Protection Policies for Azure Storage do not work on CMK-enabled storage accounts, current preview limitation. If this persists to GA, the platform cannot apply both CMK and Purview content-aware access control on the same storage account.|Medium|Medium|Tracked as Open Item 4. If limitation persists, evaluate separating storage accounts by function or accepting one control without the other. Monitor Microsoft documentation for resolution.|
|Foundry CMK Key Vault role discrepancy between current documentation, Key Vault Crypto User, and older documentation, Key Vault Crypto Service Encryption User. Broader role assigned as provisional baseline may grant more permissions than required.|Low|Low|Tracked as Open Item 5. Follow newer documentation and AVM pattern, Key Vault Crypto User, for initial deployment. Validate minimum required role during implementation spike.|
|Queue payloads in Azure Service Bus may drift from transient coordination data toward business-significant content, weakening the MMK rationale.|Medium|Medium|Document payload boundary rules in application design and ADR-006 operational controls. If queue contents begin carrying sensitive extraction or approval data, revisit Service Bus CMK decision and payload design.|

---

## References

|Reference|URL|Relevance|
|---|---|---|
|Azure Key Vault overview|[https://learn.microsoft.com/en-us/azure/key-vault/general/overview](https://learn.microsoft.com/en-us/azure/key-vault/general/overview)|Key Vault Premium tier, FIPS 140-3 Level 3, HSM-backed keys|
|Key Vault network security|[https://learn.microsoft.com/en-us/azure/key-vault/general/network-security](https://learn.microsoft.com/en-us/azure/key-vault/general/network-security)|Firewall configuration, trusted services bypass, private endpoint|
|Key Vault trusted services list|[https://learn.microsoft.com/en-us/azure/key-vault/general/overview-vnet-service-endpoints](https://learn.microsoft.com/en-us/azure/key-vault/general/overview-vnet-service-endpoints)|Trusted services enumeration, scope of bypass|
|Key Vault auto-rotation|[https://learn.microsoft.com/en-us/azure/key-vault/keys/how-to-configure-key-rotation](https://learn.microsoft.com/en-us/azure/key-vault/keys/how-to-configure-key-rotation)|Versionless key URI, rotation policy, Event Grid notification|
|Key Vault reliability|[https://learn.microsoft.com/en-us/azure/reliability/reliability-key-vault](https://learn.microsoft.com/en-us/azure/reliability/reliability-key-vault)|Availability zone replication, paired-region replication|
|Key Vault Premium FIPS upgrade|[https://techcommunity.microsoft.com/blog/microsoft-security-blog/azure-managed-hsm-and-azure-key-vault-premium-are-now-fips-140-3-level-3/4418975](https://techcommunity.microsoft.com/blog/microsoft-security-blog/azure-managed-hsm-and-azure-key-vault-premium-are-now-fips-140-3-level-3/4418975)|June 2025 upgrade to FIPS 140-3 Level 3|
|Azure Storage CMK overview|[https://learn.microsoft.com/en-us/azure/storage/common/customer-managed-keys-overview](https://learn.microsoft.com/en-us/azure/storage/common/customer-managed-keys-overview)|Storage CMK configuration, versionless key support, identity requirements, auto-rotation|
|Azure Storage CMK configuration, existing account|[https://learn.microsoft.com/en-us/azure/storage/common/customer-managed-keys-configure-existing-account](https://learn.microsoft.com/en-us/azure/storage/common/customer-managed-keys-configure-existing-account)|SAMI vs UAMI constraint: new accounts require UAMI|
|Azure AI Search CMK|[https://learn.microsoft.com/en-us/azure/search/search-security-manage-encryption-keys](https://learn.microsoft.com/en-us/azure/search/search-security-manage-encryption-keys)|Per-object encryption, irreversibility, deployment sequencing, policy enforcement|
|Azure AI Search shared private link|[https://learn.microsoft.com/en-us/azure/search/search-indexer-howto-access-private](https://learn.microsoft.com/en-us/azure/search/search-indexer-howto-access-private)|Shared private link provisioning for AI Search outbound connectivity to Key Vault and other Azure resources|
|Azure AI Foundry CMK|[https://learn.microsoft.com/en-us/azure/foundry/concepts/encryption-keys-portal](https://learn.microsoft.com/en-us/azure/foundry/concepts/encryption-keys-portal)|Foundry CMK configuration, Key Vault networking requirements, Key Vault Crypto User role|
|Azure AI Foundry-classic CMK|[https://learn.microsoft.com/en-us/azure/foundry-classic/openai/encrypt-data-at-rest](https://learn.microsoft.com/en-us/azure/foundry-classic/openai/encrypt-data-at-rest)|Older Foundry CMK documentation specifying Key Vault Crypto Service Encryption User|
|Microsoft Fabric CMK|[https://learn.microsoft.com/en-us/fabric/security/workspace-customer-managed-keys](https://learn.microsoft.com/en-us/fabric/security/workspace-customer-managed-keys)|Workspace-level encryption, envelope encryption, Fabric Platform CMK service principal, versionless key requirement, trusted services bypass|
|Azure SQL transparent data encryption with customer-managed key|[https://learn.microsoft.com/en-us/azure/azure-sql/database/transparent-data-encryption-byok-overview](https://learn.microsoft.com/en-us/azure/azure-sql/database/transparent-data-encryption-byok-overview)|Azure SQL TDE with CMK overview and operating model|
|Managed identities for transparent data encryption with customer-managed key in Azure SQL|[https://learn.microsoft.com/en-us/azure/azure-sql/database/transparent-data-encryption-byok-identity](https://learn.microsoft.com/en-us/azure/azure-sql/database/transparent-data-encryption-byok-identity)|Azure SQL CMK identity model for Key Vault access|
|Configure your own key for encrypting Azure Service Bus data at rest|[https://learn.microsoft.com/en-us/azure/service-bus-messaging/configure-customer-managed-key](https://learn.microsoft.com/en-us/azure/service-bus-messaging/configure-customer-managed-key)|Service Bus Premium CMK capability and trade-off baseline|
|Azure Monitor dedicated cluster CMK|[https://learn.microsoft.com/en-us/azure/azure-monitor/logs/customer-managed-keys](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/customer-managed-keys)|Dedicated cluster requirement, minimum commitment tier, escalation path|
|Managed identity best practices|[https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/managed-identity-best-practice-recommendations](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/managed-identity-best-practice-recommendations)|UAMI vs SAMI selection guidance|
|Purview sensitivity labels in Fabric|[https://learn.microsoft.com/en-us/fabric/enterprise/powerbi/service-security-sensitivity-label-overview](https://learn.microsoft.com/en-us/fabric/enterprise/powerbi/service-security-sensitivity-label-overview)|Sensitivity label functionality in Power BI and Fabric|
|Purview Protection Policies for Azure Storage|[https://techcommunity.microsoft.com/blog/azurestorageblog/microsoft-purview-protection-policies-for-azure-data-lake--blob-storage-availabl/4382887](https://techcommunity.microsoft.com/blog/azurestorageblog/microsoft-purview-protection-policies-for-azure-data-lake--blob-storage-availabl/4382887)|Storage protection policies, CMK incompatibility limitation|
|AI Search query-time sensitivity label enforcement|[https://learn.microsoft.com/en-us/azure/search/search-query-sensitivity-labels](https://learn.microsoft.com/en-us/azure/search/search-query-sensitivity-labels)|Preview Purview integration for document-level access control|
|Azure Policy: Storage minimum TLS version|Policy ID: fe83a0eb-a853-422d-aff8-513bc498900d|TLS 1.2 enforcement on storage accounts|