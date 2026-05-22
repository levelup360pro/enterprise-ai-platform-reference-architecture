# Enterprise AI Platform

Shared runtime, networking, identity, data protection, observability, and AI capability publication for all use cases deployed on this platform, including both direct frontend access and the M365 Channels conversational path via Copilot Studio and Power Platform.

**Start here:** [reference-architecture.md](reference-architecture.md)

## Platform Architecture Documents

| Document | Purpose |
| -------- | ------- |
| [reference-architecture.md](reference-architecture.md) | Platform-wide architecture baseline, trust boundaries, and shared services |
| [ai-hub/README.md](ai-hub/README.md) | AI Hub document set for publication, consumption, and protocol patterns |
| [ai-hub/publication-model.md](ai-hub/publication-model.md) | Publication classes for models, tools, and Foundry-hosted reusable agents |
| [ai-hub/protocol-patterns.md](ai-hub/protocol-patterns.md) | API, MCP, and A2A consumption and exposure patterns |

## Platform ADRs

| ADR     | Title                                                             | Status   |
| ------- | ----------------------------------------------------------------- | -------- |
| ADR-001 | Processing Paradigm                                               | Accepted |
| ADR-002 | Runtime Platform and EU Region Selection                          | Accepted |
| ADR-003 | Network Isolation and Data Residency Enforcement                  | Accepted |
| ADR-004 | Identity, Authentication, and Authorisation                       | Accepted |
| ADR-005 | Data Protection                                                   | Accepted |
| ADR-006 | Observability and Operational Model                               | Accepted |
| ADR-007 | Copilot Studio as Conversational Interface, Microsoft Foundry as Governed Backend | Accepted |
| ADR-008 | AI Hub and AI Gateway as Shared Platform Baseline                 | Accepted |
| ADR-009 | Publication Model for Shared Models, Tools, and Reusable Agents   | Accepted |
| ADR-010 | MCP Exposure Through the AI Hub                                   | Accepted |
| ADR-011 | Agent-to-Agent Discovery and Invocation Through the AI Hub        | Accepted |

## Diagrams

| Diagram | Path |
| ------- | ---- |
| Authoritative infrastructure baseline | [Authoritative infrastructure baseline](diagrams/enterprise-ai-platform-reference-architecture-Infrastructure%20View.png) |

## Use Cases

All use cases deploy on this platform and inherit its controls. Use-case-specific architecture is documented under `use-cases/`.
