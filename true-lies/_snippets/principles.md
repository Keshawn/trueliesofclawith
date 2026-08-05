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