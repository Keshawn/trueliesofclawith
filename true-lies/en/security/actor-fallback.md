# Actor Fallback to Agent Creator (Confused Deputy)

> Category: Security (SEC)  
> Evidence Level: F (directly proven by source code)  
> Analysis Baseline: `4f843556`

---

## TL;DR

When `context.actor_user_id` is `None` (anonymous/unauthenticated/delegated execution), the system silently falls back to `agent.creator_id` (typically an admin), creating a Confused Deputy vulnerability.

---

## 1. Fallback Point 1: File Deletion

`backend/app/services/agent_runtime/tool_step_service.py:494`:

```python
actor_user_id = context.actor_user_id or str(agent.creator_id)
```

This fallback is used for permission decisions on `delete_file` and `GROUP_DELETE_WORKSPACE_FILE`. When `actor_user_id` is `None`, file deletion executes with the identity of the Agent's creator (typically an admin).

---

## 2. Fallback Point 2: General Tool Execution

`tool_step_service.py:1879`:

```python
raw_result = await self._tool_executor(
    tool_name,
    arguments,
    agent.id,
    context.actor_user_id and uuid.UUID(context.actor_user_id)
        or agent.creator_id,
    context.session_id or "",
)
```

Same pattern—when `actor_user_id` is `None`, falls back to `agent.creator_id`.

---

## 3. actor_user_id Can Be None

`backend/app/services/agent_runtime/state.py:214`:

```python
class RuntimeContext:
    actor_user_id: str | None = None
```

The type of `actor_user_id` is `str | None`—the system is designed to allow it to be `None`.

---

## 4. Trigger Scenarios

Scenarios where `actor_user_id` may be `None` include:

- **Delegated execution** (delegated run): A2A delegation, OpenClaw gateway messages
- **Group chat context**: Agent-to-Agent `@` calls in group chats, where the original user context may be lost
- **Scheduled tasks/triggers**: Scheduler-triggered execution has no human user context
- **Anonymous execution**: Runs invoked directly via API

---

## 5. Confused Deputy Attack Chain

```
Ordinary employee Jerry @-mentions an Agent created by admin Tom in a group chat
  → Agent processes the group message; context.actor_user_id may be None
  → Tool execution falls back to agent.creator_id = Tom (admin)
  → Jerry performs file operations/tool calls with Tom's admin privileges
```

---

## 6. Relationship to AI Coding

This `or` fallback is the classic "just make it work" development pattern:

- A developer encounters a failure when `actor_user_id` is `None` on some path
- They add `or str(agent.creator_id)` as a "fallback" to make it work
- They don't realize this grants unauthorized users admin-level privileges
- The two fallback points (line 494 and line 1879) were implemented independently, suggesting the same problem was patched twice, each time in the same way