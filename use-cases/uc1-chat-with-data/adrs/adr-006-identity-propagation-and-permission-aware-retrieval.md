# UC1-ADR-006: Identity Propagation and Permission-Aware Retrieval

**Status**: Draft  
**Date**: 01/04/2026  
**Decision Scope**: How end-user authorization propagates through channels, agent tooling, Fabric, and Azure AI Search.  
**Depends on**: ADR-004 (Identity, Authentication, and Authorisation), UC1-ADR-001, UC1-ADR-002, UC1-ADR-004  
**Depended on by**: UC1-ADR-007, implementation guidance for channel integration and retrieval adapters

---

## Context

UC1 is valuable only if it respects the same enterprise permissions that govern the underlying data. When a user asks a question, the system retrieves evidence from multiple sources. Azure AI Search indexes containing UC2 extraction outputs, and potentially Fabric lakehouses containing structured enterprise data. Each of these sources has its own permission model. If the conversational layer bypasses or flattens those permissions, the assistant becomes a privilege escalation vector: a user who cannot access a contract directly could ask the assistant about it and receive an answer derived from content they should not see.

Identity propagation is possible but not uniform across all retrieval paths. Fabric Data Agent honours underlying source permissions, including row-level security (RLS) and column-level security (CLS), and requires the calling identity to have underlying source access. Azure AI Search can enforce document-level permissions, but the mechanism depends on the source type, the indexer configuration, and in some cases preview features or query-time authorization headers that filter results based on the caller's identity. Foundry publication creates a new agent identity distinct from the project identity used during development, which means permissions that were correctly configured in the project context do not automatically transfer to the published agent.

Without an explicit identity-propagation decision, UC1 risks operating in one of two failure modes. In the first, the agent runs all retrieval under its own service identity, effectively granting every user the agent's access level which is typically broader than any individual user's. In the second, identity propagation works on some paths but not others, creating an inconsistent permission surface that is difficult to reason about, test, or audit. Both failure modes are incompatible with the enterprise governance requirements that justify UC1's existence.

---

## Decision Drivers

- End-user authorization must remain authoritative at retrieval time.
- Agent/application identity and end-user identity serve different purposes and must not be conflated.
- The architecture must support both Fabric and Azure AI Search retrieval patterns.
- Permission failures and degraded authorization behaviour must be observable.

---

## Considered Alternatives

### Option A: Agent identity only

Under this approach, all retrieval operations would execute under the published agent's service identity. The agent would authenticate to Azure AI Search, Fabric, and other data sources using its own managed identity or service principal credentials. End-user identity would not be propagated to the retrieval layer. This is the simplest integration pattern and avoids the complexity of on-behalf-of token flows, conditional identity routing, and per-channel identity behaviour. It is also the default behaviour when a Foundry agent is published without explicit identity-propagation configuration.

Trade-off: Simplest to implement and avoids per-channel identity complexity, but turns the assistant into a privileged reader that sees all data the agent identity can access, regardless of what the individual user is authorized to see. Makes row-level security, column-level security, and document-level permissions meaningless in the conversational context.

### Option B: End-user identity everywhere possible, with explicit fallback denial when not possible (SELECTED)

Under this approach, the architecture preserves end-user identity or an equivalent permission context through the retrieval path wherever the underlying product supports it. For Fabric Data Agent queries, this means the user's identity flows through to the lakehouse and RLS/CLS rules apply. For Azure AI Search queries, this means document-level permission filters are applied at query time based on the user's group membership or authorization context. Where a given retrieval path or delivery channel cannot preserve the required authorization semantics, for example, because the channel does not propagate user tokens or because the retrieval service lacks permission-aware query support, that path is explicitly constrained or disabled rather than silently falling back to agent-level access.

Trade-off: Preserves enterprise permission semantics at the conversational layer and prevents privilege escalation, but introduces complexity in channel integration, token handling, and retrieval-adapter design. Some channel paths may be unsuitable for production until their identity-propagation behaviour meets the required standard.

### Option C: Mixed implicit model

Under this approach, the system would use agent identity for some retrieval calls and user identity for others, with the decision made implicitly based on what each integration path supports. There would be no architecture-level rule governing which identity applies where. Developers would choose the path of least resistance for each integration point, resulting in a mix of user-authorized and agent-authorized retrieval that varies across data sources and delivery channels.

Trade-off: Fastest path to a working prototype with mixed data sources, but creates an inconsistent and opaque permission surface. It would be difficult to reason about what a given user can access through the assistant, difficult to test authorization boundaries, and difficult to explain to auditors which identity governed which retrieval operation.

---

## Decision

UC1 adopts an end-user-authoritative permission model.

End-user identity or user-derived authorization context is propagated through the retrieval path wherever the selected channel and retrieval service support it. For Fabric Data Agent queries, this means the user's identity must flow through to the underlying lakehouse so that RLS and CLS rules apply as they would in a direct Fabric query. For Azure AI Search queries, this means document-level permission filters must be applied at query time based on the user's authorization context, using whatever mechanism the search service supports for the configured source type.

Agent and application identity is treated as a service-access identity, it authenticates the application to the Azure service, but it does not substitute for user-level authorization decisions. The agent identity determines that the application is allowed to call the service; the user identity determines what data the service returns for that specific request.

If a delivery channel or retrieval path cannot preserve the required permission semantics, the architecture must explicitly constrain or disable that path rather than silently widening access. This means some channel-retrieval combinations may be unavailable in production until their identity-propagation behavior meets the required standard. The constraint is deliberate: it is preferable to deny access on an unsupported path than to grant broader access than the user is entitled to.

---

## Consequences

### Positive

- Preserves least privilege at the conversational layer: a user sees only what they would see through direct access to the underlying data source.
- Keeps Fabric RLS/CLS and Azure AI Search document-level permissions meaningful and enforceable in the conversational context.
- Makes permission failures and denial paths architecture-visible rather than hidden inside adapter implementations.
- Provides a clear audit trail of which identity governed each retrieval operation (see UC1-ADR-007).

### Negative

- Increases complexity in channel integration and retrieval-adapter design, particularly for on-behalf-of token flows and per-channel identity behavior.
- Some delivery channels or retrieval paths may be unsuitable for production use until their identity-propagation behavior is validated, limiting initial rollout scope.
- Published-agent identity changes (which occur when a Foundry agent is published to a new environment) must be explicitly handled, as permissions configured in the project context do not automatically transfer.

### Constraints introduced

- Permission propagation must be an explicit design concern in every retrieval adapter, not an afterthought. Each adapter must document which identity governs its retrieval calls and how that identity is obtained.
- Published-agent identity changes after Foundry publication must be explicitly handled. The architecture cannot assume that project-time permissions carry over to the published agent.
- UC1 must log permission-denial and degraded-authorization outcomes as first-class operational events in the audit trail (UC1-ADR-007), not as silent failures.
- Retrieval paths that cannot preserve user-level authorization must be explicitly disabled or constrained in the architecture, not silently allowed with agent-level access.
- The Copilot Studio → Foundry integration must use OBO authentication to propagate the user's identity. Document the specific Entra ID app registration and scope configuration required. Flag this as a blocking validation item.

---

## Follow-ups

- Define concrete identity-flow diagrams during implementation.
- Validate which channel paths preserve user identity strongly enough for production use.

---

## Identity Propagation Validation Status (CA-4)

> **The end-to-end user-token propagation mechanism through the chosen channel → agent → retrieval adapter → Azure AI Search path has not been validated. The following questions must be answered before this ADR can move from Draft to Accepted:
>
> 1. **Teams → Foundry Agent Application**: Does the Bot Service publication path pass the user's Azure AD token to the Foundry Agent Application in a form that supports On-Behalf-Of (OBO) delegation? Or does the Agent Application only receive a channel-level identity?
>
> 2. **Foundry Agent → Retrieval Adapter**: Can the retrieval adapter (custom tool/function) receive and forward the user's authorization token to Azure AI Search via the `x-ms-query-source-authorization` header? This header is required for document-level ACL/RBAC enforcement.
>
> 3. **Foundry Agent → Fabric**: Can the Fabric structured-query adjunct execute queries under the user's delegated identity (OBO) so that RLS/CLS applies, or does it execute under the agent's managed identity?
>
> 4. **M365 Copilot → Agent**: Does the M365 Copilot orchestrator preserve the user's authorization token when forwarding requests to the Foundry Agent Application, or does it substitute its own identity?
>
> **If user-token passthrough is not feasible** for one or more paths, the fallback security model is: (a) the agent executes retrieval under its own managed identity, (b) application-level security filtering is applied using group membership claims from the user's session token, and (c) the retrieval adapter populates security filter fields on the search query based on the user's group memberships rather than relying on Azure AD token-based ACL enforcement. This fallback provides weaker security guarantees than true user-token passthrough but is implementable with GA components.
>
> Per-path identity validation results should be documented in the reference architecture once testing is complete.
