# Blind and Deaf: The `private` Mode Isolation Paradox

> Category: Concept (CON)  
> Evidence Level: F (directly provable from source)  
> Analysis Baseline: `4f843556`

---

## TL;DR

A `private`-mode Agent can only see other `private` Agents created by the same user — it is completely blind to `company` and `custom` Agents. The result: users must choose between "secure but useless" and "useful but naked."

---

## 1. The Product Promise: Multi-Agent Collaboration

Clawith's README claims Agents can communicate, delegate tasks, and collaborate via A2A. The ideal scenario:

> Your "personal secretary" automatically delegates a leave request to the company's "HR assistant," processes the reply, and summarizes the result for you.

This requires `private` Agents to at least **see** `company` Agents.

---

## 2. The Code Reality: `private` Is Completely Blinded

`backend/app/core/permissions.py:149-179`:

```python
def evaluate_roster_agent_visibility(source_agent, target_agent, ...):
    source_mode = _agent_access_mode(source_agent)
    target_mode = _agent_access_mode(target_agent)
    visible = False

    if source_mode == "private":
        visible = (
            target_mode == "private"
            and getattr(source_agent, "creator_id", None) == getattr(target_agent, "creator_id", None)
        )
    else:
        visible = target_mode == "company" or (target_mode == "custom" and authorized_custom_target)
```

When the source Agent is `private`:

```python
visible = (target_mode == "private" and source.creator_id == target.creator_id)
```

It can **only** see other `private` Agents created by the same person. `company` and `custom` Agents are completely excluded.

---

## 3. The Consequence Matrix

| Source Agent | Target Agent | Visible? |
|---|---|---|
| `private` | `private` (same creator) | ✅ |
| `private` | `private` (different creator) | ❌ |
| `private` | `company` | ❌ |
| `private` | `custom` | ❌ |
| `company` | `private` | ❌ |
| `company` | `company` | ✅ |

Two worlds, completely isolated — **zero overlap** between `private` and `company`.

---

## 4. Why This Design: Connection = Trust

In Clawith's A2A model, once Agent A can see Agent B, A can delegate tasks to B (see CON-005). There is no authorization layer between "visible" and "delegatable."

If `private` Agents could see `company` Agents, and the reverse were also true — `company` Agents could see `private` Agents, delegate tasks to them, and access their private data. This would be a cross-user Confused Deputy attack.

Instead of implementing fine-grained inter-Agent authorization, the developers simply **severed all connections between `private` and `company`.**

---

## 5. The Paradox: Secure vs. Useful

This design creates an irreconcilable contradiction:

```
private Agent → secure (no one can see you) → but can't collaborate with any company Agent → useless
company Agent → can collaborate with all company Agents → but everyone can see you → naked
```

Users are forced to choose between "secure but useless" and "useful but naked." There is no middle ground that is "secure and useful."

---

## 6. With `custom` Mode in Context

Combined with the `custom` mode illusion, the actual effect of all three modes:

| Mode | Who You Can See | Who Can See You | Can Collaborate? |
|---|---|---|---|
| `private` | Only same-creator private | Only same-creator private | ❌ Cannot reach company Agents |
| `company` | All company + authorized custom | All non-private | ✅ But fully exposed |
| `custom` | All company + added custom | Any non-private (= company) | ✅ But fully exposed |

The only mode that protects privacy is the only mode that can't participate in multi-Agent collaboration.

---

## 7. Relationship to AI Coding

This defect exhibits patterns consistent with task-by-task AI-assisted development:

1. **Locally optimal security fix**: Upon discovering the "connection = trust" vulnerability, the quickest fix is "sever the connection" — the fastest solution a single prompt can produce. But it didn't consider the impact on product functionality
2. **Missing cross-concern trade-off**: Security (no unauthorized access) and functionality (multi-Agent collaboration) were treated as separate tasks. No one asked at the architecture level: "can both coexist?"
3. **Semantic degradation of `private`**: `private` should mean "my data is only accessible to me," but the code turned it into "my Agent is isolated from the world" — a classic case of a concept model being oversimplified in implementation