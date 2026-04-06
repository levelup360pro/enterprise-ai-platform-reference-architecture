# UC1-ADR-004: Knowledge Layer Strategy

**Status**: Accepted  
**Date**: 01/04/2026  
**Decision Scope**: Whether Azure AI Search, Fabric semantic assets, Fabric IQ, or Foundry IQ should anchor the UC1 knowledge layer.  
**Depends on**: UC1-ADR-001, UC1-ADR-002, UC2-ADR-006  
**Depended on by**: UC1-ADR-005, UC1-ADR-006, UC1-ADR-008

---

## Context

UC1 needs a reusable knowledge layer that can serve more than one source type and more than one use case. The knowledge layer is the substrate that the conversational agent queries when answering user questions, and the choice of substrate determines what kinds of knowledge can be surfaced, how governance is enforced, what cross-source retrieval capabilities are available, and how tightly the architecture is coupled to preview services.

Four candidate anchors exist: Fabric semantic assets only, Azure AI Search, Fabric IQ, and Foundry IQ. Each occupies a different position on the maturity and capability spectrum.

**Azure AI Search** is GA for core retrieval capabilities. The stable API (2025-09-01) provides general availability for the OneLake indexer, vector search, hybrid search (keyword plus vector with reciprocal rank fusion), semantic ranker, integrated vectorization, and the Document Layout skill. Security filters based on string comparison are GA and API-version-agnostic. Document-level access control via POSIX-like ACLs, RBAC scopes, SharePoint ACLs, and Microsoft Purview sensitivity labels remains preview (2025-11-01-preview API). Agentic retrieval, knowledge bases, and knowledge sources are preview (introduced May 2025, iterated through 2025-08-01-preview and 2025-11-01-preview).

**Foundry IQ** is the managed knowledge-retrieval abstraction layer for agents within Azure AI Foundry. It is built on Azure AI Search's agentic retrieval engine, which is itself preview. Foundry IQ is preview with no publicly announced GA date as of April 2026. It manages knowledge bases composed of multiple knowledge sources (Azure Blob, OneLake, SharePoint, existing search indexes, web via Bing). It supports agentic retrieval with LLM-driven query planning (supported models limited to gpt-4o, gpt-4.1, and gpt-5 series in Azure OpenAI), retrieval reasoning effort levels (Minimal, Low, Medium), permission-aware retrieval via Microsoft Entra identity and document-level ACLs, and citation-backed responses. It provides a higher-level agent-native API compared to direct Azure AI Search integration, but inherits the preview status and limitations of the underlying agentic retrieval engine.

**Fabric IQ** is a semantic intelligence workload within Microsoft Fabric. As of April 2026, all Fabric IQ components are preview: Ontology (preview), Graph (preview), Plan (preview), Data Agent (preview), and Operations Agent (preview). Fabric IQ provides a semantic foundation layer that unifies analytical and operational data across OneLake by defining enterprise vocabularies, entity types, relationships, rules, and constraints through an Ontology model. It is designed to ground Fabric-native AI agents (Data Agent, Operations Agent) and Power BI experiences in shared business semantics. New preview capabilities announced at FabCon Atlanta (March 2026) include rules and actions embedded in Ontology for automated business processes, enhanced enterprise security with granular permissions and Azure Private Link, and forthcoming public MCP (Model Context Protocol) endpoints for Ontology. Fabric IQ is not a cross-source indexed retrieval substrate. It does not index document-derived content, does not provide hybrid keyword-plus-vector search, and does not serve the same function as Azure AI Search. Its purpose is semantic intelligence and AI grounding within the Fabric analytical ecosystem, which is complementary to but architecturally distinct from the indexed document retrieval that UC1 requires. Fabric IQ complements Foundry IQ but cannot substitute for it; each IQ workload has its own data source support and cannot share sources across workloads.

**Fabric semantic assets** (Power BI semantic models, governed Delta tables, Lakehouse/Warehouse) are GA and authoritative for governed analytics. They serve analytical truth through SQL, DAX, and the Fabric Data Agent. They are not designed as a general-purpose enterprise retrieval substrate for indexed document-derived content. 
The architecture must avoid two traps. First, choosing a product vision that is not yet mature enough to anchor production architecture: adopting a preview service as the day-one knowledge layer creates a dependency on behaviour that may change, features that may not ship, and SLAs that do not yet exist. Second, flattening every kind of knowledge into one substrate even when the authoritative source types are different: UC2 already writes governed outputs into both Fabric (structured analytical artefacts) and Azure AI Search (indexed document-derived content), and the knowledge layer strategy must respect this dual-output reality rather than forcing one plane to absorb the other.

---

## Decision Drivers

- UC2 already writes governed outputs into both Fabric (structured fields to Gold-layer Delta tables) and Azure AI Search (chunked document content with vector embeddings for semantic retrieval and citation). These are complementary, not interchangeable.
- Azure AI Search is GA for the core retrieval capabilities UC1 requires: OneLake indexer, hybrid search (keyword plus vector), semantic ranker, integrated vectorization (stable API 2025-09-01). Security filters via string comparison are GA. Native document-level ACL/RBAC and Purview sensitivity label enforcement remain preview.
- Foundry IQ provides a managed agent-native knowledge abstraction but is preview, built on Azure AI Search's agentic retrieval engine (also preview), with no publicly announced GA date.
- Fabric IQ is entirely preview (Ontology, Graph, Plan, Data Agent, Operations Agent). It is a semantic intelligence layer for Fabric-native analytics and AI grounding, not an indexed document retrieval substrate. It does not provide hybrid search, vector search, or cross-source document indexing.
- Fabric semantic assets (Delta tables, semantic models) are GA and remain the authoritative plane for governed analytics, but they cannot serve clause-level evidence retrieval, semantic search with citations, or cross-source document queries.
- The governed hybrid retrieval pattern established in UC1-ADR-002 requires both an analytical plane (Fabric) and an evidence-retrieval plane (Azure AI Search). The knowledge layer must support this dual-plane architecture.

---

## Considered Alternatives

### Option A: Fabric-only knowledge layer

All UC1 knowledge retrieval would flow through Fabric semantic assets, governed tables, and Fabric-native data access patterns. The conversational agent would query Fabric directly for both analytical and evidence-based questions, relying on Fabric's semantic modelling and governance capabilities as the single knowledge substrate. This approach provides a strong fit for analytical truth and reuses existing Fabric governance, RLS/CLS, and semantic-model investments.

However, Fabric is not designed as a general-purpose enterprise retrieval substrate for indexed document-derived content. Azure AI Search, which UC2 already feeds as a governed downstream output, cannot be replaced by Fabric for cross-source retrieval over extracted contract clauses, embedded document chunks, or multi-source knowledge. Adopting Fabric as the sole knowledge layer would either require duplicating UC2 indexed outputs into Fabric (losing the retrieval and ranking capabilities of Azure AI Search) or abandoning the document-grounded evidence retrieval that UC1 requires for contract and audit questions.

Trade-off: Simplest governance model and strongest analytical fidelity, but unable to serve as a cross-source retrieval substrate for document-derived knowledge and unable to reuse UC2 indexed outputs naturally.

### Option B: Foundry IQ as the immediate primary knowledge layer

Foundry IQ would serve as the primary knowledge abstraction, providing an agent-native knowledge-retrieval layer that officially integrates with Foundry agents and is built on Azure AI Search underneath. This is architecturally attractive because it offers a unified knowledge API designed for agent consumption, aligning naturally with the Foundry Agent Service selected in UC1-ADR-001. The integration between the agent and its knowledge sources would be first-class rather than manually wired.

However, Foundry IQ is still preview. Adopting it as the day-one operational anchor for a regulated production design introduces dependency on preview behaviour, preview SLAs, and feature availability that has not yet been confirmed for production workloads. The governance and security characteristics of the preview may not match the requirements of a regulated enterprise deployment. While Foundry IQ is a strong candidate for future adoption once it reaches GA, it is not yet the safest foundation for the initial production architecture.

Trade-off: Most architecturally elegant agent-native knowledge layer, but preview maturity makes it unsuitable as the day-one operational anchor for a regulated production design.

### Option C: Fabric IQ as the knowledge layer

Fabric IQ would serve as the knowledge substrate, leveraging its Ontology, Graph, and Data Agent capabilities to ground the conversational agent in business semantics.

However, Fabric IQ is entirely preview with no GA components. More fundamentally, Fabric IQ is architecturally unsuited for the UC1 knowledge layer requirement. It is a semantic intelligence workload designed to unify analytical and operational data within Fabric through entity modelling, vocabulary standardisation, and business-rule encoding. It does not index document-derived content, does not provide hybrid keyword-plus-vector search, does not support semantic ranking over document chunks, and does not produce citation-grounded retrieval results. Fabric IQ and Foundry IQ are complementary but separate; each has its own data source support and they cannot share sources across workloads. Fabric IQ could augment the analytical plane in future (for example, enriching Fabric Data Agent queries with Ontology context), but it cannot replace Azure AI Search as the evidence-retrieval substrate.

Trade-off: Strategically interesting for future Fabric-native AI grounding, but architecturally misaligned with the UC1 requirement for cross-source indexed document retrieval, and entirely preview.

### Option D: Azure AI Search as the operational knowledge layer, with Fabric remaining authoritative for analytics and IQ capabilities treated as future augmentation (SELECTED)

Azure AI Search would serve as the operational knowledge layer for indexed retrieval, evidence grounding, and cross-source knowledge. Fabric would remain the authoritative plane for governed analytics and semantic-model-based truth, consistent with the hybrid retrieval pattern established in UC1-ADR-002. Foundry IQ and Fabric IQ would be treated as future augmentation paths that can be adopted when their maturity is acceptable for production workloads, rather than as day-one dependencies.

This approach provides the best reuse of UC2 indexed outputs, which already flow into Azure AI Search through governed pipelines. Azure AI Search provides strong cross-source capability, spanning OneLake, SharePoint, Blob, and custom-indexed content, and is mostly GA when using classic indexes, the OneLake indexer, and integrated vectorisation. The architecture leaves room for Foundry IQ or Fabric IQ augmentation later without requiring rearchitecting, because Azure AI Search is the underlying substrate for Foundry IQ and the indexed retrieval path is independent of the IQ abstraction layer.

The cost is that the architecture must carry explicit integration between the agent and Azure AI Search (rather than relying on an IQ-managed abstraction), and the Fabric analytical plane must be integrated separately as a governed adjunct. Search security behaviour varies by source type and still includes preview areas for some indexer configurations.

Trade-off: Strongest operational foundation and best UC2 output reuse, but requires explicit integration work and carries source-specific security nuance rather than a uniform managed-knowledge abstraction.

---

## Decision

UC1 selects Azure AI Search as the operational knowledge layer for the target-state architecture. This decision anchors the knowledge layer on a GA substrate with proven cross-source retrieval capabilities while keeping the door open for future augmentation through IQ products as they mature.

**Azure AI Search** becomes the default reusable indexed retrieval layer. It receives UC2 downstream outputs through governed pipelines, supports hybrid search (keyword plus vector) over document-derived content, and provides the cross-source retrieval capability that UC1 needs for contract clauses, extracted fields, and embedded document chunks. The stable API (2025-09-01) covers OneLake indexer, vector search, hybrid search, semantic ranker, and integrated vectorization. Document-level security is achievable at GA via security filters (string comparison); native ACL/RBAC enforcement (preview) should be evaluated for adoption when it reaches GA.

**Fabric** remains the source of truth for governed analytics and semantic-model behaviour. It serves analytical queries (counts, sums, joins, trend analysis) through the Fabric Data Agent operating on Gold-layer Delta tables. It is not replaced or subordinated; it continues to serve its authoritative role for analytical truth, consistent with the governed hybrid retrieval pattern in UC1-ADR-002.

**Foundry IQ** is treated as an optional future wrapper over Azure AI Search. When both Foundry IQ and its underlying agentic retrieval engine (currently 2025-11-01-preview) reach GA with acceptable SLAs and security posture for regulated production workloads, it can be adopted as the agent-native knowledge abstraction. This adoption would simplify agent-to-Search integration without requiring changes to the underlying indexed content, embedding pipelines, or Azure AI Search infrastructure. The migration path is architecturally clean because Foundry IQ is built on the same Azure AI Search substrate.

**Fabric IQ** is treated as a strategic semantic enhancement path for the Fabric analytical plane. It is not a candidate for the UC1 knowledge layer because it does not provide indexed document retrieval, hybrid search, or citation-grounded responses. Its future value lies in enriching the Fabric Data Agent with Ontology-grounded business semantics, which could improve the quality of analytical queries in the adjunct retrieval plane. Adoption depends on Fabric IQ components reaching GA and demonstrating production readiness.

---

## Consequences

### Positive

- Strongest alignment with the downstream assets UC2 already produces, avoiding duplication or re-indexing of content that is already governed and available in Azure AI Search.
- Best cross-source retrieval posture among the evaluated options, spanning OneLake (GA indexer), SharePoint, Blob, and custom-indexed knowledge in a single retrieval substrate with GA hybrid search and semantic ranking.
- Avoids making any preview service mandatory for the first production-scale architecture, reducing the risk of depending on behaviour or SLAs that may change before GA. This applies to Foundry IQ, Fabric IQ, and Azure AI Search's agentic retrieval features equally.
- Keeps the analytical truth and retrieval truth distinction visible in the architecture, consistent with UC1-ADR-002, rather than collapsing both into a single abstraction.
- Clean future migration path to Foundry IQ when it reaches GA, because Azure AI Search is the underlying substrate and no re-indexing or content restructuring would be required.

### Negative

- Requires explicit design for how Fabric analytical truth is represented or referenced when search is the primary knowledge substrate, particularly for questions that span both information planes (cross-plane queries as defined in UC1-ADR-002).
- Means the architecture must carry source-specific security nuance for document-level access control. Security filters (string comparison) are GA but require application-side identity resolution. Native ACL/RBAC enforcement and Purview sensitivity label integration are preview. The security model varies across OneLake, Blob, and SharePoint indexers.
- Defers the architectural simplification that Foundry IQ could provide, requiring the team to maintain explicit Azure AI Search integration code (query construction, security filter population, hybrid/vector search execution, evidence-package formatting) until Foundry IQ reaches GA.
- Does not benefit from Fabric IQ's Ontology-grounded semantic enrichment for analytical queries until those capabilities reach GA; the Fabric Data Agent operates without Ontology context in the initial architecture.

### Constraints introduced

- Azure AI Search indexes become governed enterprise assets rather than disposable application caches. They must be managed with the same lifecycle discipline as Fabric governed tables: versioning, access control, monitoring, and change management.
- Foundry IQ, Fabric IQ, and Azure AI Search's agentic retrieval features should not be assumed as required first-production dependencies until they reach GA with acceptable SLAs for regulated workloads.
- All indexed content consumed by UC1 must flow through UC2 governed downstream boundaries (see UC2-ADR-006); direct indexing of raw source documents by UC1 is not permitted.
- Document-level security configuration must be validated per source type. For initial production, security filters (GA) are the baseline; native ACL/RBAC enforcement (preview) is an enhancement path to be evaluated at GA.
- The retrieval-mode selection rules defined in UC1-ADR-002 (analytical queries routed to Fabric, evidence queries routed to Azure AI Search, cross-plane queries orchestrated by the Foundry agent across both) must be enforced at the orchestration layer (UC1-ADR-001) and logged for audit.

---

## Follow-ups

- Monitor Foundry IQ and Azure AI Search agentic retrieval for GA announcements. When both reach GA, evaluate adoption as the managed knowledge abstraction. Migration from direct Azure AI Search integration to Foundry IQ should not require re-indexing or content restructuring.
- Monitor Fabric IQ Ontology and Data Agent for GA status. When GA, evaluate whether Ontology-grounded business semantics improve analytical query quality in the Fabric adjunct plane.
- Evaluate native ACL/RBAC enforcement and Purview sensitivity label integration (both preview) for production readiness when they reach GA. Until then, implement security filters (GA) as the baseline for document-level access control.
- Define the integration pattern between the Foundry Prompt Agent and Azure AI Search.
