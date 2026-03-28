# ADR-007: Copilot Studio as Conversational Interface, Microsoft Foundry as Governed Backend

**Status:** Accepted  
**Date:** 27/03/2026  
**Decision Scope:** How the platform integrates Microsoft Copilot Studio, Power Platform Managed Environments, and Microsoft Foundry for private, governed, production-grade AI systems.  
**Depends on:** Platform ADR-002 (Runtime Platform and EU Region), Platform ADR-003 (Network Isolation and Data Residency), Platform ADR-004 (Identity and Authentication), Platform ADR-005 (Data Protection)  
**Depended on by:** None

---

## Context

ADR-002 selected the compute substrate and reference EU region. That platform-region decision remains unchanged; this ADR defines the integration boundary, not a new primary processing region. ADR-003 defined the network isolation model and private endpoint inventory for the processing layer. This ADR defines how user-facing conversational AI integrates with the governed backend, and where the architectural boundary sits between Copilot Studio and the platform's backend services.

Copilot Studio is the Microsoft-native conversational delivery layer, designed for user-facing agents, Microsoft 365 integration, and low-friction business adoption. Microsoft Foundry is the platform backend for governed AI execution where use cases require private networking, controlled orchestration, multi-agent coordination, enterprise security, tracing, evaluation, and production-grade operational control.

Many organisations already hold Microsoft 365 and Power Platform investments. Copilot Studio is frequently the default entry point for AI delivery because it offers a familiar conversational interface and rapid time to value.

As workflows grow more complex, constraints emerge. Conversational state becomes harder to manage cleanly. Orchestration across multiple specialist agents becomes difficult. Complex tool invocation and governed backend execution become fragmented across topics, actions, and flows. Compliance, auditability, and survivability requirements sit below the user interface layer rather than inside it.

This is consistent with Microsoft's own positioning. Microsoft documentation and technical guidance increasingly describe Copilot Studio and Microsoft Foundry as complementary platforms, with Copilot Studio as the conversational surface and Microsoft Foundry as the backend for deeper orchestration, tool integration, agent coordination, and enterprise control. It is also consistent with real-world implementation experience. RBA Consulting (February 2026) published a case study of a multi-agent system that began on Copilot Studio and migrated its backend to Microsoft Foundry after encountering limits in agent-to-agent coordination, conversation state control, and orchestration complexity.

The critical architectural question is not whether to use one platform or the other. It is where the boundary sits between the conversational delivery channel and the governed backend intelligence, and how the two communicate securely.

Copilot Studio provides two networking paths to Microsoft Foundry. The native connected agent integration uses the Foundry service's public endpoint, routed over the Microsoft backbone. Traffic stays within Microsoft's network infrastructure and does not traverse the open internet, but it addresses a publicly resolvable DNS name and requires the Foundry resource to have its public endpoint enabled. Power Platform Managed Environments with Azure Virtual Network support provide a second path: Azure subnet delegation routes outbound Power Platform traffic through delegated subnets in the customer's VNet, enabling private connectivity to backend services via private endpoints. This path does not require the Foundry resource to have its public endpoint enabled.

For production regulated environments where the Foundry resource has public network access disabled, the native connected agent path is unavailable. VNet delegation with private endpoints is the only supported connectivity model in that scenario.

Foundry Agent Service has its own private networking model using BYO VNet with subnet delegation. Both Copilot Studio (via VNet delegation) and Foundry Agent Service can participate in the same private Azure estate when their VNets are peered to the platform VNet or to a shared hub that provides access to the required private endpoints.

---

## Decision Drivers

- **Reuse Microsoft-native conversational delivery channels**: Reuse Microsoft-native conversational delivery channels where they are already adopted.
- **Keep interface and backend separate**: Keep the user-facing interface separate from governed backend intelligence.
- **Support advanced backend workflows**: Support complex, multi-agent, tool-rich, and stateful backend workflows without encoding them in Copilot Studio topic logic.
- **Maintain private connectivity**: Maintain private connectivity between Power Platform and Azure resources for production regulated environments.
- **Provide a defensible production control story**: Provide a defensible production control story: governance, auditability, and operational traceability at the backend layer.
- **Preserve architectural flexibility**: Preserve architectural flexibility so the conversational front end can evolve independently of backend intelligence services.
- **Avoid the re-architecture wall**: Avoid the re-architecture wall that organisations encounter when they stretch Copilot Studio into enterprise processing workflows.

---

## Considered Alternatives

### Option A: Copilot Studio as both conversational interface and backend orchestration layer

Build conversational UI, tool invocation, memory handling, and orchestration primarily inside Copilot Studio and Power Platform components.

**Why considered**: fastest path for simple conversational use cases, strong fit where workflows are shallow and mostly connector-based, lowest barrier to entry, single platform reduces operational complexity for small teams.

**Why not chosen**: as complexity grows, orchestration logic becomes fragmented across topics and flows, conversation-state handling becomes harder to reason about, and multi-agent patterns become difficult to manage cleanly. Backend governance, private AI service integration, and auditability are harder to express as a durable architecture. Copilot Studio is a poor fit once orchestration becomes complex and stateful. Organisations that adopt this pattern for enterprise workloads encounter a re-architecture wall that costs more than building correctly from the start.

### Option B: Microsoft Foundry only, with custom interface

Bypass Copilot Studio and build a custom application interface directly over Foundry services.

**Why considered**: maximum backend control and architectural purity, strong fit for fully bespoke applications where no existing M365 or Power Platform investment exists.

**Why not chosen**: rejects the practical value of Copilot Studio where organisations already have M365 and Power Platform adoption, removes a strong delivery channel for business-facing conversational scenarios, requires engineering effort for conversational interfaces that Copilot Studio provides out of the box, excludes citizen developers and business users from contributing.

### Option C: Copilot Studio as conversational interface, governed backend services for processing (SELECTED)

**Why chosen**: each layer is used for what it does best. Copilot Studio provides a rapid, accessible conversational surface for business users. Governed backend services, with Microsoft Foundry as the preferred backend where agentic orchestration is required, provide production-grade processing for engineering teams. The re-architecture wall is avoided because use cases that start simple in Copilot Studio and grow in complexity have a defined path to the backend without rebuilding the user-facing layer. Governance, observability, and security are applied consistently at the processing layer. The architecture composes across use cases: a backend agent or API built for one use case can be invoked by a Copilot Studio agent in another use case without duplicating the processing logic. Private networking is first-class on both sides. The conversational interface and backend services can evolve independently as both platforms mature.

### Option D: Independent selection per use case without a platform-level decision

Each use case team chooses its own platform based on local requirements.

**Why considered**: maximum flexibility per team.

**Why not chosen**: leads to inconsistent governance posture, duplicated infrastructure, incompatible integration patterns, and no shared security or observability model. The organisation loses the ability to compose capabilities across use cases or enforce consistent compliance controls.

---

## Decision

Adopt a conversational interface/governed backend split pattern. Copilot Studio is the preferred conversational interface where a Microsoft-native user interaction layer is desirable. In this platform, Copilot Studio is used only as the conversational surface. Microsoft Foundry is the preferred backend for advanced AI execution where agentic orchestration is required, including multi-agent coordination, governed tool use, complex workflow execution, and enterprise-grade backend control.

Copilot Studio agents are published to Microsoft 365 channels (primarily Teams and M365 Copilot) via the built-in Teams and Microsoft 365 Copilot channel in Copilot Studio. Deployment is governed by the client's Teams Admin Center and Power Platform admin policies.

Other governed backend services (such as Container Apps-hosted APIs per ADR-002) may sit behind the same conversational interface where the use case does not require Foundry-specific capabilities.

Power Platform Managed Environments with VNet delegation are the required connectivity mechanism for Copilot Studio to reach backend Azure services privately in production regulated environments where backend services have public network access disabled.

Private endpoints, private DNS, managed identity, and region-aligned network design are part of the production reference pattern, not optional enhancements. Copilot Studio is treated as a delivery channel, not as the authoritative location of business-critical backend intelligence.

---

## Decision Details

### 1. Conversational interface and backend responsibilities are separated

Copilot Studio owns the conversational entry point, user interaction, and controlled invocation of backend services. In this platform it is used only as the conversational surface. It does not own backend orchestration, workflow state, review handling, failure handling, or operational process management.

Microsoft Foundry and Azure backend services own advanced reasoning and orchestration, multi-agent coordination, stateful backend processing, tool integration beyond simple connector patterns, governed access to enterprise data and private services, and backend observability, security controls, and operational traceability.

The boundary sits at the point where a use case requires one or more of the following: governed multi-step processing with auditable state transitions, custom orchestration logic beyond Copilot Studio's topic/flow model, confidence-based routing with human-in-the-loop review, enterprise-grade failure handling with bounded retries and dead-letter semantics, multi-agent coordination with explicit handoff logic, or production evaluation with continuous monitoring. If a use case requires any of these, its processing layer belongs in the governed backend. Microsoft Foundry is the preferred backend where agentic orchestration is required. Container Apps-hosted APIs (per ADR-002) serve use cases where the backend is a governed API or pipeline rather than an agentic workflow.

### 2. Two networking paths exist between Copilot Studio and Microsoft Foundry

The native Copilot Studio connected agent integration calls the Foundry service's public endpoint over the Microsoft backbone. Traffic does not traverse the open internet, but it addresses a publicly resolvable endpoint and requires the Foundry resource to have public network access enabled. This path is acceptable for environments where backbone-routed traffic between Microsoft first-party services meets the organisation's security posture and the Foundry resource permits public endpoint access.

For production regulated environments where the Foundry resource has public network access disabled, this path is unavailable. In that scenario, Power Platform Managed Environments with VNet delegation are required. The Copilot Studio agent calls the backend endpoint via an HTTP Request node. The call exits through the delegated subnet, traverses VNet peering, and reaches the backend private endpoint using a private IP address resolved via private DNS. No public endpoint is involved.

The platform reference pattern assumes public network access is disabled on backend Azure resources per ADR-003. VNet delegation is therefore the default integration path for this platform.

### 3. Power Platform VNet delegation topology is explicit

For supported Power Platform regions, delegated subnets must exist in the paired Azure regions for the Power Platform region. For Europe, this means West Europe and North Europe. Each delegation VNet must be peered to the VNet or hub that provides access to the platform's private endpoints. The two delegation VNets do not require direct peering between themselves. Power Platform uses them independently for regional failover.

If Azure resources are hosted in other VNets or regions (such as the platform VNet in Sweden Central per ADR-002), connectivity is established through VNet peering to the platform VNet or to a shared hub.

Power Platform VNet delegation uses the Microsoft.PowerPlatform/enterprisePolicies resource provider. The delegated subnet cannot be shared with other services or other enterprise policies. Each Power Platform environment is linked to one enterprise policy.

### 4. Subnet planning follows explicit sizing guidance

For Power Platform delegated subnets, size according to the number of environments attached to an enterprise policy. Microsoft guidance recommends approximately 25 to 30 IPs per production environment and 6 to 10 per nonproduction environment. Each subnet reserves five IP addresses.

For Microsoft Foundry Agent Service, if adopted as a hosted agent runtime, a dedicated agent subnet delegated to Microsoft.App/environments is required alongside a separate private endpoint subnet. The detailed subnet specification is documented in the platform reference architecture, not in this ADR.

The platform processing layer already uses a compute subnet delegated to Microsoft.App/environments for Azure Container Apps. If the platform later adopts Foundry Agent Service, subnet compatibility and separation must be assessed explicitly rather than assumed.

### 5. Integration paths from Copilot Studio to backend services must remain controlled

**Platform default:**

- **HTTP Request node to private-endpoint-enabled backend services.** This is the primary verified private integration mechanism and the default path for this platform. It applies to Foundry Agent Service endpoints, Container Apps-hosted APIs, and any other backend service exposed via private endpoint.

**Allowed alternatives, subject to validation:**

- **Native Foundry Agent connector (connected agent integration).** This uses the Foundry public endpoint over the Microsoft backbone. It is acceptable where the organisation permits backbone-routed traffic and the Foundry resource has public access enabled. It is not available where public access is disabled.
- **MCP connector within Copilot Studio.** Private MCP is supported through the VNet subnet where the MCP server is privately hosted. Public MCP endpoints follow the same constraints as any public-endpoint integration.
- **Mediated API patterns** such as API Management where backend abstraction or protocol mediation is needed.

No design should assume unrestricted public internet connectivity from Power Platform to backend Azure services in production regulated environments.

### 6. Traffic control and logging require an explicit design choice

Where outbound internet access is required from delegated Power Platform subnets, NAT Gateway is the baseline control for managed outbound egress. Azure Firewall is preferred where central rule enforcement, inspection, and traffic logging are required. NSG flow logging has known limitations for delegated-subnet and private-endpoint-heavy architectures. Where auditable traffic logging is a regulatory requirement, Azure Firewall provides the necessary visibility.

---

## Consequences

### Positive

- Clear separation between conversational interface and governed backend execution.
- Stronger fit for complex, multi-agent, and stateful enterprise AI systems.
- Better control over private networking, compliance, and backend observability.
- Reuses Copilot Studio where it adds value without forcing backend architecture into it.
- Reduces the risk of encoding critical orchestration logic into low-code conversation structures.
- The architecture composes across use cases without duplicating processing logic.
- The two networking paths (backbone-routed native connector and VNet-delegated private endpoint) give organisations a choice aligned to their security posture rather than forcing a single model.
- The backend is not locked to Foundry Agent Service. Container Apps-hosted APIs and other governed services can serve use cases that do not require agentic orchestration.

### Negative

- More moving parts than a Copilot Studio-only design.
- Requires explicit network planning across Power Platform and Azure boundaries.
- Introduces additional DNS, private endpoint, VNet peering, and regional design overhead.
- Teams must understand where conversational interaction stops and backend processing begins.
- Integration contracts between Copilot Studio and backend services must be defined, versioned, and maintained.
- The native connected agent path and the private endpoint path have different capabilities, constraints, and failure modes. Teams must understand which path their deployment uses and why.

---

## Constraints

- Managed Environments are required for Power Platform VNet scenarios.
- Power Platform delegated subnets must be dedicated and region-paired according to the Power Platform region mapping.
- Foundry Agent Service private networking, if adopted, requires dedicated subnet design separate from the platform Container Apps compute subnet.
- Public-network-first shortcuts are not acceptable for production regulated deployments where backend resources have public access disabled.
- Backend governance, auditability, and operational ownership must not be delegated to Copilot Studio topics or flows alone.
- Use cases requiring governed multi-step processing, human-in-the-loop review, or auditable state management must place their processing layer in the governed backend.
- Copilot Studio agents that invoke backend services must use the defined integration surface.
- Direct coupling between Copilot Studio conversation state and backend processing state is not permitted.

---

## Risks and Mitigations

|Risk|Likelihood|Impact|Mitigation|
|---|---|---|---|
|The native Foundry Agent connector within Copilot Studio uses the Foundry public endpoint and is unavailable when the Foundry resource has public access disabled.|High (by design)|High|The HTTP Request node over VNet-supported private endpoints is the verified private integration path and the default for this platform. If Microsoft later enables the native connector to route through private endpoints, the integration path can be reassessed.|
|Copilot Studio multi-agent orchestration matures and blurs the conversational interface/backend boundary.|Medium|Medium|The boundary criteria are defined explicitly in Decision Detail 1. Re-evaluation follows the criteria in the follow-ups section.|
|Power Platform VNet delegation introduces complexity that small client IT teams may not be prepared for.|Medium|Medium|The platform provides IaC templates for enterprise policy creation, VNet delegation, and subnet configuration. Networking prerequisites are validated during pre-deployment assessment.|
|Entra Conditional Access policies block Power Platform service principals or Foundry workspace identities.|Medium|High|Validate identity prerequisites during pre-deployment assessment and document required exclusions in the client integration model.|

---

## Architecture Implications

The reference architecture diagrams should reflect the Power Platform side (Managed Environment, enterprise policy, delegated subnets in West Europe and North Europe, VNet peering to platform VNet or hub, NAT Gateway or Azure Firewall on delegated subnets), the Azure platform side per ADR-003, and the integration path from Copilot Studio through the delegated subnet, across VNet peering, to the platform's private endpoints. The conversational interface/backend boundary should be labelled explicitly on the diagram. Both networking paths (native connector via backbone and HTTP Request node via private endpoint) should be represented, with the private endpoint path marked as the default for this platform. Showing both paths prevents teams from unknowingly using the native connector path and bypassing the platform's security posture, since the native path is the one promoted in Microsoft's getting-started guides and tutorials.

---

## Notes

Copilot Studio remains a valid platform for straightforward conversational use cases. This ADR does not reject Copilot Studio. It rejects using Copilot Studio as the default place to solve backend orchestration, governed multi-agent execution, and private enterprise AI architecture.

The Power Platform VNet topology (West Europe and North Europe) is separate from the platform processing VNet (Sweden Central per ADR-003). These are different VNets in different regions serving different purposes. Connectivity between them is established via VNet peering.

The distinction between backbone-routed traffic and private-endpoint traffic is material. Backbone-routed traffic between Microsoft first-party services does not traverse the open internet, but it uses publicly resolvable endpoints and requires the target service to have public access enabled. Private-endpoint traffic uses private IP addresses resolved via private DNS and does not require public access on the target service. These are different security postures with different implications for regulated environments. The platform defaults to the private-endpoint model because ADR-003 disables public access on backend resources.

The backend is not exclusively Microsoft Foundry. Foundry is the preferred backend where agentic orchestration is required. Container Apps-hosted APIs (per ADR-002) serve use cases where the backend is a governed API or pipeline. The conversational interface/backend split applies regardless of which backend service is behind the boundary.

---

## Follow-ups

- Define a standard integration contract schema for Copilot Studio to backend service calls via the HTTP Request node, including request/response format, error handling, and timeout behaviour. Determine whether a single contract envelope serves all backend types (Foundry Agent Service, Container Apps API) or whether each backend type defines its own contract within a shared envelope structure.
- Validate the authentication model when a Copilot Studio agent calls a Foundry agent on behalf of an authenticated user over the private endpoint path. Determine whether Entra Agent Identity is sufficient for backend-to-backend calls or whether OAuth Identity Passthrough is required to preserve user-level identity through the integration path.
- Validate end-to-end private connectivity from a Copilot Studio agent in a Managed Environment with VNet delegation to a Foundry Agent Service endpoint or Container Apps-hosted API in the platform VNet. Confirm the call traverses the delegated subnet and VNet peering without public internet egress. Confirm that the native connected agent path fails when Foundry public access is disabled (negative test validates the private path is the effective path).
- Define explicit criteria for re-evaluating the conversational interface/backend boundary as Copilot Studio multi-agent orchestration matures. Criteria should include state management capability, audit trail fidelity, failure handling semantics, private networking coverage, and governance controls available natively within Copilot Studio. Additionally, reassess the networking path if Microsoft enables the native connected agent integration to route through private endpoints, which would simplify the integration model for regulated environments.