# Visibility ≠ Authority: The Conflated Semantics of "Visible"

> Category: Concept (CON)  
> Evidence Level: F (directly provable from source)  
> Analysis Baseline: `4f843556`

---

## TL;DR

In Clawith's Directory model, "being able to see an Agent" simultaneously carries three meanings — "can discover it," "can contact it," and "can delegate tasks to it" — but the code does not distinguish between these three things.

---

## 1. The Problem: One "Visible" Carries Too Much

The Agent Directory is the infrastructure through which Agents discover and communicate with each other. Which Agents a source Agent can "see" in the Directory is determined by `evaluate_roster_agent_visibility`.

But this function returns more than a simple "is it visible?" boolean:

`backend/app/core/permissions.py:18-23`:

```python
@dataclass(frozen=True)
class RosterVisibility:
    """Visibility result for roster-driven agent and human lookup."""

    visible: bool
    can_contact: bool
    unavailable_reason: str | None = None
```

It bundles two things together: **can you see it** (`visible`) and **can you contact it** (`can_contact`). In practice, `visible` determines whether the Agent appears in the Directory listing, while `can_contact` determines whether the `send_message_to_agent` tool is available.

`backend/app/services/agent_directory.py:118`:

```python
"contact_tools": ["send_message_to_agent"] if visibility.can_contact else [],
```

---

## 2. What Gets Conflated

In real organizational collaboration, these four things typically require different authorization decisions:

| Concept | Meaning | Typical Authorization |
|---|---|---|
| **Discoverable** | Can be found in the directory | Same tenant / department |
| **Contactable** | Can receive messages | Bilateral relationship or approval |
| **Delegate-to** | Can be assigned tasks | Managerial authority or delegation grant |
| **Manageable** | Configuration can be modified | Admin role |

But in Clawith, these boundaries are blurred:

- `visible` carries both "discoverable" and "contactable" (`can_contact` is computed by the same function)
- `can_contact` carries both "contactable" and "delegate-to" (`send_message_to_agent` supports `notify`, `consult`, and `task_delegate` modes)

---

## 3. A2A Delegation Directly Uses the Visibility Check

The most critical evidence is in the A2A runtime. When an Agent calls another Agent by UUID, `_resolve_target` uses `evaluate_roster_agent_visibility` to verify authorization:

`backend/app/services/agent_runtime/a2a_runtime.py:417-431`:

```python
visibility = evaluate_roster_agent_visibility(
    source_agent,
    target,
    authorized_custom_target=authorized_custom_target,
)
if not visibility.visible:
    raise A2ARuntimeError(
        "a2a_target_not_visible",
        f"Agent {target.name} is not visible in the source Agent's Directory",
    )
if not visibility.can_contact:
    reason = visibility.unavailable_reason or "target_unavailable"
    raise A2ARuntimeError(
        "a2a_target_unavailable",
        f"Agent {target.name} is unavailable ({reason})",
    )
```

The code means one thing clearly: **"visible in the Directory + contactable = can delegate."**

There is no separate check asking: "Is the source Agent authorized to delegate tasks to the target? Does the target Agent accept delegations from this source?"

---

## 4. Two Resolution Paths, Two Authorization Logics

`_resolve_target` actually has two paths using different authorization logic:

| Path | Trigger | Authorization Check | Location |
|---|---|---|---|
| By UUID | Caller passes `target_agent_id` | `evaluate_roster_agent_visibility` (visibility check) | `a2a_runtime.py:417-431` |
| By Name | Caller passes `agent_name` | `AgentAgentRelationship` table + `evaluate_agent_relationship_status` | `a2a_runtime.py:442-462` |

Same delegation operation, two entry points, two authorization logics. This is itself an example of implementation inconsistency.

---

## 5. The Visibility Semantics Matrix

The visibility rules from `evaluate_roster_agent_visibility` (`permissions.py:149-179`):

| Source Mode | Target Mode | Visible? | Contactable? |
|---|---|---|---|
| `private` | `private` & same creator | ✅ | Depends on status |
| `private` | `company` | ❌ | — |
| `company` | `company` | ✅ | Depends on status |
| `company` | `custom` & authorized | ✅ | Depends on status |
| `custom` | `company` | ✅ | Depends on status |
| `custom` | `custom` & authorized | ✅ | Depends on status |

The key insight: a `company`-mode Agent can see all `company` and authorized `custom` Agents. This means **a "company-visible" Agent can be contacted and delegated to by any other Agent in the same tenant** — as long as the target is running.

---

## 6. Systemic Consequences

When "visible" simultaneously carries "discoverable," "contactable," and "delegate-to":

1. **Fine-grained authorization is impossible**: You cannot make an Agent "visible but not delegatable" — the authorization check has no such dimension
2. **`company`-mode Agents are over-exposed**: Any same-tenant Agent can delegate tasks to them — this is not a design choice but the **absence of a delegation model**
3. **Adding a new authorization dimension requires touching many places**: `RosterVisibility` semantics are embedded in Directory, A2A, and tool definitions

---

## 7. Relationship to AI Coding

This defect exhibits patterns consistent with task-by-task AI-assisted development:

1. **Locally coherent concept**: `RosterVisibility` is reasonable in the Directory context — you need to know what Agents you can see and whether they are online. The problem is that this concept is reused **without modification** for A2A delegation authorization
2. **Cross-module semantic drift**: The Directory module's "visible and contactable" is used by the A2A module as "delegatable" — this kind of semantic borrowing is likely when two modules are developed under different prompts/tasks
3. **Missing global authorization layer**: If a unified "inter-Agent authorization" policy layer existed, `RosterVisibility` could not be misused as a delegation check — task-by-task AI development is precisely prone to missing such cross-module abstraction layers