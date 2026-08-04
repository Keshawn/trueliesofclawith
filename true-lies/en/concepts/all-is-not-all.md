# "All" Is Not All: Systemic Semantic Inconsistency of Totality Claims

> Category: Concept (CON)  
> Evidence Level: F (directly provable from source)  
> Analysis Baseline: `4f843556`

---

## TL;DR

Multiple features in Clawith claim to show "all" of something, but none actually include everything. This is not an isolated bug — it is a systemic pattern: **the word "all" has never been aligned between UI and backend across the system.**

---

## 1. Example 1: Activity Feed's "All Activities"

The Dashboard's "Recent Activity" panel shows agent activity from the `AgentActivityLog` table.

`backend/app/models/activity_log.py` — the `action_type` enum:

```python
action_type: Enum(
    "chat_reply", "tool_call", "feishu_msg_sent", "agent_msg_sent",
    "web_msg_sent", "task_created", "task_updated", "file_written",
    "error", "schedule_run", "heartbeat", "plaza_post",
)
```

**Missing**: `a2a_delegated`, `consult`, `notify`, `task_delegated`.

Any work done through A2A delegation — Agent A delegates a task to Agent B, Agent B completes it and returns the result — **does not appear in the activity log**. The Dashboard's "all activities" actually excludes all inter-agent collaboration.

---

## 2. Example 2: Directory's "All Agents"

The `query_directory` tool returns "the list of agents in the current tenant," but `_agent_directory_conditions` filters by `access_mode`:

- A `private` source agent can only see same-creator `private` agents
- A `company` source agent cannot see `private` agents

"All" actually means "the subset visible to your access_mode."

---

## 3. Example 3: `custom` Mode's "Custom Visibility"

The `custom` mode promises users can customize "who can see my agent." In reality:

- `AgentAgentRelationship` is a **viewer-side configuration** (`agent_id` = who is looking), not a viewed-entity policy
- Any admin can add you from any non-private agent's perspective

"Custom" actually means "anyone can customize seeing you."

---

## 4. These Three Examples Are Not Isolated

| Feature | Claims | Actual |
|---|---|---|
| Activity Feed | All activities | Missing A2A delegation |
| Directory | All agents | Missing cross-access_mode agents |
| `custom` mode | Custom visibility | Viewer-side control, viewed has no say |

Three examples from three different modules (Activity Log, Directory, Permissions), all exhibiting the same pattern:

```
UI/product level: promises "all"
    ↓
Code level: implements "the subset visible under current local conditions"
    ↓
Result: the semantics of "all" are never globally defined or enforced
```

---

## 5. Root Cause: No Global Definition of "All"

In a correct system, "all" must be explicitly defined:

```
All activities = all action_type values
All agents = all agents in the tenant (visibility filtering is a separate layer)
All visibility = visibility policy controlled by the viewed entity
```

But Clawith has no such global definition. Each module interprets "all" locally, the frontend displays its own understanding, and the backend filters with its own. **The word "all" is scattered across local implementations and never aligned.**

---

## 6. Relationship to AI Coding

1. **Locally coherent, globally drifting semantics**: Each module's "all" has its own definition in its local context (Activity Log's "all" = the types in the enum; Directory's "all" = agents visible to the current access_mode). Each module is "correct" in isolation, but "all" loses consistency across modules
2. **No global concept registry**: When AI Coding generates code task by task, no one asks "wait, what does 'all' actually mean in this system?" — each task defines "all" within its own scope
3. **UI text and backend logic are never cross-validated**: The frontend writes "all activities," the backend writes `select * from agent_activity_logs` — both are locally correct, but no one checks "does the backend actually return everything?"