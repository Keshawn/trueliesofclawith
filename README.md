# True Lies of Clawith

A case study of how contradictory product direction and AI-assisted, task-by-task implementation can interact.

**Languages:** **English** · [中文](zh/README.md)

---

## About This Site

This site publishes the findings of an independent analysis of the [Clawith](https://github.com/dataelement/Clawith) agent platform (analysis baseline: commit `4f843556`). The analysis examines the gap between the project's public claims and its source-code reality, and the role AI Coding played in creating, amplifying, or obscuring that gap.

## About the Authorship

**This document was written and analyzed primarily with the assistance of large language models (LLMs).** Evidence collection, code search, pattern recognition, causal inference, and article drafting were all performed by LLMs assisting a human researcher. The human researcher is responsible for research direction, disclosure decisions, and final conclusions.

---

## Findings

All 10 findings below have been source-verified (evidence level F).

### Concepts

- [Digital Employee, or Personal Assistant?](en/concepts/digital-employee-narrative.md)

### Architecture

- [Resource Ownership: A Unified Model That Never Formed](en/architecture/resource-ownership.md)
- [A2A Delegation & Group Chat: Outside the Work Model](en/architecture/a2a-work-model.md)

### Implementation

- [Path Boundaries: 18 Identical Fragile Checks](en/implementation/path-boundary.md)
- [Relationship Migration: Unfinished Half-Product](en/implementation/relationship-migration.md)
- [Gateway API Key: Old and New Logic Coexist](en/implementation/gateway-key-migration.md)

### Security

- [Tool/Sandbox Config Update Has IDOR](en/security/tool-config-idor.md)
- [Feishu Event Webhook: Zero Signature Verification](en/security/feishu-webhook.md)
- [Gateway API Key: Plaintext & Non-Constant-Time](en/security/gateway-key-plaintext.md)
- [Actor Fallback to Agent Creator (Confused Deputy)](en/security/actor-fallback.md)

---

## Research Principles

### 1. The question is broader than one project

> **What can happen when AI Coding makes software implementation faster than a team can form, maintain, and verify a coherent model of the whole system?**

### 2. AI Coding is not assumed to be the original cause

Our working model:

```text
ambiguous or contradictory product direction
                    ×
fast, local, task-by-task implementation
                    ×
insufficient ownership of system-wide invariants
                    ↓
contradictions are encoded, repeated, obscured, and amplified at scale
```

### 3. We distinguish evidence from interpretation

1. **Project statements** — what the project publicly claims
2. **Code and history facts** — what source code directly shows
3. **Interpretation** — what those facts imply
4. **AI Coding relevance** — what role AI Coding played in each finding

### 4. Clawith is a specimen, not the final target

**Locally plausible code does not guarantee a globally coherent or secure system.**