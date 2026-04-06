# UC1-ADR-003: Delivery Channel Selection

**Status**: Draft  
**Date**: 31/03/2026  
**Decision Scope**: Which end-user channels UC1 should support now versus later, under what readiness conditions, and which integration pattern connects Copilot Studio to the Foundry Agent Service orchestration layer.  
**Depends on**: UC1-ADR-001 (Orchestration Layer Selection), ADR-003 (Network Isolation and Data Residency Enforcement), ADR-004 (Identity, Authentication, and Authorisation)  
**Depended on by**: UC1-ADR-005 (Response Grounding), UC1-ADR-006 (Identity Propagation), UC1-ADR-007 (Operational State and Audit Trail)  

---

## Context

The business-facing channels for UC1 are expected to be Microsoft Teams and Microsoft 365 Copilot. These are attractive because they place the assistant in tools users already use daily, eliminating the friction of a separate application and increasing discoverability. However, the maturity and governance characteristics of those channels vary materially by backend option and by integration pattern, and those differences are architecture-relevant, not merely UX details.

ADR-001 selects Microsoft Foundry Agent Service as the orchestration layer and positions Copilot Studio as the conversational front-end. This creates a two-layer delivery architecture: Copilot Studio owns the user-facing conversation and channel publication, while Foundry Agent Service owns orchestration, retrieval, and response generation. The method by which Copilot Studio calls the Foundry agent is a first-order architectural decision, not an implementation detail, because it determines whether citations, identity, and network isolation survive the boundary between the two layers.

The native Copilot Studio Foundry connector (the "Connected Agents" feature) is in preview and cannot process Foundry agents that have tools or knowledge sources attached. Only bare Foundry agents with no tools, no Foundry IQ knowledge base, and no file search return responses through the connector. Foundry agents with tools or knowledge sources cause all calls to fail. This is a documented preview limitation reported on the Power Platform community in December 2025 and referenced in the Azure Copilot Studio and Foundry labs repository. No publicly confirmed resolution date exists as of March 2026. Because ADR-001 mandates that the Foundry agent includes retrieval adapters (tools) connecting to Azure AI Search and Fabric, the native connector path is blocked for UC1.

This constraint elevates the integration pattern from a wiring concern to a delivery-channel decision. The choice of integration pattern directly affects citation behaviour, identity propagation, network isolation, and operational governance, all of which are channel-readiness criteria for a regulated environment.

Beyond the integration constraint, channel-level limitations affect UC1's value proposition. Foundry Agent Service published agents do not support citations when published directly via Bot Service. The Bot Service publication path does not support Private Link. The M365 Copilot channel applies URL sanitisation that may remove or downgrade embedded URLs to plain text, particularly when the agent is invoked via @mention. The text content of the agent's response is not rewritten or paraphrased by the M365 Copilot orchestrator. Unsupported message types (basic cards, video, image, file) are rejected with a ContentFiltered error rather than silently modified. Citations from generative answers should use the citation section rather than inline URLs.

Each of these limitations affects the core value proposition of an evidence-grounded enterprise assistant: if citations are suppressed or their URLs stripped, users cannot follow references to source documents; if Private Link is unavailable, the network posture weakens.

For a regulated, evidence-sensitive use case like UC1, both channel selection and integration pattern selection are governance decisions. The architecture must avoid a situation where the desire for early channel ubiquity or integration simplicity forces the team to accept weaker source attribution, weaker network isolation, or dependence on preview behaviour that may change. At the same time, the architecture must not indefinitely withhold channel access from users who need the assistant in their daily workflow.

---

## Decision Drivers

- Users want the assistant in familiar Microsoft channels. UC1 cannot weaken source attribution or network/governance posture just to gain early channel ubiquity.
- Teams and Microsoft 365 Copilot do not currently provide equivalent capability across all UC1 backend patterns. The maturity gap between the two channels is architecture-relevant.
- The native Copilot Studio Foundry connector is blocked for agents with tools or knowledge sources, requiring an alternative integration path that preserves citation fidelity, identity propagation, and network isolation end-to-end.
- Channel rollout should be reversible and governed, with explicit readiness criteria evaluated per channel rather than assumed from product announcements.
- The integration pattern must preserve citation fidelity, identity propagation, and network isolation end-to-end. These are non-negotiable for a regulated, evidence-sensitive use case.

---

## Considered Alternatives

### Channel Alternatives

#### Option A: Microsoft 365 Copilot first

Microsoft 365 Copilot would be the primary and initial delivery channel for UC1. The assistant would appear as an agent within the M365 Copilot experience, benefiting from strong discoverability across the Microsoft 365 surface area and alignment with Microsoft's broader agent-ecosystem vision. Users would encounter the assistant in the same context where they already work with documents, email, and collaboration tools.

However, the M365 Copilot channel has specific documented limitations that affect UC1. The channel may remove or downgrade embedded URLs in agent responses, which affects citation links but not citation text. Additional limitations include: no Conversation Start topic support, GIF images not rendered, Action.Execute Adaptive Cards not supported, basic cards and video and image and file message types return ContentFiltered errors, and user feedback reactions are not supported. The @mention pipeline applies stricter output sanitisation that may remove, hide, or downgrade URLs to plain text. For a regulated, evidence-sensitive use case, M365 Copilot's URL sanitisation behaviour must be validated against UC1's citation format before production deployment.

Trade-off: Maximum discoverability and alignment with Microsoft's agent vision, but URL sanitisation behaviour requires validation to confirm citation links survive, and several message-type limitations constrain response formatting.

#### Option B: Teams only

Microsoft Teams would be the sole delivery channel for UC1. This is a more practical starting point for early pilots because Teams supports some backend patterns that are not currently available in M365 Copilot, and the team has more control over the conversational experience within the Teams bot framework. Teams is already the primary collaboration tool for the target user population, so discoverability within Teams is sufficient for initial adoption.

However, restricting to Teams as the final enterprise channel strategy is too narrow. It risks making the long-term architecture look more limited than intended and may create user-experience fragmentation if other enterprise assistants are delivered through M365 Copilot. A Teams-only commitment also does not address the strategic goal of placing the assistant wherever users naturally work across the Microsoft 365 surface.

Trade-off: More controlled and practical for early regulated pilots, but too narrow as a permanent channel strategy and risks positioning UC1 as a limited Teams-only tool.

#### Option C: Phased dual-channel strategy (SELECTED)

Both Teams and Microsoft 365 Copilot would be target channels, but rollout would be phased based on explicit readiness criteria rather than simultaneous. Teams would serve as the preferred pilot channel where an early conversational rollout is needed, because its channel characteristics are better understood and more controllable for a regulated use case. Microsoft 365 Copilot would be treated as a strategic target-state channel, with go-live gated on documented resolution of current limitations around URL sanitisation, identity flow, and network posture.

This approach recognises that both channels are desirable but not equally ready, and that channel readiness is a governance question that should be evaluated against explicit criteria rather than assumed from product announcements. The phased model allows the team to deliver immediate value through Teams while preserving M365 Copilot as a governed future milestone. The cost is that stakeholders must accept phased rollout rather than immediate channel parity, and the channel matrix must be tracked and re-evaluated as Microsoft's channel capabilities evolve.

Trade-off: Delivers immediate value through Teams while preserving M365 Copilot as a governed future target, but requires stakeholder acceptance of phased rollout and ongoing channel-readiness assessment.

### Integration Pattern Alternatives

#### Pattern 1a: Copilot Studio Native Connector to Foundry Agent Service

Copilot Studio connects to the Foundry agent using the built-in Connected Agents feature. The maker provides the Foundry project endpoint URL and Agent ID. Copilot Studio's generative orchestrator decides when to route a user turn to the Foundry agent based on the agent's name and description metadata. Same-tenant requirement applies: Copilot Studio and the Foundry project must be in the same Microsoft Entra ID tenant.

Current state as of March 2026: Preview. The connector cannot process Foundry agents that have tools or knowledge sources attached. Only bare Foundry agents return responses. This limitation was reported on the Power Platform community in December 2025 and is referenced in the Azure Copilot Studio and Foundry labs repository. No resolution date confirmed. Because ADR-001 mandates that the Foundry agent includes retrieval adapters (tools) connecting to Azure AI Search and Fabric, this pattern is blocked for UC1.

Citation behaviour: Not pass-through. Foundry API annotations are not rendered as Copilot Studio citations. Identity propagation: Unvalidated. No documentation confirms user token propagation to Foundry via the native connector. Private Link: Unvalidated. No documentation confirms VNet routing for native connector calls.

Not viable for Phase 1. Track for re-evaluation at GA.

#### Pattern 1b: Copilot Studio HTTP Request or Power Automate to Foundry Agent Service (SELECTED)

Copilot Studio topics include an HTTP Request node or trigger a Power Automate flow that calls the Foundry Agent Service REST API directly. The topic manages the multi-step API interaction: Create Thread, Create Run, poll or wait for completion, List Messages, parse the response including annotations, and render the result within the Copilot Studio conversation. Thread IDs are stored in Copilot Studio conversation variables to maintain multi-turn state.

**Sub-options evaluated:**

**HTTP Request node (selected).** Native Copilot Studio capability. No additional licensing. Full control over headers including Authorization for OBO. Requires JSON schema definition for response parsing. Respects Power Platform VNet when the environment is configured as a Managed Environment with Virtual Network support. Selected as the primary sub-option because it provides the strongest network isolation posture and the simplest dependency chain.

**Power Automate flow.** Uses the Azure AI Foundry Agent Service connector (Premium tier). Supports Create Run, Create Thread, Invoke Agent, List Agents, List Messages actions. Throttle limit of 1,000 API calls per connection per 60 seconds. Adds a Power Automate dependency and licensing requirement but simplifies error handling and retry logic. Viable alternative if topic-level JSON parsing proves insufficient during Proof 1.

**MCP server integration (not selected).** Copilot Studio MCP support reached GA in May 2025 with Streamable HTTP transport, OAuth 2.0 with OBO, and dynamic tool discovery. A custom MCP server wrapping the Foundry API would absorb payload changes in server code rather than Copilot Studio topics. However, MCP servers must be publicly reachable over HTTPS because Copilot Studio services run outside customer virtual networks. This conflicts with UC1's network isolation requirement. The Foundry API follows the OpenAI Assistants API contract with pinned API versions; payload changes are opt-in and manageable through JSON schema updates in the Copilot Studio topic. For a single-consumer integration, the infrastructure overhead (custom server, APIM, WAF, IP filtering) is not justified. Becomes relevant if multiple front-ends need a shared Foundry integration layer or if the organisation already operates APIM.

**Pattern 1b characteristics (HTTP Request sub-option):**

Citation behaviour: Viable with engineering. Parse Foundry API file_citation annotations or use system-prompt citation formatting in the Foundry agent's response body.

Identity propagation: Architecturally feasible. OBO supported for custom connectors in Copilot Studio. User token passable in HTTP Authorization header.

Private Link: Likely viable. HTTP Request node respects VNet in Managed Environment, documented and tested for Azure Key Vault private endpoints. Same mechanism applies to any HTTP endpoint reachable from the delegated subnet.

### Pattern 2: Direct Foundry Agent to Teams and M365 via Azure Bot Service

The Foundry agent is published directly to Teams and M365 Copilot through Azure Bot Service channel registration, bypassing Copilot Studio entirely. The Foundry agent handles conversation management, orchestration, and response rendering.

Citation behaviour: Limited. Bot Service does not natively render Foundry annotations as Teams Adaptive Card citations. Identity: Manual. Must implement Entra ID auth and OBO in agent code. Private Link: Supported for the Foundry and Bot Service runtime but not for the Bot Service publication channel path. Teams channel: Supported via standard Bot Framework channel registration. M365 Copilot: Limited. Not a native M365 Copilot declarative agent without additional manifest configuration. Distribution: Manual. No Teams app store "Built for your org" path without Copilot Studio.

Fallback for organisations that cannot use Copilot Studio. Full orchestration control but loses the low-code front-end, native M365 Copilot agent experience, sensitivity label support, DLP policy governance, maker audit logs, and organisational distribution through the Teams app store.

### Pattern 3: Custom Web Application to Foundry Agent Service API

A custom web application calls the Foundry Agent Service API directly. The application handles authentication, conversation rendering, citation display, and all UX concerns.

Citation behaviour: Full control. Identity: Full control via standard Entra ID auth flows. Private Link: Supported via App Service with Private Endpoint to Foundry. Teams channel: Requires embedding as a Teams tab or messaging extension, not a native conversational agent. M365 Copilot: Not native. Cannot surface as M365 Copilot declarative agent.

Maximum control, minimum discoverability. Suitable for specific use cases where Teams and M365 integration is not required or where custom UX requirements exceed what Copilot Studio can deliver.

### Pattern 4: Copilot Studio Direct to AI Search without Foundry

Copilot Studio connects to Azure AI Search as a native knowledge source without Foundry Agent Service in the middle. This eliminates the Foundry integration complexity entirely.

Rejected. This bypasses the orchestration layer selected in ADR-001. The Foundry agent provides retrieval adapter abstraction (UC1-ADR-004), cross-plane query coordination covering indexed knowledge and structured analytics, governance metadata qualification (UC1-ADR-008), and the system prompt controls that enforce response grounding rules (UC1-ADR-005). Connecting Copilot Studio directly to AI Search would require reimplementing all of these capabilities in Copilot Studio topics, which is not feasible at enterprise scale and would create a maintenance burden. However, this pattern is documented as a diagnostic fallback: if the Foundry Agent Service is unavailable, Copilot Studio can fall back to direct AI Search queries for degraded-mode operation.

---

## Decision

UC1 adopts a phased dual-channel strategy that treats channel rollout as a governed deployment decision rather than an automatic consequence of having a conversational agent. UC1 adopts Pattern 1b (Copilot Studio with HTTP Request or Power Automate calling Foundry Agent Service) as the primary integration pattern for connecting the conversational front-end to the orchestration layer.

Near term: Teams is the preferred pilot channel where an early conversational rollout is needed. Teams provides sufficient discoverability for initial adoption, supports the backend patterns UC1 requires, and gives the team more control over the end-to-end experience including citation presentation and identity flow. The Copilot Studio agent calls the Foundry Agent Service via HTTP Request node, with the user's OBO token in the Authorization header, thread IDs persisted in conversation variables for multi-turn context, and response parsing that extracts the Foundry agent's text and citation references for rendering.

M365 Copilot early validation: The M365 Copilot channel is enabled in parallel with the Teams pilot for the same pilot user group. This validates M365 Copilot-specific limitations (no Conversation Start topic, URL sanitisation, unsupported message types) against actual UC1 response patterns without committing to production rollout.

Target state: Both Teams and Microsoft 365 Copilot are supported as production channels. The architecture is designed to be channel-agnostic at the agent orchestration layer (see UC1-ADR-001), so extending to M365 Copilot does not require rearchitecting the retrieval or grounding logic.

Integration pattern target state: When the native Copilot Studio Foundry connector (Pattern 1a) exits preview and meets all tracking criteria documented in the Monitoring the Native Connector section, evaluate migration from HTTP Request to native connector. This is a configuration change, not an architecture change.

Gate condition: Broader Microsoft 365 Copilot rollout for a given backend pattern happens only when its documented channel limitations are acceptable for the regulated use case. Specifically, the gate criteria include: citation text and citation URLs are preserved end-to-end without sanitisation or removal, identity flow supports the required governance posture, and network isolation (Private Link) is available if required by the deployment context. Promotion of this ADR from Proposed to Accepted requires completing the four validation proofs documented in the Validation Proofs section.

---

## Channel Characteristics Under Pattern 1b

Because Pattern 1b uses Copilot Studio as the front-end, the channel characteristics are determined by Copilot Studio's publish capabilities, not by the Foundry Agent Service directly.

### Teams Channel

Authentication: Automatic Entra ID with no manual setup required when published to Teams. Markdown: Partially supported. Multiple-choice options: Up to six as hero card. Welcome message: Supported via Conversation Start topic. Adaptive Cards: Supported. Sensitivity labels: Displayed for SharePoint knowledge sources; custom rendering needed for AI Search sources. Rate limiting: Teams applies rate limiting to agent messages. Group chats and channels: Supported, but knowledge sources requiring end-user authentication are restricted to one-to-one chat only. Distribution: Installation link, "Built with Power Platform" section for shared users, "Built for your org" section for admin-approved agents. App setup policies: Admin can auto-install and pin the agent for targeted user groups.

### M365 Copilot Channel

Activation: "Make agent available in Microsoft 365 Copilot" toggle when adding the Teams channel. User invocation: @mention in M365 Copilot chat. Conversation Start: Not supported; agent offers suggested prompts instead. Media types: GIFs not rendered; images only via Adaptive Cards. Embedded URLs: M365 Copilot may remove embedded URLs for security; the @mention pipeline applies stricter output sanitisation that may remove, hide, or downgrade URLs to plain text. Workarounds: use Markdown link formatting, return URLs in structured JSON fields, or provide navigational text instead of bare URLs. Citations from generative answers should include URLs in the citation section rather than inline. Unsupported node types: Speech, hand-off to agent, Action.Execute Adaptive Cards. Unsupported message types: Basic cards, video, image, and file return ContentFiltered errors (these are rejected, not silently modified). User feedback: Reactions not supported. Distribution: "Built by your org" section in M365 Agent Store for admin-approved agents. Response text fidelity: The M365 Copilot orchestrator does not rewrite or paraphrase the text content of agent responses. The documented modifications are limited to URL sanitisation and unsupported message type rejection.

### Phased Channel Rollout

Phase 1: Teams one-to-one chat for a pilot user group of fewer than 50 users, validating integration, citations, identity, and Private Link. M365 Copilot enabled for the same pilot group to validate M365 Copilot-specific limitations in parallel.

Phase 2: Teams org-wide via admin-approved distribution after pilot validation. M365 Copilot org-wide via Agent Store; decision to expand or restrict based on Phase 1 findings.

Phase 3: Custom web application only if specific UX requirements emerge that Copilot Studio cannot meet.

---

## Citation Strategy Under Pattern 1b

Copilot Studio does not automatically pass through Foundry Agent Service API annotations as rendered citations. Two approaches are viable.

### Option A: System-Prompt Citation Formatting (Phase 1)

The Foundry agent's system prompt instructs the model to include source references in the response text body. The instruction directs the model to cite the source document for each statement of fact, list each source at the end of the response with document name and chunk reference, and prefix citations with an UNREVIEWED marker if governance metadata indicates review_state is not approved.

Copilot Studio renders the Foundry agent's text response as-is, preserving the citation text. This approach is simple, requires no response parsing engineering, and works immediately. Because M365 Copilot's URL sanitisation targets embedded URLs but does not modify plain text, citations formatted as document names and chunk references (without bare URLs) will survive both the Teams and M365 Copilot channels. If citation URLs are needed in M365 Copilot, they should use Markdown link formatting rather than bare URLs.

Trade-off: Citations are text-based. Clickable deep-links to source documents require Markdown formatting and are subject to M365 Copilot URL sanitisation rules. Adequate for Phase 1 pilot; may need upgrading for Phase 2.

### Option B: Annotation Parsing (Target for Phase 2)

The HTTP Request node receives the full Foundry API response including file_citation annotations. A Power Automate flow or Copilot Studio Power Fx expression parses the annotations, extracts document references and chunk locations, and formats them as Adaptive Card elements with clickable links to the source documents in SharePoint, Blob Storage, or the document management system. In the M365 Copilot channel, Adaptive Cards can display images (a documented workaround for unsupported media types), and structured content within Adaptive Cards is less aggressively sanitised than natural language text.

Trade-off: More engineering effort. Richer UX. Requires that the Foundry agent returns structured annotations, validated in Proof 4 of the validation proofs.

Phase 1 uses Option A. Option B feasibility is validated during Phase 1 pilot. Option B is adopted in Phase 2 if validation succeeds.

---

## Identity Flow Under Pattern 1b

ADR-006 requires the calling user's Entra ID identity to propagate through to Azure AI Search for document-level security trimming. Under Pattern 1b the flow is: User authenticates via Teams or M365 Copilot with automatic Entra ID, Copilot Studio agent obtains the user's token, HTTP Request node passes the OBO token in the Authorization header to the Foundry Agent Service, Foundry agent uses the delegated token for downstream calls to Azure AI Search which applies security-trimmed queries based on the user's identity.

OBO configuration requires: a connector app registration in Entra ID representing the Copilot Studio HTTP integration with single-tenant supported account types and delegated permissions for the Foundry Agent Service scope; an exposed API on the connector app registration with a custom scope such as access_as_user; authorisation of the Azure API Connections service principal with client ID fe053c5f-3692-4f14-aef2-ee34fc081cae; Enable on-behalf-of login set to true in the connector authentication configuration; and the Foundry Agent Service configured to accept the delegated token and use it for downstream Azure AI Search calls where the Foundry agent's managed identity holds Search Index Data Reader and the user's identity via OBO provides the security filter context.

This configuration is architecturally sound based on documented OBO support for custom connectors in Copilot Studio. It has not been validated end-to-end for the specific Foundry Agent Service API. Validation is required as Proof 2 in the validation proofs.

---

## Network Isolation Under Pattern 1b

The UC1 security model requires all data-plane traffic to remain within the private network. Under Pattern 1b the VNet path is: Copilot Studio operating in a Managed Environment with VNet enabled routes the HTTP Request through the delegated subnet into the Azure VNet, which contains private endpoints for the Foundry Agent Service, Azure AI Search, Azure OpenAI, and the Storage Account.

Prerequisites: Power Platform environment configured as a Managed Environment; Virtual Network support enabled with subnet delegation to Microsoft.PowerPlatform/enterprisePolicies; Foundry Agent Service private endpoint deployed in the same VNet or a peered VNet accessible from the delegated subnet; all downstream services behind private endpoints in the same network topology.

Copilot Studio HTTP Request nodes route through the delegated VNet when the environment has VNet support enabled. This is documented and tested for Azure Key Vault private endpoints. The same mechanism applies to any HTTP endpoint reachable from the delegated subnet. An ARM template for automated VNet configuration is available in the Copilot Studio samples repository on GitHub.

Gap: No specific documentation confirms this path for the Foundry Agent Service endpoint. The mechanism is generic HTTP over delegated subnet to private endpoint, so there is no technical reason it would not work, but it requires validation as Proof 3 in the validation proofs.

---

## Same-Tenant Constraint

Copilot Studio and Microsoft Foundry must be in the same Microsoft Entra ID tenant. This is a hard platform constraint documented by Microsoft and confirmed in the Azure labs repository. The constraint applies to both the native connector (Pattern 1a) and the REST API path (Pattern 1b, where the OBO token is tenant-scoped). For multi-tenant deployment scenarios, each tenant requires its own Copilot Studio environment and Foundry project. Cross-tenant agent invocation is not supported.

---

## Copilot Studio Governance Controls

Copilot Studio provides governance controls that apply regardless of which delivery pattern is selected. These are relevant to the regulated environment context and supplement the platform-level controls.

Data Loss Prevention policies allow admins to govern which connectors, knowledge sources, HTTP endpoints, and channels are permitted. Maker audit logs in Microsoft Purview provide full visibility into agent configuration changes. Agent activity logs in Microsoft Sentinel enable runtime monitoring and alerting on agent behaviour. Sensitivity label display shows SharePoint sensitivity labels in responses, with custom rendering needed for AI Search sources. Customer-managed encryption keys are available for Copilot Studio environments. Environment routing allows admins to direct makers to governed environments. Agent runtime protection status lets makers see security posture before publishing. Publication control allows admins to disable publishing of agents with generative AI features tenant-wide.

---

## Cost Considerations

Copilot Studio uses a message-based billing model with two licensing options: pay-as-you-go per-message charge, and a message pack subscription of 25,000 messages per month. M365 Copilot license holders get access to Copilot Studio with no per-message charge. Each user interaction turn that invokes generative AI consumes messages. The HTTP Request call to the Foundry Agent Service does not consume additional Copilot Studio messages beyond the turn that triggered it, but the Foundry API call incurs Azure consumption costs for model inference and AI Search queries.

If the integration uses Power Automate flows instead of HTTP Request nodes, Premium connector licensing applies. The Azure AI Foundry Agent Service connector is classified as Premium.

Azure consumption costs for model inference, AI Search queries, and storage are billed through Azure consumption and are independent of the delivery channel. These are governed by the platform cost model documented in the reference architecture.

---

## Validation Proofs Required Before Accepted Status

Four validation proofs must be completed before this ADR moves from Proposed to Accepted.

**Proof 1: Integration.** Build a Copilot Studio topic calling Foundry Agent Service REST API via HTTP Request. Pass user query. Receive response with annotations. Render result. Validate multi-turn via stored thread IDs. Acceptance criteria: Successful five-turn conversation with context maintained across turns. Owner: Architecture lead. Blocking: Yes.

**Proof 2: Identity (OBO).** Configure OBO auth on HTTP Request path. Pass user's Entra ID token to Foundry. Foundry agent uses it for AI Search security-trimmed queries. Acceptance criteria: Two users with different document ACLs receive different search results for the same query. Owner: Architecture lead and Security. Blocking: Yes.

**Proof 3: Private Link.** Enable Power Platform VNet support. Deploy Foundry Agent Service private endpoint. Confirm HTTP Request calls route through private network. Acceptance criteria: Network trace or NSG flow logs confirm no public internet egress for the Foundry API call. Owner: Architecture lead and Infrastructure. Blocking: Yes.

**Proof 4: Citations.** Validate Foundry API responses include file_citation annotations when agent uses AI Search knowledge. Parse annotations. Render in Copilot Studio as structured citation text or Adaptive Card. Test in both Teams and M365 Copilot channels to confirm citation text survives URL sanitisation. Acceptance criteria: Response contains at least document name and chunk reference for each grounded statement, and citation content is not removed or modified by either channel. Owner: Architecture lead. Blocking: No for Phase 1 (system-prompt citations validated separately during pilot). Blocking for Phase 2 promotion.

All blocking proofs must pass before this ADR moves to Accepted. If Proof 2 or Proof 3 fails, Pattern 1b is not viable for production and the architecture falls back to Pattern 2 or Pattern 3 with explicit documentation of the trade-offs.

---

## Monitoring the Native Connector (Pattern 1a)

The native Copilot Studio Foundry connector is the preferred long-term integration path because it eliminates the HTTP Request engineering overhead, may provide native citation pass-through, and aligns with Microsoft's stated platform convergence direction.

Tracking criteria for migration from Pattern 1b to Pattern 1a:

- Foundry agents with tools and knowledge sources must be supported and not failing.
- Citation pass-through must render Foundry annotations as Copilot Studio citations.
- OBO identity propagation must flow the user token through the connector to Foundry for downstream auth.
- VNet routing must route connector calls through the Power Platform VNet.
- The connector must reach GA status and exit preview.

When all criteria are met, evaluate migration from Pattern 1b to 1a. Document the evaluation in an ADR-003 amendment.

---

## Consequences

### Positive

- Preserves business value from familiar Microsoft channels without forcing unsafe equivalence between channels that have materially different maturity and governance characteristics.
- Keeps architecture and rollout aligned with actual product maturity rather than optimistic product-roadmap assumptions, reducing the risk of deploying into a channel that cannot meet evidence-grounding requirements.
- Makes channel approval a governance decision with explicit criteria instead of a marketing checkbox, ensuring that citation behaviour, identity flow, and network posture are verified before each channel goes live.
- HTTP Request integration gives full control over authentication headers, response parsing, and error handling while the native connector matures.
- The architecture is forward-compatible: when Pattern 1a reaches GA and meets all tracking criteria, migration from HTTP Request to native connector is a configuration change, not an architecture change.
- Copilot Studio provides a production-ready governed front-end with automatic Entra ID authentication, DLP policies, audit logging through Purview and Sentinel, and organisational distribution through the Teams app store and M365 Agent Store.

### Negative

- Requires stakeholders to accept phased rollout rather than immediate channel parity, which may create perception issues if other enterprise tools achieve M365 Copilot presence sooner.
- Creates additional release-management overhead because the channel matrix must be tracked explicitly and re-evaluated as Microsoft's channel capabilities evolve.
- The gate criteria require ongoing monitoring of Microsoft documentation and preview-to-GA transitions, adding an operational burden to the team.
- HTTP Request integration requires engineering: topic authoring, JSON schema definition, response parsing, multi-turn thread management, error handling. This is not zero-code.
- Two moving parts (Copilot Studio and Foundry) create a wider failure surface than a single-service approach.
- M365 Copilot channel limitations constrain response formatting and eliminate some UX patterns including greetings, media, and reactions.

### Constraints Introduced

- Channel go-live decisions must include review of citation behaviour, identity behaviour, and network posture for the specific backend pattern being published.
- Teams and Microsoft 365 Copilot are not automatically treated as equivalent production channels for UC1. Each must independently meet the readiness criteria.
- No channel publication may proceed if it would remove citation text that the agent explicitly includes in its response, because citation fidelity is a core UC1 requirement.
- Citation URLs must use Markdown link formatting or the citation section rather than bare inline URLs to survive M365 Copilot URL sanitisation.
- The Copilot Studio to Foundry integration must use OBO authentication to propagate the user's identity. The specific Entra ID app registration, scope, and Azure API Connections service principal authorisation are documented in the Identity Flow section and must be validated before production deployment.
- Copilot Studio and Foundry must reside in the same Entra ID tenant.

---

## Impact on Other ADRs

- **UC1-ADR-001**: Add note that Copilot Studio to Foundry integration uses HTTP Request (Pattern 1b) due to native connector preview limitations. No change to orchestration layer decision.
- **UC1-ADR-005**: Add constraint that citation metadata is delivered via system-prompt formatting in Phase 1 or parsed Foundry API annotations in Phase 2. Citation URLs must use Markdown link formatting to survive M365 Copilot URL sanitisation. Specify selected approach after Proof 4.
- **UC1-ADR-006**: Add constraint that OBO authentication is required on the HTTP Request path. Document Entra ID app registration, scope, and Azure API Connections service principal authorisation. Flag as blocking validation item.
- **UC1-ADR-007**: Add note that Copilot Studio maker audit logs in Purview and agent activity logs in Sentinel supplement the operational audit trail. Include in the observability design.
- **UC1-ADR-008**: No change. UC1 remains read-only consumer of UC2 outputs. Delivery channel does not affect this relationship.

---

## Channel Readiness Go/No-Go Criteria

The phased dual-channel strategy requires concrete, implementation-grade criteria for each channel to be approved for production use. The following table defines the go/no-go rules under Pattern 1b:

|Criterion|Teams|Microsoft 365 Copilot|Assessment Method|
|---|---|---|---|
|Citation rendering|System-prompt citations rendered as text in agent response. No known URL sanitisation applied in Teams channel. Go for pilot with Option A. Conditional go for production pending Option B validation (Proof 4).|System-prompt citation text (document names, chunk references) is not modified by the M365 Copilot orchestrator. Bare citation URLs may be removed or downgraded to plain text by the @mention pipeline. Go for pilot with Option A using Markdown link formatting for any URLs. Conditional go for production pending confirmation that Adaptive Card citations (Option B) survive sanitisation.|Test with published agent in both channels; compare Foundry response body with channel-rendered output; specifically test bare URLs versus Markdown-formatted URLs versus citation section URLs|
|Identity passthrough|Teams SSO passes user identity to Copilot Studio. OBO from Copilot Studio HTTP Request to Foundry must be validated. Conditional go pending Proof 2.|M365 Copilot SSO available. The M365 Copilot orchestrator does not modify identity context for Copilot Studio agents. Conditional go pending validation that user token propagates end-to-end through Copilot Studio HTTP Request to Foundry.|Deploy test agent; validate user token at each hop: Teams/M365 Copilot to Copilot Studio to Foundry to AI Search|
|Private Link / network isolation|HTTP Request from Copilot Studio in VNet-enabled Managed Environment to Foundry private endpoint. Conditional go pending Proof 3. Foundry Agent Service runtime itself is fully network-isolated.|Same Copilot Studio HTTP Request path applies. Conditional go pending Proof 3.|Network trace or NSG flow logs; Azure Network Watcher|
|Data residency (EU)|Copilot Studio environment in EU region. Foundry and all downstream services in EU region per platform ADR-003. HTTP Request stays within VNet. Requires validation that Copilot Studio message processing stays within EU data boundary.|M365 Copilot orchestrator processing location: requires validation against Microsoft's M365 data boundary documentation.|Review Copilot Studio geographic data residency documentation; review M365 data boundary configuration|
|Response fidelity|Teams renders text and Adaptive Cards; no known response reshaping for Copilot Studio agents. Go.|M365 Copilot does not rewrite or paraphrase agent response text. URL sanitisation is the only documented modification. Unsupported message types are rejected with ContentFiltered errors, not silently modified. Go for pilot with text-based responses. Conditional go for production pending validation that all UC1 response formats (text, Adaptive Cards) render correctly.|Compare Foundry agent output with Copilot Studio test pane output with channel-rendered output; verify no content loss; specifically test URL survival in different formats|

Fallback channel behaviour: If the primary pilot channel (Teams via Pattern 1b) fails to meet go criteria for a specific deployment context, the fallback is Pattern 3: direct Foundry API access through a minimal web application. This provides full citation rendering, full identity control, and full Private Link support but sacrifices the embedded-in-Teams discoverability.

Channel demotion rules: If a channel that was previously approved for pilot use regresses (e.g., Microsoft changes Copilot Studio channel behaviour, introduces new limitations, or the native connector GA breaks existing HTTP Request patterns), the channel must be demoted to unapproved status and users migrated to the fallback until the regression is resolved.

---

## Follow-ups

- Reassess Pattern 1a when the native Copilot Studio Foundry connector reaches GA and supports agents with tools and knowledge sources.
- Reassess M365 Copilot production readiness when URL sanitisation behaviour has been validated against UC1's citation formats in both @mention and direct-chat contexts.
- Reassess this ADR when Foundry direct publication supports citations and Private Link, which would change the Pattern 2 evaluation.
- Execute Proofs 1 through 4.
- Update this ADR to Accepted or record the fallback decision based on proof outcomes.