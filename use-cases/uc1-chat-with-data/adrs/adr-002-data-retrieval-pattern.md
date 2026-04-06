# UC1-ADR-002: Data Retrieval Pattern

**Status**: Accepted  
**Date**: 30/03/2026  
**Decision Scope**: Whether UC1 should retrieve data through direct Fabric querying, Azure AI Search grounded retrieval, or a governed hybrid that treats both as architecturally required retrieval planes serving distinct query patterns.  
**Depends on**: UC1-ADR-001, UC2-ADR-006 (Staging as the Governed Downstream Boundary), UC2 reference architecture  
**Depended on by**: UC1-ADR-004, UC1-ADR-005, UC1-ADR-006, UC1-ADR-008

---

## Context

UC1 must answer at least three distinct classes of question, each of which requires a different retrieval mechanism because the underlying information exists in fundamentally different forms. Understanding why requires examining what UC2 actually produces.

The UC2 pipeline processes source documents and produces two distinct categories of downstream output from the same source material. The document types vary by industry and deployment context. In aircraft leasing the corpus includes lease agreements, amendments, side letters, and legal correspondence. In financial services it may include loan facility agreements, credit approvals, and regulatory filings. In insurance it may include policy wordings, endorsements, claims correspondence, and reinsurance treaties. In healthcare or pharmaceutical contexts it may include clinical trial agreements, regulatory submissions, and supply contracts. In professional services it may include engagement letters, service-level agreements, and consultancy frameworks. The architecture is designed to handle any document-intensive domain where the same dual-output pattern applies: structured fields extracted from documents and the full document content preserved for evidence-based retrieval.

The first category is structured extraction. UC2 pulls discrete fields from documents and writes them to Delta tables in the Microsoft Fabric Lakehouse. The specific fields depend on the document type and industry. Examples include: counterparty name, effective dates, parties, jurisdiction, total obligation or contract value, document type, and cross-references to related documents (such as amendment references in leasing, endorsement references in insurance, or addendum references in professional services). These are tabular, queryable, joinable with other Fabric data. They belong in the Gold layer of the Medallion architecture because they are business-ready structured data that feeds a 360 view, Power BI reporting, and downstream system integrations. This structured extraction has passed through UC2's full governance pipeline: confidence scoring, routing evaluation, optional agentic validation, human review where required, and PII classification. It is governed, validated, derived data. A Fabric Data Agent can query these tables natively because they are structured analytical truth. Questions such as "How many active agreements does Counterparty X have?" or "What is the total obligation value across all agreements expiring in 2027?" or "Which policies have endorsements pending renewal this quarter?" are aggregation, filtering, and joining operations over structured data. SQL or DAX over Gold tables answers them correctly.

The second category is the full document content: the actual text of the agreements, the clause language, the negotiated terms, the specific wording of termination or cancellation rights, the exact phrasing of liability caps or coverage exclusions, the amendments or endorsements that modified a specific section of the original document. This content does not reduce to structured fields. It is unstructured by nature. UC2 extracts key fields from it, but those fields are a summary of the document, not a replacement for it. When a user asks "What are the termination rights in Counterparty X's 2022 master agreement?" or "What exclusions apply under Policy Y's third endorsement?" or "What are the data-processing obligations in the 2023 DPA with Vendor Z?", the answer is not in a structured field. The answer is in a specific clause, in a specific section, of a specific document. To answer that question, the system needs to find the right document or the right chunk of the right document, retrieve the relevant passage, and present it with a citation to the source. That is a retrieval problem, not a query problem. It requires semantic search over document content, not SQL over structured tables. This document content is chunked, embedded, and indexed in Azure AI Search as part of UC2's staging-to-indexer path.

These two output categories exist because they serve fundamentally different purposes. The structured extraction summarises documents into queryable business data. The indexed document content preserves the source material for evidence-based retrieval. One is derived analytical truth. The other is the original evidentiary record. They are complementary, not redundant, and they cannot replace each other. This duality holds regardless of the industry or document type: a lease agreement in aircraft leasing, a loan facility in banking, a policy wording in insurance, and a clinical trial agreement in pharma all produce the same pattern of structured fields plus full document content.

Could the full document text be stored in Fabric? Technically yes, as raw text in a Lakehouse table. But Fabric is not a search engine. It does not do semantic similarity matching. It does not do vector search. It does not do hybrid keyword-plus-vector retrieval with relevance ranking. It does not return passages with positional citations back to the source document. Using a data warehouse as a search index is the wrong tool for the job. The query patterns are fundamentally different.

There is also a governance reason for maintaining both planes separately. The structured extraction in the Gold layer has passed through UC2's full governance pipeline and is governed as validated summary data. The document content indexed in AI Search retains original text, governed by a retrieval-focused model with citation support. When a user retrieves a clause from AI Search, they are reading the original document content. When they query a Gold table, they are reading a derived, validated summary. Both are valuable. Both need governance. But the governance mechanisms are different, and keeping them in their appropriate platforms makes the control story cleaner.

This dual-output reality means UC1 must serve three query pattern categories, not two.

**Analytical queries** ask questions that are answered by aggregation, filtering, trend analysis, or metric computation over structured Gold-layer data. These are routed to the Fabric structured-query adjunct because Fabric semantic models and governed tables are the authoritative plane for analytical truth. Forcing these through a search index would either degrade answer quality or require duplicating analytical logic into the search layer.

Examples across industries:

- Aircraft leasing: "How many active leases does Counterparty X have?" "What is the total obligation value across all leases expiring in 2027?"
- Financial services: "What is our total credit exposure to Borrower Y across all facility agreements?" "How many loan facilities are due for renewal in Q2?"
- Insurance: "Which policies have endorsements pending renewal this quarter?" "What is the aggregate insured value across all active property policies?"
- Professional services: "How many active engagements does Client Z have?" "What is the total contracted value across all SOWs expiring this year?"

**Knowledge retrieval queries** ask questions that require source-grounded evidence from the actual document content. These are routed to Azure AI Search because they require hybrid search (combining keyword matching with semantic vector similarity), relevance ranking, and the ability to return specific passages with source citations. AI Search with integrated vectorisation handles this.

Examples across industries:

- Aircraft leasing: "What are the termination rights in Counterparty X's 2022 master lease?" "Show me all references to force majeure across Counterparty X's documents."
- Financial services: "What covenants apply to Borrower Y's revolving credit facility?" "Show the exact wording of the cross-default clause in the 2023 facility agreement."
- Insurance: "What exclusions apply under Policy Y's third endorsement?" "Show the subrogation clause in the reinsurance treaty with Reinsurer Z."
- Healthcare/pharma: "What are the data-sharing obligations in the clinical trial agreement with Site A?" "Show the indemnification provisions in the 2024 supply contract."
- Professional services: "What limitation of liability applies under the engagement letter with Client Z?" "Show the IP ownership clause in the consultancy framework agreement."

**Cross-plane queries** span both structured data and document content. The first part is answered by Fabric; the second part requires retrieval from the indexed document content in AI Search. This is exactly the hybrid multi-tool orchestration pattern that drove the Foundry Agent Service selection in UC1-ADR-001. The Foundry agent queries both planes and synthesises the answer.

Examples across industries:

- Aircraft leasing: "Counterparty X has three leases expiring in Q1 2027. What are the renewal terms in each?"
- Financial services: "Borrower Y has two facilities above 10M. What are the financial covenants in each?"
- Insurance: "We have five property policies renewing next month. What are the coverage exclusions in each current wording?"
- Professional services: "Client Z has three active SOWs. What are the termination-for-convenience provisions in each?"

Forcing all questions through one retrieval pattern would either degrade answer quality or cause the architecture to bypass the platform's established downstream boundaries. Collapsing both information planes into a single mechanism obscures the answer quality and governance characteristics of each.

---

## Decision Drivers

- UC2 produces two distinct downstream artefact categories from the same source documents: structured extraction to Fabric Delta tables and full document content indexed in Azure AI Search. These are not redundant outputs; they serve fundamentally different query patterns. This holds regardless of the industry or document type being processed.
- Analytical questions (aggregation, filtering, metrics) are best answered by Fabric semantic models and governed tables, not by retrieved passages.
- Knowledge retrieval questions (clause-level evidence, specific document wording, cross-document search) demand document-grounded evidence, citations, and source-level traceability that only Azure AI Search provides.
- Cross-plane questions that require both structured context and document evidence exist as a real and important query class and require the Foundry agent to orchestrate retrieval from both planes.
- The retrieval pattern must not bypass UC2 staging and governance boundaries. Structured extraction governance (confidence scoring, human review, PII classification) and retrieval governance (document-level security, citation integrity) are different control models that should not be conflated.
- The architecture must remain understandable, testable, and auditable when the wrong retrieval mode is chosen or when a mixed-mode question requires both planes.

---

## Considered Alternatives

### Option A: Direct Fabric query only

All user questions would be answered through Fabric semantic assets or Data Agent-style structured querying. The conversational agent would translate natural language into queries against curated semantic models, governed tables, or Fabric-native analytical endpoints. This approach provides a strong fit for structured analytical questions because it reuses existing Fabric governance, semantic modelling, and RLS/CLS enforcement. The conceptual complexity is lower than a multi-plane design because there is only one retrieval substrate to reason about.

However, this approach cannot serve knowledge retrieval questions. When a user asks for the specific wording of a termination clause, a coverage exclusion, a financial covenant, or a data-processing obligation, the answer is not in a structured field. It is in a specific clause of a specific document. Fabric does not do semantic similarity matching, vector search, hybrid keyword-plus-vector retrieval with relevance ranking, or passage-level citation back to source documents. Forcing document-grounded questions through a system optimised for analytical summaries degrades answer quality and encourages users to treat approximate analytical answers as if they were source-grounded evidence. It also wastes the indexed document content that UC2 already produces and governs as a downstream output in Azure AI Search.

Trade-off: Simpler single-plane architecture, but unable to serve document-grounded evidence retrieval or reuse UC2 indexed outputs, leaving knowledge retrieval and cross-plane questions entirely unserved.

### Option B: Azure AI Search only

All user questions would be answered through indexed retrieval, using Azure AI Search as the sole knowledge path. The conversational agent would issue hybrid search queries (keyword plus vector) against indexes populated by UC2 extraction and embedding pipelines. This approach provides the strongest fit for document-centric evidence retrieval and the best reuse of UC2 indexed outputs. Unifying multiple knowledge sources into a single retrieval substrate simplifies the orchestration layer.

However, analytical questions are not naturally or reliably answered from indexed content. Fabric semantic-model behaviour and metric logic would need to be flattened into search artefacts, losing the fidelity and governance characteristics that Fabric provides for analytical truth. "How many active agreements does Counterparty X have?" is an aggregation query over structured data, not a retrieval problem. Trying to answer it by searching over document chunks produces unreliable results. This applies equally whether the question is about lease counts, loan facility exposure, policy renewals, or engagement totals. Treating aggregation, trend, and metric questions as if they were merely a retrieval problem misuses the tool and degrades answer quality.

Trade-off: Unified retrieval substrate with strong document-grounding capabilities, but degrades analytical fidelity by flattening Fabric semantic-model logic into indexed approximations, and wastes the structured extraction that UC2 already produces and governs in Fabric.

### Option C: Governed hybrid with both retrieval planes architecturally required (SELECTED)

Azure AI Search serves as the default retrieval plane for indexed knowledge and document-derived content. The Fabric structured-query adjunct handles questions that should be answered from structured analytical assets. The Foundry agent orchestrates retrieval across both planes for cross-plane questions. The conversational agent includes retrieval-mode classification logic that routes questions to the appropriate plane based on the nature of the question, then assembles the response with appropriate source attribution for whichever path was used. This approach matches the reality that UC2 produces two distinct downstream output categories from the same source documents, and that UC1 must serve three query pattern categories that map to those outputs.

The cost is explicit mode classification and retrieval coordination. The architecture must determine which questions go to which plane, handle ambiguous cases where both planes could plausibly respond, support cross-plane questions that require parallel retrieval from both, and maintain clear audit trails that record which retrieval path answered each question. This creates more operational complexity than a single-plane solution, but it preserves evidence-first answers for document-centric questions, semantic-model fidelity for analytical questions, and synthesised answers for cross-plane questions.

Trade-off: More complex orchestration and mode-classification logic, but preserves the fidelity and governance characteristics of both retrieval planes instead of degrading one to fit the other, and serves all three query pattern categories that users will actually ask.

---

## Decision

UC1 adopts a governed hybrid retrieval pattern that treats both the Fabric analytical plane and the Azure AI Search knowledge plane as architecturally required, not as alternatives where one might replace the other. The architecture makes the distinction between analytical truth and retrieval truth explicit instead of hiding it behind a generic "chat with data" label.

Both planes are necessary because UC2 produces two fundamentally different downstream output categories from the same source documents. Structured extraction (discrete fields written to Fabric Delta tables) serves analytical queries. Indexed document content (chunked text, embeddings, and metadata written to Azure AI Search) serves knowledge retrieval queries. These are not redundant. They represent different information about the same source material, governed through different control models, serving different query patterns that cannot be collapsed without degrading answer quality or bypassing governance boundaries. This duality is inherent to the document-intelligence pattern itself, not specific to any single industry or document type.

**Default path**: Azure AI Search serves as the primary retrieval plane for indexed knowledge retrieval, evidence retrieval, and reuse of UC2 indexed outputs. When a user asks a question that requires source-grounded evidence, document-level citations, clause-level retrieval, or cross-source search over document content, the agent routes to Azure AI Search indexes populated by UC2 pipelines.

**Adjunct path**: Governed structured querying over Fabric semantic assets or curated tables handles live analytics questions that should not be reduced to indexed approximations. When a user asks a question best answered by aggregation, trend analysis, metric computation, or filtered tabular results against governed Fabric data, the agent routes to the Fabric analytical plane. This path remains narrow and governed; it is an adjunct capability, not a parallel general-purpose knowledge layer.

**Cross-plane path**: For questions that require both structured context and document evidence, the Foundry agent issues retrieval calls to both planes and synthesises the answer. The architecture prefers an evidence-first composition where search supplies the grounding evidence and Fabric supplies structured analytical context. This cross-plane orchestration is a primary reason for selecting Foundry Agent Service as the orchestration layer in UC1-ADR-001, because it supports parallel tool invocations that Copilot Studio cannot provide.

The retrieval coordinator within the Foundry agent (see UC1-ADR-001) is responsible for classifying each question into one of the three query pattern categories, routing to the appropriate plane or planes, handling fallback when one plane is unavailable, and generating audit-trail records for each retrieval decision. The mode classification operates as follows: the coordinator evaluates the user's question against classification heuristics that distinguish analytical intent (aggregation keywords, metric requests, count and trend language) from evidence intent (clause references, specific wording requests, "show me" and "what does it say" patterns) and mixed intent (questions that reference both structured facts and document evidence). The classification decision is recorded in Azure SQL before any retrieval call is made, creating an auditable record of why the system chose a particular retrieval path.

---

## Consequences

### Positive

- Reuses both Fabric and Azure AI Search outputs from UC2 without collapsing them into one artefact type, preserving the governance and fidelity characteristics of each. Structured extraction governance (confidence scoring, human review, PII classification) and retrieval governance (document-level security, citation integrity) remain separate and appropriate to their control models.
- Provides a strong path for source-grounded evidence retrieval with document-level citations, which is a core requirement across document-intensive industries: clause-level questions in leasing, covenant questions in banking, exclusion questions in insurance, obligation questions in pharma, and liability questions in professional services all follow the same retrieval pattern.
- Avoids forcing all structured questions through search or all document questions through Fabric, keeping answer quality high for all three query pattern categories.
- Supports cross-plane questions as a first-class query pattern, enabling the system to answer questions that span structured analytical context and document evidence in a single conversational turn.
- Makes future architecture decisions easier because the retrieval modes are explicit and the routing logic is a testable component rather than an implicit assumption. New retrieval sources, document types, or industry-specific extraction schemas can be added to either plane without restructuring the other.

### Negative

- Requires a retrieval coordinator and mode classifier within the agent orchestration layer, adding engineering complexity to the conversational pipeline. The mode classifier must handle ambiguous questions where the query pattern category is not immediately obvious.
- Introduces more audit complexity because operators need to understand which retrieval path answered which question, and ambiguous mode-classification decisions must be logged and reviewable.
- Cross-plane questions introduce additional latency because the agent must wait for results from both retrieval planes before composing the answer, and mixed-source freshness mismatches must be qualified in the response.
- Raises the importance of clear grounding and fallback policies when retrieval-mode selection is uncertain, requiring explicit handling of questions that could plausibly be answered by either plane.

### Constraints introduced

- UC1 must not call UC2 raw workflow state or intermediate extraction artefacts directly. All retrieval must go through governed downstream boundaries: Azure AI Search indexes for document content, Fabric governed tables and semantic models for structured data.
- Structured-query adjacencies must remain governed and narrow; they cannot become an informal bypass around the indexed knowledge layer. Fabric is authoritative for analytics. AI Search is authoritative for evidence. Neither replaces the other.
- Azure AI Search remains the default plane for source-grounded conversational evidence, even when Fabric remains authoritative for analytics.
- Every retrieval decision must be logged with sufficient context to determine which plane answered and why, which query pattern category was classified, and whether a cross-plane query was involved, supporting audit trail requirements.
- Cross-plane queries must record retrieval results from both planes separately in the audit trail, so that the provenance of each component of a synthesised answer can be traced to its source plane.

---

## Follow-ups

- Define retrieval-mode classification heuristics and validation criteria during implementation, including handling of ambiguous questions that do not clearly map to a single query pattern category.
- Explicitly test cross-plane questions where both Fabric and Azure AI Search must be queried, validating that the synthesised answer correctly attributes each component to its source plane.
- Validate that mode classification accuracy is sufficient for production use; define a mechanism for monitoring and improving classification quality over time based on user feedback and audit trail analysis.