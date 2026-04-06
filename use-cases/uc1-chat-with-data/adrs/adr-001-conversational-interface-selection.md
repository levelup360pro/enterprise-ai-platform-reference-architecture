# UC1-ADR-001: Conversational Orchestration Layer Selection

**Status**: Accepted  
**Date**: 30/03/2026  
**Decision Scope**: How UC1 selects the primary orchestration layer and its relationship to delivery channels across Fabric Data Agent, Copilot Studio, and Foundry Agent Service.  
**Depends on**: Enterprise AI Platform reference architecture, ADR-003 (Network Isolation and Data Residency Enforcement), ADR-004 (Identity, Authentication, and Authorisation), UC2 reference architecture  
**Depended on by**: UC1-ADR-002, UC1-ADR-003, UC1-ADR-005, UC1-ADR-006, UC1-ADR-007, UC1-ADR-008

---

## Context

UC1 must provide a conversational entry point that safely exposes governed enterprise information to business users. The assistant needs to answer questions that span two fundamentally different information planes: structured analytical truth held in Fabric semantic models and curated tables, and document-derived knowledge indexed in Azure AI Search as a downstream output of UC2. The layer chosen for UC1 is therefore not merely a user-experience decision. It determines how retrieval modes are orchestrated, how identity flows end-to-end from the user through the agent to the underlying data sources, how reusable the architecture is across future use cases, and how much the solution depends on services that have not yet reached general availability.

This decision separates two concerns that are often conflated. The orchestration layer is the component that receives a user query, determines which retrieval tools to invoke, coordinates their execution, synthesises the results, and returns a grounded response. The delivery channel is the surface through which the user interacts with the system: Microsoft Teams, Microsoft 365 Copilot, a web application, or another interface. These are independent architectural choices. The orchestration layer and the delivery channel do not have to be the same product.

Three candidate products exist, and they are materially different in capability, maturity, and governance posture. Fabric Data Agent is a Fabric-native conversational analytics artifact strongest for live questions over semantic models and curated Fabric data; it respects Fabric RLS/CLS natively but has no direct path for unstructured retrieval and imposes a 25x25 response cap. Copilot Studio is a low-code agent shell within the Power Platform ecosystem that provides a business-friendly conversation design workflow, native Teams and M365 Copilot deployment, and over 1,300 connectors; however, it has documented limitations in complex multi-agent orchestration including sequential-only execution, context and state loss between agents, and session/token limits under complex scenarios. Microsoft Foundry Agent Service is a code-first, managed orchestration runtime that is GA at the service level and supports parallel multi-tool execution, custom model selection, and explicit governance over tool invocation, but requires more engineering effort and carries current channel-publication limitations.

Microsoft's own published guidance (Tech Community, January 2026) and partner architecture documentation consistently position Copilot Studio as the conversational front-end for end users and Foundry Agent Service as the backend orchestration engine for complex multi-tool scenarios. The recommended enterprise pattern across multiple Microsoft-endorsed sources is Copilot Studio as the delivery channel with Foundry Agent Service handling backend orchestration. This is not a niche recommendation. It reflects how Microsoft designed these products to work together.

The shared platform reference architecture favours explicit orchestration with full control over tool invocation, governance, and audit state. However, this is not a blanket mandate that everything must be code-first. UC2 chose code-first because it is a backend document processing pipeline with state management, confidence-based routing, agentic validation, and human review workflows. UC1 is a conversational interface. If a low-code interface met all the requirements for orchestration, it would be a valid choice. The decision must be based on capability fit, not on alignment with a platform implementation pattern that was chosen for different reasons.

---

## Decision Drivers

- UC1 must query both structured analytical truth in Fabric and indexed knowledge in Azure AI Search, requiring hybrid multi-tool orchestration across both retrieval planes.
- UC1 must reuse UC2 outputs rather than build a separate knowledge estate.
- The conversational interface should be selected based on its ability to support the required capabilities, not on whether it is low-code or code-first. If a low-code interface met all requirements, it would be a valid choice.
- The orchestration layer must support parallel retrieval across multiple tools (Fabric Data Agent for structured queries, AI Search for document knowledge) and combine results coherently. Sequential-only execution is insufficient for the cross-source query patterns UC1 requires.
- The architecture should minimise avoidable dependence on preview integrations for core orchestration paths.
- The delivery channel and the orchestration layer are separable concerns. The architecture should support Copilot Studio as a front-end delivery channel even if Foundry handles backend orchestration.
- Teams and Microsoft 365 Copilot are desirable delivery channels, but the architecture should not be forced into a weak orchestration path just to achieve immediate channel ubiquity.
- The selected orchestration layer must support future extension to additional knowledge sources, retrieval tools and future use cases.

---

## Considered Alternatives

### Option A: Fabric Data Agent as the primary orchestration layer

Fabric Data Agent would serve as the end-user conversational surface and orchestration layer, optionally published into Microsoft 365 Copilot. This approach offers the fastest path to value because Fabric Data Agent is purpose-built for live analytical questioning over curated semantic models and governed Fabric data. It respects Fabric RLS/CLS and source permissions natively, requires minimal custom orchestration, and provides the strongest fit for Fabric-native structured analytics scenarios.

However, Fabric Data Agent is not a full enterprise knowledge assistant. It has no direct path for unstructured retrieval over document-derived knowledge such as UC2 indexed outputs, SharePoint content, or Blob-stored artefacts. Its 25x25 response cap limits the depth of answers it can provide, and its core and integration maturity are not uniformly documented across all the scenarios UC1 requires. Cross-source retrieval spanning both Fabric analytical assets and Azure AI Search indexed knowledge is outside its design centre.

Trade-off: Fastest time-to-value for structured analytics, but fundamentally unable to serve as a cross-source orchestration layer spanning both Fabric and Azure AI Search retrieval planes.

### Option B: Copilot Studio as the primary orchestration layer

Copilot Studio would act as both the user-facing shell and the orchestration engine, delegating structured questions to a connected Fabric Data Agent while using its own topic and knowledge-source mechanisms for other question types. This is a natural fit for teams aligned to the Power Platform ecosystem and provides a business-friendly conversation design workflow with low-code tooling. It is deployable to Teams immediately and reduces the engineering effort required for basic conversational flows.

However, Copilot Studio has documented limitations that prevent it from serving as the orchestration engine for UC1's requirements.

Complex multi-agent orchestration in Copilot Studio hits session and token limits, causing agents to fail with default error messages. This is confirmed in both community reports and Microsoft employee responses (Reddit, March 2026). Context and state are not reliably passed between orchestrated agents. Microsoft explicitly confirmed that when orchestrating multiple agents, Copilot Studio may not pass context or variables correctly between them, causing sub-agents to lose their purpose or input. Capability mapping issues mean the orchestration logic may not correctly route user queries to sub-agent capabilities, leading to generic fallback responses.

Copilot Studio executes agents sequentially and cannot coordinate parallel retrieval across multiple tools. UC1 requires the ability to query Fabric structured data and AI Search document knowledge simultaneously and combine the results. This type of concurrent multi-agent execution is outside Copilot Studio's current design. Third-party architecture analysis (Bravent, January 2026) states directly: "This type of concurrent multi-agent execution is difficult to achieve in Copilot Studio (which executes agents sequentially), but Foundry is designed for performance and scalability in production."

The connected Fabric Data Agent integration remains preview and is not currently supported in Microsoft 365 Copilot for the connected-agent scenario. No confirmed first-class Azure AI Search grounding path was identified in official first-party documentation.

Microsoft's own published guidance (Tech Community, January 2026) and partner architecture documentation consistently position Copilot Studio as the conversational front-end for end users, not as the orchestration engine for complex multi-tool scenarios. The recommended enterprise pattern is Copilot Studio as the delivery channel with Foundry Agent Service handling backend orchestration. This is a material distinction: Copilot Studio is not rejected as a component of the UC1 architecture, but it is not suitable as the orchestration layer.

Trade-off: Fastest low-code path for simple conversational scenarios, but documented orchestration limitations (sequential execution, context loss, session limits) and preview-only Fabric integration prevent it from serving as the multi-tool orchestration engine UC1 requires. Remains a strong candidate as the user-facing delivery channel with Foundry orchestration behind it.

### Option C: Foundry Agent Service as the primary orchestration layer (SELECTED)

Foundry Agent Service would serve as the primary orchestration layer via a Foundry prompt agent, with Fabric Data Agent and Azure AI Search retrieval treated as subordinate tools or retrieval planes rather than as the primary UI shell. This approach provides the strongest fit for multi-tool orchestration where the agent coordinates across indexed retrieval, structured querying, and future capabilities via MCP or custom tools. The service is GA at the base orchestration level, providing a stable foundation for production deployment. Foundry supports parallel execution of multiple tool invocations, enabling the agent to query Fabric Data Agent for structured data and AI Search for document knowledge simultaneously and combine the results. The long-term extensibility story is strongest here because adding new retrieval sources or tool integrations is a natural operation rather than a retrofit.

Foundry provides Azure-native governance controls (RBAC, Key Vault, private endpoints, VNets, Application Insights, custom telemetry) that are more configurable than the inherited M365 governance model in Copilot Studio. For a regulated environment requiring explicit audit trails of tool invocations, custom content filtering, and fine-grained access control over which tools an agent can invoke, Foundry provides the necessary control surface.

The delivery channel is a separate concern from the orchestration layer. A Foundry-orchestrated agent can be surfaced through multiple channels. Copilot Studio can act as the conversational front-end, receiving user messages and forwarding them to the Foundry agent for orchestration, then presenting the response to the user. This is the pattern Microsoft endorses for enterprise scenarios. Alternatively, the Foundry agent can be published directly to Teams or M365 Copilot, or surfaced through a custom web application. Channel publication to Teams and Microsoft 365 Copilot carries current limitations that must be explicitly gated, including the absence of citation support and Private Link in the published-agent channel path. Fabric Data Agent integration inside Foundry remains preview if adopted as a subordinate tool.

Trade-off: Highest engineering effort and current channel-publication limitations, but the only option that supports hybrid multi-tool orchestration with parallel execution across both retrieval planes, and aligns with Microsoft's own recommended enterprise pattern of Foundry for orchestration with Copilot Studio as the delivery channel.

---

## Decision

UC1 selects Microsoft Foundry Agent Service as the primary orchestration layer.

This decision is based on capability requirements, not on a blanket preference for code-first implementation. Copilot Studio was evaluated as a potential orchestration layer and found to have documented limitations that prevent it from reliably orchestrating the hybrid multi-tool retrieval UC1 requires. Those limitations are sequential-only execution, context and state loss in multi-agent orchestration, session and token limits under complex scenarios, and preview-only integration with Fabric Data Agent. These are not theoretical concerns. They are confirmed in official Microsoft documentation, Microsoft employee responses, and published architecture guidance.

The architecture recognises that Copilot Studio and Foundry Agent Service serve different roles in the Microsoft agent ecosystem. Microsoft's own guidance positions Copilot Studio as the conversational front-end and Foundry as the backend orchestration engine for complex scenarios. UC1 follows this pattern. Foundry Agent Service is selected as the orchestration layer because it supports hybrid multi-tool retrieval, parallel execution, and explicit governance over tool invocation. Copilot Studio remains a candidate as the user-facing delivery channel, surfacing the Foundry-orchestrated agent to Teams and Microsoft 365 Copilot users. This separation of delivery channel from orchestration engine is not a limitation but a deliberate architectural choice aligned with how Microsoft designed these products to work together.

**Fabric Data Agent** remains the preferred initial quick-win path for structured conversational analytics over curated Fabric assets. It provides immediate value for questions that are naturally answered by semantic models and governed tables, and it can serve as an early demonstration of conversational capability while the Foundry-based target-state architecture is built out. In the target-state architecture, Fabric Data Agent becomes a subordinate tool that the Foundry agent invokes for structured Fabric queries.

**Copilot Studio** remains a candidate as the user-facing delivery channel for Teams and M365 Copilot. In this role, it handles the conversational surface (receiving user messages, presenting responses) while Foundry handles the orchestration (determining which tools to invoke, coordinating retrieval, synthesising results). This is the pattern Microsoft endorses for enterprise scenarios where orchestration complexity exceeds what Copilot Studio can handle natively.

**Foundry Agent Service** is the orchestration layer that coordinates retrieval across both information planes, manages tool invocation governance, and provides the extensibility point for future use cases and additional retrieval tools.

---

## Consequences

### Positive

- Aligns UC1 with Microsoft's own recommended enterprise pattern: Copilot Studio as the conversational front-end, Foundry Agent Service as the backend orchestration engine. This is supported by published Microsoft guidance and partner case studies.
- Supports parallel multi-tool retrieval across Fabric structured data and AI Search document knowledge, which Copilot Studio's sequential execution model cannot provide.
- Preserves the ability to combine Azure AI Search retrieval with Fabric analytical queries through explicit tool-based orchestration rather than relying on a single product to bridge both planes.
- Avoids coupling the orchestration layer to a product with documented limitations in complex multi-agent scenarios (context loss, session limits, sequential execution).
- Provides Azure-native governance controls (RBAC, Key Vault, private endpoints, VNets, Application Insights) that are more configurable than the inherited M365 governance model, meeting regulated environment requirements for explicit audit trails and fine-grained access control.
- Makes future MCP- or tool-based integrations a natural fit instead of a retrofit, because the Foundry agent model is designed around tool invocation.
- Separates the delivery channel decision from the orchestration decision, allowing the architecture to adopt Copilot Studio as the user-facing surface without constraining the orchestration capabilities.

### Negative

- Requires more engineering effort for orchestration, retrieval coordination, grounding, and audit state than either of the simpler single-product alternatives.
- Does not remove the need to manage current channel limitations for Teams and Microsoft 365 Copilot publication, including the absence of citation support and Private Link in the published-agent channel path.
- Introduces a phased story rather than a single product choice that solves both immediate and long-term needs, requiring stakeholders to understand the distinction between the quick-win Fabric Data Agent path, the Copilot Studio delivery channel, and the target-state Foundry orchestration layer.
- If Copilot Studio is used as the delivery channel, the integration between Copilot Studio and the Foundry agent introduces an additional component that must be built, tested, and maintained.

### Constraints introduced

- The Foundry agent is the orchestration boundary, not the system of record for enterprise content. It coordinates retrieval but does not own the data.
- The delivery channel (Copilot Studio, Teams direct publication, web app, or other) is a separate architectural decision that must be made in conjunction with this ADR but is not determined by it.
- Channel publication must be treated as a governed deployment decision, not an automatic consequence of having an agent. Each channel must meet citation, identity, and network-posture requirements before go-live.
- Fabric Data Agent may be used as a subordinate specialist capability invoked by the Foundry agent but is not the sole long-term orchestration layer for UC1.
- No conversational interface may bypass UC2 governance boundaries to access raw extraction state or intermediate pipeline artefacts directly.

---

## Follow-ups

- Re-evaluate this ADR if Copilot Studio's orchestration capabilities materially improve, specifically around parallel execution, context preservation in multi-agent workflows, and session/token limits for complex scenarios.
- Re-evaluate this ADR if Microsoft's channel limitations for Foundry publication materially improve, especially around citations and Private Link.
- Re-evaluate Copilot Studio's role as delivery channel once the connected Fabric Data Agent integration and M365 Copilot support reach GA.
- Define the delivery channel selection as a separate ADR that references this orchestration decision and evaluates the Copilot Studio front-end / Foundry back-end pattern against direct Foundry publication to Teams and other channel options.