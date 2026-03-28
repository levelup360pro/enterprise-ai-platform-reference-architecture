# UC2-ADR-002: Post-Extraction PII Classification

**Status**: Draft  
**Date**: 16/03/2026  
**Decision Scope**: When and how PII is identified in UC2 extraction outputs.  
**Depends on**: Enterprise AI Platform reference architecture, ADR-001 (Processing Paradigm), ADR-003 (Network Isolation and Data Residency Enforcement), ADR-004 (Identity, Authentication, and Authorisation), UC2-ADR-001 (Layered Hexagonal Architecture), UC2-ADR-004 (Worker as the Single Orchestration Owner)  
**Depended on by**: UC2-ADR-006 (Staging as the Governed Downstream Boundary)

---

## Context

UC2 extracts structured data from enterprise documents containing parties, dates, jurisdictions, obligations, and financial terms. This data routinely includes personally identifiable information: names, addresses, signatures, company registration numbers, and in some cases financial account details.

The system must identify PII-bearing fields so that downstream consumers can apply appropriate access controls, masking, and audit policies. The question is where in the pipeline that identification happens and what action is taken on the result.

Content Understanding uses LLMs to perform classification and extraction. Redacting PII before extraction would remove the signals that extraction depends on: party names, addresses used to infer jurisdiction, signature blocks that confirm execution. Pre-extraction sanitisation would degrade extraction accuracy for the data that matters most.

Foundry's built-in content safety filters (ADR-001) operate at the prompt and completion level. They filter harmful content categories. They do not perform entity-level PII detection or produce field-level PII metadata for downstream governance.

---

## Decision Drivers

- Extraction accuracy depends on PII-bearing content (party names, addresses, financial terms). Removing or masking this content before extraction reduces the value of the output.
- Downstream consumers need to know which fields contain PII so they can apply access controls, masking, or elevated audit. This requires structured PII metadata, not redaction.
- PII detection must use a dedicated capability, the Azure AI Language capability, not the same content safety layer that filters harmful content at the LLM prompt level.
- The Worker owns all external service calls and staging writes (UC2-ADR-004). PII classification must fit within that orchestration model.

---

## Considered Alternatives

### Option A: Pre-extraction PII redaction

Run the Azure AI Language capability, accessed through the Microsoft Foundry resource, on the raw document before sending it to Content Understanding. Redact or mask identified PII. Send sanitised content for extraction.

Trade-off: Protects PII from reaching the LLM. But destroys extraction accuracy. Party names become placeholders. Addresses used for jurisdiction inference are removed. Signature blocks are stripped. The extraction output loses the fields that give it business value. Not viable for UC2.

### Option B: No explicit PII step

Rely on Foundry content safety filters and downstream access controls. Do not run a dedicated PII detection pass.

Trade-off: Simpler pipeline. But downstream consumers receive structured data with no metadata indicating which fields contain PII. Access control decisions are left to the consumer with no signal from the pipeline. Governance depends on consumers independently identifying PII in extraction outputs, which is inconsistent and unauditable.

### Option C: Post-extraction PII classification (SELECTED)

Run the Azure AI Language capability, accessed through the Microsoft Foundry resource, on the accepted structured output after confidence evaluation and human review, if applicable. Classify fields with PII metadata. Write PII-classified output to staging. Do not redact.

Trade-off: PII-bearing content reaches the LLM during extraction. This is accepted because Content Understanding requires it for accurate extraction. The mitigation is that Foundry processes data within EU regional boundaries (ADR-003), under managed identity (ADR-004), with content safety filters active (ADR-001). Microsoft's data processing terms for Foundry services state that customer data is not used to train or improve foundation models. PII is identified and classified before it leaves the UC2 pipeline boundary, giving downstream consumers the metadata they need.

---

## Decision

UC2 classifies PII-bearing fields after extraction, not before.

The Worker calls the Azure AI Language capability, accessed through the Microsoft Foundry resource, on accepted structured output. This happens after confidence evaluation and, where applicable, after human review approval. The output is structured PII metadata associated with the accepted extraction output: which output fields or values contain PII, the detected category, and the confidence of the detection.

PII metadata is written alongside the structured extraction output to staging. No redaction is performed. The extraction output retains its full content. Downstream consumers use the PII metadata to support access control, masking, and audit policies appropriate to their context.

PII classification is a core domain capability (UC2-ADR-001). It defines a protocol for PII detection. An infrastructure adapter implements that protocol against the Azure AI Language capability accessed through the Microsoft Foundry resource. The Worker orchestrates the call and writes the result. No other component in the pipeline performs PII detection or writes PII metadata.

---

## Consequences

### Positive

- Extraction accuracy is preserved. Content Understanding receives the full document content it needs for accurate classification and extraction.
- Downstream consumers receive explicit, structured PII metadata produced by a dedicated detection capability. Access control decisions are informed by the pipeline, not left to consumer interpretation.
- PII classification is auditable. The PII detection call, its output, and the resulting metadata are logged as part of the workflow state in Azure SQL.

### Negative

- PII-bearing content is processed by Content Understanding during extraction. This is mitigated by Foundry's EU regional deployment, managed identity access, content safety filters, and private endpoint connectivity. Microsoft's data processing terms for Foundry services state that customer data is not used to train or improve foundation models, but the data does reach the LLM.
- PII detection adds a processing step and a service call per document. For high-volume ingestion, this adds latency and cost to the pipeline.
- The quality of PII classification depends on the quality of the accepted extraction output. If extraction is poor, even after human review, PII detection may miss fields or misclassify them.

### Constraints introduced

- PII detection must run after confidence evaluation and human review, never before extraction.
- PII metadata must be persisted alongside the structured output in staging. They are not separate artefacts with separate lifecycles.
- No component other than the Worker may invoke PII detection or write PII metadata to staging.
- PII classification is a governance enrichment step. It does not alter the accepted extraction payload.
