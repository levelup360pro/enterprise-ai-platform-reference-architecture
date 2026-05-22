# ADR-004: Identity, Authentication, and Authorisation

**Status**: Accepted  
**Date**: 08/05/2026  
**Decision Scope**: Identity primitives, authentication model, authorisation strategy, and mediation boundaries for platform workloads, the AI Hub, and shared AI capability consumers  
**Depends on**: ADR-001 (Processing Paradigm), ADR-002 (Runtime Platform and EU Region Selection), ADR-003 (Network Isolation and Data Residency Enforcement)  
**Depended on by**: ADR-005 (Data Protection), ADR-006 (Observability and Operational Model), ADR-009 (Publication Model for Shared Models, Tools, and Reusable Agents), ADR-010 (MCP Exposure Through the AI Hub), ADR-011 (Agent-to-Agent Discovery and Invocation Through the AI Hub)  

---

## Context

ADR-003 established the private connectivity baseline. The platform now also includes an AI Hub with APIM as AI Gateway. That changes the identity problem materially. The platform no longer has only human users and private workload services. It now has multiple classes of consumers, multiple shared capability contracts, and a mediation boundary that must distinguish consumer identity from backend execution identity.

The identity model must answer five questions clearly:

1. How human users authenticate to platform applications and conversational channels.
2. How workload-internal services authenticate to private Azure backends.
3. How shared AI consumers authenticate to the AI Hub.
4. How APIM authenticates to backend model, tool, and agent services.
5. How identity context is preserved where a downstream decision requires it, without turning backend authorisation into uncontrolled credential passthrough.

---

## Decision Drivers

- **Zero Trust baseline**: Every call path needs explicit authentication and least-privilege authorisation.
- **Shared capability isolation**: Shared AI consumers need distinct quota, audit, and revocation boundaries.
- **Private backend posture**: Backend services remain private and should not depend on public shared secrets as their primary trust model.
- **Protocol coverage**: The identity model must support API, MCP, and A2A patterns without changing the core trust model each time.
- **Operational clarity**: Consumer identity, gateway identity, and backend identity must be separable so failures and audit events are attributable.
- **Bounded propagation**: User or caller context should be propagated only where required by the contract, not as the default trust mechanism for every downstream hop.

---

## Considered Alternatives

### Option A: Direct consumer identity to every backend service

Each consumer authenticates directly to Foundry, tool services, and shared agents. APIM is bypassed or used only as a thin proxy.

**Why considered**: maximises end-to-end identity purity and minimises mediation logic.

**Why not chosen**: it destroys the AI Hub contract boundary. Consumers become coupled to backend identities, backend role assignments proliferate, and quota or onboarding control moves away from the gateway. This is the wrong model for shared platform capabilities.

### Option B: Shared secrets or backend API keys as the primary trust model

Consumers or intermediary services call backend services using shared keys or connection strings.

**Why considered**: simple to implement for prototypes and some legacy integrations.

**Why not chosen**: weak revocation boundaries, poor attribution, key-sprawl risk, and weak alignment to the platform's managed-identity-first posture.

### Option C: Layered identity model with APIM as the shared mediation boundary (SELECTED)

Human users authenticate to application or conversational entry points. Workload-internal services use managed identities for direct private backend access where the capability is not shared. Shared AI consumers authenticate to APIM. APIM enforces the consumer contract and authenticates to backend services using managed identity or approved credential mediation. Identity context is propagated only where the downstream decision requires it.

**Why chosen**: keeps the AI Hub boundary real, preserves backend privacy, supports per-consumer isolation, and works across API, MCP, and A2A without making backend services depend on the consumer's credential shape.

---

## Decision

Adopt a layered identity model.

Human users authenticate through Microsoft Entra ID to platform applications and conversational channels. Workload-internal platform services use managed identities for direct private backend access when the capability remains internal to that workload. Shared AI consumers authenticate to APIM through a Product/subscription boundary, with JWT validation layered in where the consumer has its own workload identity or where a stronger caller attestation model is required.

APIM is the backend caller for shared capabilities. It authenticates to Foundry, tool services, and reusable agent endpoints using managed identity or an approved backend credential mediation path. Consumer subscription keys are not reused as backend credentials.

Caller identity context is propagated only when the contract requires downstream policy evaluation, audit correlation, or permission-aware behaviour. Even in those cases, propagation is bounded and explicit. Backend authorisation does not depend on uncontrolled caller passthrough as the default trust model.

---

## Decision Details

### 1. Principal classes are explicit

The platform recognises four principal classes:

- **Human principals**: enterprise users authenticating through Entra ID to direct applications or conversational channels.
- **Workload principals**: managed identities used by workload-local services such as frontend, API, worker, indexer, or job hosts.
- **Gateway principal**: the APIM managed identity used to authenticate from the AI Hub to backend shared services.
- **Machine consumer principals**: application identities or agent identities that consume shared AI capabilities through APIM.

Each class has different authorisation boundaries and should be observable independently.

### 2. Shared AI consumer authentication terminates at APIM

For shared model, tool, and reusable-agent consumption, APIM is the consumer authentication boundary.

The standard pattern is:

- Product/subscription for consumer isolation and revocation
- JWT validation where the consumer has an Entra-managed application or agent identity
- policy-based enforcement of which consumer can access which published capability

This makes the Product the operational onboarding and offboarding unit.

### 3. APIM is the backend caller for shared capabilities

For shared AI capabilities, APIM authenticates to the backend service. Backend services trust APIM's managed identity or approved credential mediation path, not the raw consumer subscription key.

This applies to:

- Foundry model and agent backends
- approved tool APIs
- MCP backends
- A2A agent runtimes

This keeps backend trust relationships stable even as consumers change.

### 4. Direct workload-private backend access remains allowed

The AI Hub is the preferred boundary for shared capabilities, not for every internal call. A workload can continue to call private Foundry, AI Search, Storage, SQL, Service Bus, or other platform services directly with managed identity where the capability remains internal to that workload and is not being published through the AI Hub.

### 5. Identity context propagation is bounded

Some downstream decisions require caller context. Examples include permission-aware retrieval, user-scoped policy enforcement, or caller-specific audit correlation.

In those cases, the platform may propagate identity context through APIM in a controlled form, such as validated claims or an approved token-forwarding pattern. This is an explicit contract choice. It is not the default for every shared-capability invocation.

### 6. A2A identity separates caller attestation from backend trust

For A2A, the calling agent authenticates to APIM as a consumer. APIM enforces the shared contract and authenticates to the target agent backend. If the target agent needs to know who the caller was, APIM propagates that identity context explicitly. The target backend still trusts APIM as the network and execution caller.


## CMK Identity Inventory and Key Vault RBAC Assignments

ADR-005 owns the encryption rationale and per-service CMK constraints. ADR-004 is the authoritative location for the identities and Key Vault RBAC assignments used by that CMK model.

### CMK Identity Inventory

| Service | Identity | Purpose |
| --- | --- | --- |
| Azure Storage (ADLS Gen2) | `uami-storage-cmk` | Accesses the storage CMK in Key Vault when the storage account is provisioned with CMK at creation time. |
| Azure AI Search | `sami-search` | Accesses Key Vault for AI Search CMK operations through the shared private link path. |
| Azure AI Foundry | Foundry system-assigned managed identity (preferred) or approved dedicated UAMI | Accesses the CMK for the Foundry resource. |
| Microsoft Fabric | Fabric Platform CMK service principal | Accesses the Fabric workspace CMK in Key Vault. |
| Azure SQL Database | Azure SQL server managed identity or approved dedicated UAMI | Accesses the TDE protector in Key Vault. |

### Key Vault RBAC Assignments for CMK

| Identity | Key Vault role | Notes |
| --- | --- | --- |
| `uami-storage-cmk` | Key Vault Crypto Service Encryption User | Required for Azure Storage CMK operations. |
| `sami-search` | Key Vault Crypto Service Encryption User | Required for AI Search CMK operations. |
| Foundry CMK identity | Key Vault Crypto User | Current documented baseline for Foundry CMK; validate whether the narrower service-encryption role is sufficient before tightening. |
| Fabric Platform CMK service principal | Key Vault Crypto Service Encryption User | Required for Fabric workspace CMK. |
| Azure SQL CMK identity | Key Vault Crypto Service Encryption User | Required for Azure SQL TDE with CMK. |

---

## Consequences

### Positive

- Clear separation between consumer identity and backend execution identity.
- Stronger per-consumer revocation, quota, and audit boundaries through APIM Products and subscriptions.
- Consistent identity model across API, MCP, and A2A.
- Backend role assignments stay bounded to platform-managed identities rather than every consumer.
- Direct workload-private access remains available where mediation adds no value.

### Negative

- More identity concepts to document and operate than a direct-call model.
- Some integrations require both subscription and JWT handling, which teams must understand correctly.
- Explicit propagation contracts are needed where downstream behaviour depends on caller context.

---

## Constraints

- Managed identities are the default for service-to-service access.
- Shared capability consumers must not receive direct ad hoc backend role assignments as the normal integration model.
- Subscription-scoped onboarding is the baseline for shared AI capability consumption.
- Caller context propagation must be explicit and contract-bound.
- Backend services remain private. The identity model must not force public exposure to satisfy consumer access.

---

## Risks and Mitigations

|Risk|Likelihood|Impact|Mitigation|
|---|---|---|---|
|Teams treat subscription keys as sufficient and skip stronger caller validation where it is required.|Medium|High|Define per-consumer onboarding templates that state when subscription-only is acceptable and when JWT validation is mandatory.|
|Backend services start to depend on direct consumer identities, weakening the AI Hub boundary.|Medium|High|Keep APIM as the required backend caller for shared capabilities and review new backend role assignments as an architecture concern.|
|Overuse of identity passthrough recreates tight coupling between consumers and backends.|Medium|Medium|Treat propagation as an exception that must be justified by contract requirements such as permission-aware retrieval or caller-specific audit needs.|
|Different protocol teams implement inconsistent authentication patterns.|Medium|Medium|Anchor API, MCP, and A2A exposure to the same layered identity model and document the protocol-specific differences in ADR-010 and ADR-011.|

---

## Follow-ups

- Define the standard onboarding template for shared AI consumers, including Product, subscription, JWT requirements, and expected observability fields.
- Validate the exact minimum RBAC role set APIM requires on Foundry for shared model and reusable-agent invocation in the target deployment mode.
- Validate the identity propagation pattern for scenarios that require caller-aware downstream policy enforcement, especially where Copilot Studio, APIM, and Foundry-hosted reusable agents are chained.
