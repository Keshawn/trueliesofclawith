---

## Part III · Complete Index

All findings have been source-verified (evidence level F, analysis baseline `4f843556`).

### Dimension 1 · Cognitive Pipeline & Compute Scheduling (5 articles)

| # | Article | In One Sentence |
|---|---------|-----------------|
| 018 | [One Pipe: Every Tool Output Is a Conversation](topics/018.md) | All tool outputs—stdout, stderr, file contents—are dumped into `messages[]` as conversation, with no audit trail |
| 019 | [A Dumpster You Throw Everything Into](topics/019.md) | soul.md, memory.md, skills/, focus, triggers, and enterprise_info are concatenated into a single system message—no layering, no caching |
| 020 | [Fake Caching and the Heartbeat Tax: How to Burn Your Money](topics/020.md) | Prompt caching is disabled for all providers except Qwen; heartbeat fires a full LLM session every 4 hours with zero concurrency control |
| 021 | [When "Everything Is a File" Becomes "Everything Is a Disaster"](topics/021.md) | Memory is a global markdown file shared by all users, injected into the system prompt—any user can poison it |
| 022 | [The Zero Processing Philosophy](topics/022.md) | The root cause of the four above: CLA.md declares "ONE set of file tools covers EVERYTHING"—and AI faithfully obeyed |

### Dimension 2 · Sandbox Boundaries & Tenant Isolation (2 articles)

| # | Article | In One Sentence |
|---|---------|-----------------|
| 001 | [Path Boundaries: 18 Identical Fragile Checks](topics/001.md) | 18 instances of `str(path).startswith(str(base))` across the codebase, all missing the path separator check—no shared abstraction |
| 002 | [Tool/Sandbox Config Update Has IDOR](topics/002.md) | Any logged-in user can modify any Agent's sandbox type, URL, and API key—the endpoint checks only that the user is logged in |

### Dimension 3 · External Communication & Network Trust (1 article)

| # | Article | In One Sentence |
|---|---------|-----------------|
| 003 | [Feishu Event Webhook: Zero Signature Verification](topics/003.md) | `verification_token` and `encrypt_key` are stored in the database but never read—anyone who knows the `agent_id` can forge Feishu events |

### Dimension 4 · Lifecycle & Visibility Control (2 articles)

| # | Article | In One Sentence |
|---|---------|-----------------|
| 005 | [Worse Than Doing Nothing: The Dashboard's Fake "Online" Status](topics/005.md) | Heartbeat fires every 60s, queries DB, assembles context, calls the LLM—but never updates `agent.status`; the Dashboard always shows "online" |
| 006 | ["All" Is Not All: Systemic Inconsistency of Totality Claims](topics/006.md) | Activity Feed, Directory, and `custom` mode all claim to show "all" of something—none actually include everything |

### Dimension 5 · Digital Assets & Credential Governance (2 articles)

| # | Article | In One Sentence |
|---|---------|-----------------|
| 009 | [Gateway API Key: Plaintext Storage & Non-Constant-Time Comparison](topics/009.md) | Verification tries plaintext first, uses Python `==` (not constant-time), SHA256 has no salt—comments label plaintext as "new behavior" |
| 010 | [Gateway API Key: Old and New Logic Coexist](topics/010.md) | Creation uses SHA256 hash; verification tries plaintext first—the two paths are inconsistent, and the migration has no completion plan |

### Dimension 6 · Organizational Topology & Collaboration (9 articles)

| # | Article | In One Sentence |
|---|---------|-----------------|
| 004 | [Actor Fallback to Agent Creator (Confused Deputy)](topics/004.md) | When `actor_user_id` is `None`, the system silently falls back to `agent.creator_id`—a Confused Deputy granting unauthorized admin privileges |
| 007 | [Resource Ownership: A Unified Model That Never Formed](topics/007.md) | 8/13 resource models lack `tenant_id`; credentials scattered across 5+ locations; output artifacts use 4 different persistence methods |
| 008 | [A2A Delegation & Group Chat: Outside the Work Model](topics/008.md) | Delegated runs bypass the Focus work-item model and OKR system, and don't write to `AgentActivityLog`—producing un-auditable ghost work |
| 011 | [Digital Employee, or Personal Assistant?](topics/011.md) | The README promises "digital employees," but the code has no `agent` RBAC role, identity is derived from `creator_id`, and quotas live on the User table |
| 012 | [Visibility ≠ Authority: The Conflated Semantics of "Visible"](topics/012.md) | "Visible" simultaneously means discoverable, contactable, and delegatable—the code has no separate authorization for delegation |
| 013 | [Communication = Delegation: No Separation of Contact and Task Authority](topics/013.md) | `send_message_to_agent` bundles `notify`, `consult`, and `task_delegate` into one tool behind a single `can_contact` gate |
| 014 | [A System Without Access Control: Everyone Is Equal in Company Mode](topics/014.md) | `company` mode gives every user identical permissions—no departments, no role hierarchy, and `use`/`manage` distinction is bypassed by A2A delegation |
| 015 | [Blind and Deaf: The "Private" Mode Isolation Paradox](topics/015.md) | `private` Agents can only see same-creator `private` Agents—users must choose between "secure but useless" and "useful but naked" |
| 016 | [The "Custom" Mode Illusion: Visibility Control That Doesn't Exist](topics/016.md) | `custom` mode controls who *you* can see, not who can see *you*—any admin can add you from any non-private Agent's perspective |

### Independent Signal · Zombie Features & Implementation Debt (1 article)

| # | Article | In One Sentence |
|---|---------|-----------------|
| 017 | [Relationship Migration: An Unfinished Half-Product](topics/017.md) | 9 REST APIs and ~700 lines of frontend code remain fully intact after migration—unused, uncleaned, and the rich relationship model was degraded to a hardcoded `"collaborator"` |
---
