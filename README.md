# Enterprise AI Platform: Reference Architecture

**Production-grade AI platform on Microsoft enterprise AI stack. Designed for EU-regulated industries with document-intensive operations. Architecture decisions, design rationale, and implementation guidance. One decision at a time.**

This repository documents the architecture of an enterprise AI platform that unifies document processing, knowledge retrieval, contract analysis, and audit capabilities on the Microsoft Azure stack.

There is no code here. This is an architecture reference. The design decisions, trade-off analyses, and infrastructure patterns required to take a production-grade enterprise AI system from whiteboard to production in a regulated European environment.

The use cases reflect real problems found in regulated enterprises: large volumes of complex documents across fragmented systems, regulatory compliance obligations, and the need for AI systems that are governed, auditable, and human-supervised before anything reaches production.

---

## The Problem

Across regulated industries (aviation leasing, banking, insurance, asset management, etc.) the same structural problems appear.

**Document chaos.** Decades of contracts, maintenance records, certificates, and compliance documents scattered across shared drives, legacy systems, email archives, and paper. Teams spend days locating a single missing document. That typically happens under deadline pressure, during an audit, or right before a contractual transition when it matters most.

**Disconnected teams.** Technical, financial, commercial, and legal departments all working in separate systems. No shared view of the customer, the asset, or the contractual relationship. Information moves between teams via email, spreadsheets, and meetings. Decisions are made on incomplete data because nobody has the full picture.

**Contract complexity.** Hundreds of pages per agreement. Cross-jurisdictional. Ambiguous clauses. Amendments referencing amendments. Return conditions spread across multiple documents spanning years. Nobody reads all of it. Everybody assumes somebody else has.

**Reactive compliance.** Audit preparation is a scramble. Regulatory evidence is assembled manually under time pressure. AI systems deployed without governance documentation. No confidence scores, no audit trails, no routing policy, no human review thresholds. The EU AI Act high-risk obligations take effect August 2026. Most organisations are not ready.

**Manual KYC and due diligence.** Compliance analysts assemble customer cases by hand: passports, utility bills, company registries, ownership structures, sanctions screens, adverse media checks. Documents arrive from multiple systems in multiple formats. Missing evidence is chased over email. Entity relationships are mapped in spreadsheets. A single missed document or incorrect ownership chain means onboarding a sanctioned entity or failing a regulatory exam. The work is repetitive, high-stakes, and almost entirely manual in most small and mid-sized institutions.

**Small IT teams relative to operational complexity.** These are not technology companies. They manage billions in assets with lean teams. Off-the-shelf products don't fit the operational specifics. Building from scratch is not realistic. They need someone who understands the architecture, the constraints, and the delivery path.

None of these are technology problems. They are architecture problems. The technology exists. What is missing is the design that makes it work together, governed, in production, at enterprise scale.

---

## What This Architecture Addresses

Five capabilities, built on a shared platform. Not five separate projects. One data and AI platform with five functions. One of them — Document Intelligence — is the foundation that the others build on.

**Document Intelligence (UC2).** Extraction, classification, and unification of enterprise documents into governed structured outputs for downstream analytics and retrieval. Confidence scoring on every extracted field. Extraction outcomes are routed through policy-driven controls. Some proceed directly. Some enter intermediate validation. Some require human review before operationalisation. Structured output lands in Microsoft Fabric and Azure AI Search. This is the foundational pipeline. UC1, UC3, UC4, and UC5 consume its governed outputs; they do not duplicate its ingestion, extraction, routing, or staging architecture.

**Chat with Data (UC1).** Natural language querying across structured data in Fabric and enterprise knowledge, using RAG with AI Search and Foundry models. Business users ask questions in plain English and get grounded answers with source citations. Copilot Studio as the delivery channel where it makes sense for M365 organisations. Depends on UC2 for the indexed document corpus and on Fabric for curated analytical truth.

**Contract Intelligence (UC3).** Automated comparison between sent and returned contract versions. Extraction of both documents, deterministic field-level diff, and semantic analysis of what changed and why it matters. Risk classification of deviations. Content Understanding Pro Mode with reference data for template-based comparison. Depends on UC2 for extraction and classification.

**Audit Preparation (UC4).** Internal capability that mimics what AI-powered external auditors now do. Knowledge base over policies, prior findings, regulatory requirements, and operational documentation. Proactive gap identification before auditors arrive. Conversational interface for self-assessment. Structured findings with severity and remediation. Depends on UC2 for the indexed document corpus and on UC1's retrieval patterns.

**KYC Case Assembly (UC5).** Governed assembly of compliance cases from fragmented identity, entity, and ownership documents across multiple source systems. Document-level processing inherits UC2's ingestion, extraction, validation, and staging pipeline unchanged. UC5 adds case-level orchestration, entity resolution, external validation against sanctions and registry sources, KYC-specific evidence packaging, and perpetual monitoring triggers. Human review remains mandatory for case sign-off. Depends on UC2 for all document-level processing.

---

## Architecture Principles

These are not aspirations. They are constraints that shape every design decision in this repository.

**Governance is infrastructure, not documentation.** Controls are implemented in workflows: access boundaries, logging, confidence thresholds, human review queues, rollback criteria. A PDF policy that nobody reads after the first week is not governance.

**Every AI output has a confidence score and a source citation. Human review triggers at defined control points:** when confidence drops below threshold, when the output influences a legal or financial decision, or when the use case is classified as high-risk. The rest flows through. That is how you get the efficiency of automation without the theatre of reviewing everything.

**EU data residency is enforced by architecture, not by process.** Azure Policy denies GlobalStandard model deployments. All services are region-locked to the EU. DataZone Standard ensures prompts and completions stay within EU boundaries. This is not optional. It is baked into the platform.

**Production is the learning environment.** Endless pilots and workshops that never move a KPI are not a strategy. This architecture defines a narrow, survivable path to production with explicit constraints to get there.

**Constraints first compress the timeline.** Security, compliance, operating model, failure modes are addressed up front. Not retrofitted 18 months later when someone asks "did anyone check with legal?"

**Production on GA, validate on Preview.** Production deployments depend only on GA services and API versions. Preview features are used in development to validate future capabilities and inform architecture decisions. Where an architecture option targets a preview feature, a GA fallback path is documented.

---

## Technology Stack

Built on the Microsoft enterprise stack. Not because it is the only option. Because most organisations in this space already have M365 licenses, Azure subscriptions, and Fabric capacity. Building on what they already operate is the sensible default.

**AI Services:** Microsoft Foundry Models (LLM inference, EU region-locked), Azure AI Content Understanding (document classification, extraction, inference, with Document Intelligence for deterministic extraction where generative processing is unnecessary), Azure AI Content Safety (guardrails at model endpoint).

**Data Platform:** Microsoft Fabric (Medallion Architecture, Lakehouse, Delta tables, Data Factory, Spark), Azure AI Search (hybrid retrieval, vector + keyword + semantic ranking, RAG).

**Application Runtime:** Azure Container Apps for all application components and workflow execution. Azure Service Bus for asynchronous dispatch and workflow resumption. Azure SQL Database for workflow state and operational truth.

**Identity and Security:** Microsoft Entra ID managed identities, Azure Key Vault Premium (CMK encryption, FIPS 140-3 Level 3), Azure Policy (region enforcement, SKU restrictions, key-based auth prohibition), VNet with private endpoints on all services.

**Observability:** OpenTelemetry to Application Insights. Container Apps revision and workload telemetry. Service Bus queue depth and dead-letter metrics. SQL-backed workflow traces. Audit logs.

---

## How Design Decisions Are Made

Every significant architecture choice is documented as an Architecture Decision Record (ADR) in this repository. Some decisions apply across the shared platform. Others are use-case-specific. Platform ADRs define the baseline controls, runtime, network, identity, and operating model. Use-case ADRs capture decisions specific to a capability such as document intelligence, including ingestion, routing, validation, review, and governed staging. Each ADR includes the context, the options considered with pros and cons, the decision, the consequences, and where relevant, the follow-up actions.

These are not theoretical. They come from hands-on experience building on the Microsoft AI platform in regulated environments and direct research of current platform capabilities: GA vs preview status, regional availability, private networking support, and production readiness. Where a feature is in preview or lacks critical capability (such as private networking), it is noted and an alternative is chosen.

The ADR log is the backbone of this repository. If you want to understand why any architectural choice was made, the answer is there.

ADRs are not immutable. When an implementation finding, a platform change, or a new constraint invalidates a prior decision, the original ADR is superseded by a new one. The superseded ADR remains in the repository with its status updated. The decision trail is preserved, not overwritten.

---

## EU AI Act Compliance

The EU AI Act high-risk obligations take effect in August 2026. If AI-extracted data directly influences legal or financial decisions (contract terms feeding compliance assessments, automated audit findings informing regulatory reporting), the system may be classified as high-risk under Annex III.

This architecture addresses the high-risk requirements as infrastructure, not as a post-build compliance exercise:

**Article 9 (Risk Management):** Risk classification and control mapping are defined before a single line of code ships. The AI Control Pack methodology produces a defensible control story that security teams, boards, and regulators can review.

**Article 11 (Technical Documentation):** The ADR log in this repository is the technical documentation. Every design decision, alternative considered, and rationale is recorded.

**Article 14 (Human Oversight):** Policy-driven human oversight. Extraction outcomes are routed using confidence and other control inputs. Where policy enables it, intermediate validation can enrich or challenge low-confidence outputs before final routing. Human review remains mandatory where the policy or use case requires explicit approval. Quality gates between pipeline stages prevent ungoverned outputs from reaching downstream consumers.

**Article 12 (Record-Keeping):** Every AI output includes confidence scores, source grounding, and audit trail entries. Workflow traces capture the full decision lineage from document ingestion to structured output.

---

## Repository Structure
```text
enterprise-ai-platform/
  README.md
  platform/
    README.md
    reference-architecture.md
    adrs/
    diagrams/
  use-cases/
    uc1-chat-with-data/
      README.md
      reference-architecture.md
      adrs/
      diagrams/
    uc2-document-intelligence/
      README.md
      reference-architecture.md
      adrs/
      diagrams/
    uc3-contract-intelligence/
    uc4-audit-automation/  
    uc5-kyc-case-assembly/
```

**platform/** contains the cross-cutting architecture. `reference-architecture.md` is the platform walkthrough: what services exist, how they connect, the network topology, control boundaries, and data flows. `adrs/` contains the platform decision records. `diagrams/` contains the platform infrastructure diagrams. Platform ADRs apply to every use case.

**use-cases/** contains the architecture for each capability. When a use case is documented, it follows the same structure: `README.md`, `reference-architecture.md`, `adrs/`, and `diagrams/`. UC2 (Document Intelligence) is the first fully documented use case. The other use-case folders currently exist as placeholders for future documentation.

| Use Case                   | Document                                                                                                                       | Description                                                                                                                                                                                                                                                                                               |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| UC1: Chat with Data        | [use-cases/uc1-chat-with-data/reference-architecture.md](use-cases/uc1-chat-with-data/reference-architecture.md)               | Natural language querying across Microsoft Fabric for curated analytical truth and Azure AI Search for indexed for source-grounded enterprise knowledge.                                                                                                                                                  |
| UC2: Document Intelligence | [use-cases/uc2-document-intelligence/reference-architecture.md](use-cases/uc2-document-intelligence/reference-architecture.md) | Extraction, classification, and unification of enterprise documents into governed structured outputs for downstream analytics and retrieval. One example is building a governed view of the contractual relationship with a given client from the underlying document estate and related enterprise data. |
| UC3: Contract Comparison   | Planned                                                                                                                        | Automated delta identification between sent and returned contract versions.                                                                                                                                                                                                                               |
| UC4: Audit Preparation     | Planned                                                                                                                        | Internal audit capability mimicking AI-powered external auditors, using RAG-based knowledge retrieval.                                                                                                                                                                                                    |
| UC5: KYC Case Assembly | Planned | Governed assembly of compliance cases from identity, entity, and ownership documents. Inherits UC2's document processing pipeline. Adds case-level orchestration, entity resolution, external validation, evidence packaging, and perpetual monitoring. |

There is no source code in this repository. The architecture and design decisions are the deliverables.

---

## Who This Is For

**Technical leaders** in regulated industries evaluating how to move from AI ambition to production. If your organisation has dozens of AI ideas but no structured way to evaluate, govern, and deliver them, this architecture shows one proven path.

**IT managers and heads of IT** responsible for platform decisions. If you are being asked to "do AI" but need to satisfy security, compliance, and audit before anything ships, the governance model here is designed for that conversation.

**CTOs and CIOs** If you are building an AI strategy and need the architecture layer underneath it, this is what that looks like in practice.

**Compliance and financial crime leaders** evaluating how AI can reduce KYC review times, improve case consistency, and produce regulator-ready evidence packs without replacing human accountability for risk decisions.

**Security and compliance teams** who need to understand what a governed AI system actually requires. Not a vendor slide. Not a policy document. The actual controls, thresholds, and audit mechanisms.

---

## What This Is Not

This is not a product. It is not a SaaS platform. It is not a vendor pitch.

It is a reference architecture from 20+ years in the industry, 10+ of those delivering production systems in regulated environments, and enough situations where ambition outpaced architecture to know what breaks and why.

If you want to understand how a production-grade enterprise AI platform is designed on the Microsoft stack, and why every architectural choice was made, this repository documents both transparently.

If you need help designing, governing, and delivering something like this for your organisation, that is what I do.

---

## About

Designed by Manuel Tomas Estarlich. Cloud & AI Solutions Architect with 20+ years in enterprise delivery across financial services, public sector, renewable energy, manufacturing, travel, and IT consultancy.

LinkedIn: [https://www.linkedin.com/in/manuel-tomas-estarlich/](https://www.linkedin.com/in/manuel-tomas-estarlich/)

LevelUp360: [https://levelup360.pro](https://levelup360.pro/)

Newsletter: [From Clarity to Production](https://www.linkedin.com/build-relation/newsletter-follow?entityUrn=7428478846196367362)

---

## License

MIT