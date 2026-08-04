# True Lies of Clawith — Revision in Progress

> **Status (2026-08-03):** The previously published case-study pages have been temporarily withdrawn while we reorganize the evidence, argument, and publication structure. They will return selectively after review.

**Languages:** **English** · [中文](zh/README.md)

---

## About the Authorship of This Document

**This document was written and analyzed primarily with the assistance of large language models (LLMs).**

Evidence collection, code search, pattern recognition, causal inference, and article drafting throughout this research were all performed by LLMs assisting a human researcher. This is itself a meta-level dimension of the case study: the process of examining the consequences of AI Coding is itself a product of AI Coding.

We do not hide this fact. On the contrary, it demonstrates the core thesis of this document: AI Coding can rapidly produce locally plausible, verified text and code, but the final judgment—which facts are worth publishing, how to organize the argument, and what responsibility to bear toward readers—remains a human decision.

In this document, the human researcher is responsible for:

- Defining research direction and scope
- Deciding which findings enter formal publication
- Assessing disclosure risk
- Bearing final responsibility for published conclusions

The LLM is responsible for:

- Searching and locating source-code evidence
- Identifying code patterns
- Drafting analysis
- Formatting and organizing text

---

## Part I · Research Principles

### 1. The question is broader than one project

This project uses Clawith as a case study to examine a broader question:

> **What can happen when AI Coding makes software implementation faster than a team can form, maintain, and verify a coherent model of the whole system?**

The purpose is not merely to list defects in Clawith, nor to claim that every defect in the project was caused by AI.

### 2. AI Coding is not assumed to be the original cause

Some candidate findings concern choices that precede implementation: product positioning, domain concepts, governance assumptions, and mutually incompatible requirements. Those problems can exist whether code is written by humans or AI.

Our working model is therefore not:

```text
AI Coding → every project defect
```

It is:

```text
ambiguous or contradictory product direction
                    ×
fast, local, task-by-task implementation
                    ×
insufficient ownership of system-wide invariants
                    ↓
contradictions are encoded, repeated, obscured, and amplified at scale
```

AI Coding may be an **accelerator, amplifier, implementation shaper, or obscuring layer**. Its role must be assessed separately for each finding.

### 3. We distinguish evidence from interpretation

The revised publication will keep these layers separate:

1. **Project statements** — what the project publicly claims.
2. **Code and history facts** — what source code and commits directly show.
3. **Runtime behavior** — what can be reproduced under stated conditions.
4. **Interpretation** — what those facts imply about the product or architecture.
5. **AI Coding relevance** — whether AI Coding plausibly created, amplified, shaped, obscured, or had no demonstrated relationship to the issue.

Code patterns alone do not prove who authored a particular change. We will not present correlation as authorship proof.

### 4. Clawith is a specimen, not the final target

Clawith is useful because it is a real, non-trivial agent platform with public attention and broad functionality: identity, permissions, multi-agent communication, tools, code execution, external integrations, and multi-tenant behavior. These interacting boundaries make it suitable for studying system-level consequences.

The broader lesson concerns engineering practice: **locally plausible code does not guarantee a globally coherent or secure system.**

### 5. Publication discipline

During the revision:

- Existing material is treated as a pool of candidate facts and interpretations.
- Duplicate claims will be merged and reordered before publication.
- Only selected claims will receive final source, version, runtime-condition, and remediation-status verification.
- Security details will be reviewed for responsible-disclosure risk before republication.
- Historical findings will be clearly distinguished from current behavior.
- Corrections and uncertainty will be stated explicitly.

## Part II · Findings

All findings have been source-verified (evidence level F, analysis baseline `4f843556`). Please use the site navigation to browse by category. Also available in [中文](zh/README.md).

---

## What will follow

More findings will be published as they are verified. Security findings (SEC-*) will not be disclosed in detail before upstream fixes are confirmed. The English version will be translated in sync after the Chinese structure and arguments stabilize.