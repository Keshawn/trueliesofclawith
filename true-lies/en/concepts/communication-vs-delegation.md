# Communication = Delegation: A2A Contact and Task Delegation Are Not Separated

> Category: Concept (CON)  
> Evidence Level: F (directly provable from source)  
> Analysis Baseline: `4f843556`

---

## TL;DR

In Clawith, as long as an Agent can send a message to another Agent (`send_message_to_agent`), it automatically has the ability to delegate tasks to it (`task_delegate` mode) — there is no authorization boundary between communication and delegation.

---

## 1. One Tool, Three Modes

The sole tool for inter-Agent communication is `send_message_to_agent`. It supports three modes:

`backend/app/services/builtin_tool_definitions.py:526-537`:

```python
"name": "send_message_to_agent",
"description": "Send a private A2A message to a digital employee from query_directory. "
                "notify completes after the durable send. consult and task_delegate "
                "create a delegated Run; this Run waits and resumes with the correlated result...",
"msg_type": {"type": "string",
             "enum": ["notify", "consult", "task_delegate"],
             "description": "(1) Target needs to DO WORK and return results? → task_delegate. "
                            "(2) Just FYI? → notify. "
                            "(3) Quick factual question? → consult. "
                            "When unsure, prefer task_delegate."},
```

The three modes differ in meaning:

| Mode | Meaning | Target Agent Behavior |
|---|---|---|
| `notify` | Notification | Async processing; source does not wait |
| `consult` | Consultation | Creates a delegated Run; source waits for result |
| `task_delegate` | Task Delegation | Creates a delegated Run; source waits for result |

---

## 2. The Problem: No "Delegation Authority" Layer

`notify` and `task_delegate` are semantically completely different:

- **Notify**: I'm telling you something; you don't need to respond
- **Delegate**: I'm asking you to do something; give me the result

In real organizational collaboration, these are two different authorizations. Being able to message a colleague does not mean being able to assign them work. But in Clawith, both capabilities are controlled by **the same tool**, subject to **the same permission check**.

---

## 3. The Single Gate: `can_contact`

The gate into `send_message_to_agent` is the `can_contact` flag, derived from the Directory visibility evaluation:

`backend/app/services/agent_directory.py:118`:

```python
"contact_tools": ["send_message_to_agent"] if visibility.can_contact else [],
```

The `can_contact` computation logic (`permissions.py:149-179`) only considers three things:

1. Whether source and target Agents share a tenant
2. Whether their `access_mode` values are compatible (`company` can see `company`; `private` only sees same-creator `private`)
3. Whether the target Agent is online / not expired

It does not check:

- Whether the source Agent is authorized to delegate tasks to the target
- Whether the target Agent accepts delegations from this source
- Whether the delegation chain should be restricted (e.g., allow `notify` but not `task_delegate`)

---

## 4. Verification in the A2A Runtime

When `send_message_to_agent` is invoked, the A2A runtime's `_resolve_target` handles authorization verification. But as noted, it uses either the visibility check (UUID path) or relationship table check (name path):

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
    ...  # reject
# ← after passing, notify / consult / task_delegate are treated identically
```

After the `evaluate_roster_agent_visibility` call, all three modes are treated identically — there is no additional check specific to `task_delegate`.

---

## 5. Consequence: Full Exposure of `company` Agents

If an Agent's `access_mode` is `company` (the default), it is visible and contactable by all non-`private` Agents in the same tenant. This means:

- **Any** `company` or `custom` Agent in the same tenant can send it `task_delegate` requests
- The target Agent will create new Runs, execute operations, and return results as directed by the delegator
- There is no configuration option for "allow communication but deny delegation"

This is not a "configuration mistake" — it is the result of **the architecture having no delegation authorization layer at all**.

---

## 6. Comparison: Industry Practice

In RPA and AI platform domains, communication and delegation are typically explicitly separated:

| Platform | Communication | Delegation | Approval |
|---|---|---|---|
| UiPath | Robots communicate via Orchestrator queues | Requires explicit delegation configuration | Supports approval flows |
| ServiceNow | Virtual Agents communicate via Flow | Requires Delegation rules | Built-in approval |
| Clawith | `send_message_to_agent` | Same tool, no extra check | None |

---

## 7. Relationship to AI Coding

This defect exhibits patterns consistent with task-by-task AI-assisted development:

1. **Lack of granularity at the tool definition level**: The three modes of `send_message_to_agent` are a flat enum — this could be the product of a single prompt, with no subsequent "we need to split this tool" iteration
2. **Authorization check stays at the "can communicate" level**: When implementing `_resolve_target`, the AI reused the existing `evaluate_roster_agent_visibility` function without asking "does delegation need additional authorization?" — a classic "locally usable, globally missing" pattern
3. **Missing cross-tool invariants**: If a unified "inter-Agent authorization matrix" policy layer existed, the three modes of `send_message_to_agent` could not be covered by the same single check — task-by-task AI development is precisely prone to missing such cross-module abstractions