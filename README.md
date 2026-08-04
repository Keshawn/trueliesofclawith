# True Lies of Clawith

A case study of how contradictory product direction and AI-assisted, task-by-task implementation can interact.

> **📖 Read the full findings at [keshawn.github.io/trueliesofclawith](https://keshawn.github.io/trueliesofclawith/)** · [中文版](https://keshawn.github.io/trueliesofclawith/zh/)

---

## About This Site

This site publishes the findings of an independent analysis of the [Clawith](https://github.com/dataelement/Clawith) agent platform (analysis baseline: commit `4f843556`). The analysis examines the gap between the project's public claims and its source-code reality, and the role AI Coding played in creating, amplifying, or obscuring that gap.

All findings are published as a bilingual MkDocs site deployed via GitHub Pages. The full index of findings is maintained on the site itself — see the link above.

## About the Authorship

**This document was written and analyzed primarily with the assistance of large language models (LLMs).** Evidence collection, code search, pattern recognition, causal inference, and article drafting were all performed by LLMs assisting a human researcher. The human researcher is responsible for research direction, disclosure decisions, and final conclusions.

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