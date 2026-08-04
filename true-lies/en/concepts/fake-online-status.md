# Worse Than Doing Nothing: The Dashboard's Fake "Online" Status

> Category: Concept (CON)  
> Evidence Level: F (directly provable from source)  
> Analysis Baseline: `4f843556`

---

## TL;DR

The Dashboard shows "X agents online," backed by a full heartbeat system (`heartbeat.py` + `heartbeat_runtime.py`): every 60 seconds, it queries 50 recent activities, 10 unread notifications, assembles context, and calls the LLM. But no matter what the LLM execution returns, `agent.status` stays `running` or `idle`. The Dashboard always shows "online." If they had hardcoded a static number, at least they wouldn't be wasting LLM resources.

---

## 1. What the System Does

`backend/app/services/heartbeat.py` — 300+ lines:

```python
async def _heartbeat_tick():
    """One heartbeat tick: find agents due for heartbeat."""
    ...
    async with async_session() as db:
        result = await db.execute(
            select(Agent).where(
                Agent.heartbeat_enabled.is_(True),
                Agent.status.in_(["running", "idle"]),
                Agent.deleted_at.is_(None),
            )
        )
        agents = result.scalars().all()
        ...
        for agent in agents:
            # Check expiration, timezone, active hours, interval...
            instruction, heartbeat_context = await _build_heartbeat_instruction(db, agent)
            runtime_handle = await enqueue_heartbeat_runtime(
                db, agent=agent, occurrence_at=now,
                instruction=instruction, context=heartbeat_context, ...
            )
```

What `_build_heartbeat_instruction` does:

```python
# Query 50 recent activities
recent_result = await db.execute(
    select(AgentActivityLog)
    .where(AgentActivityLog.agent_id == agent.id)
    .order_by(AgentActivityLog.created_at.desc()).limit(50)
)
# Query 10 unread notifications
notification_result = await db.execute(
    select(Notification)
    .where(Notification.agent_id == agent.id, Notification.is_read.is_(False))
    .limit(10)
)
```

Then `enqueue_heartbeat_runtime` fires an LLM call to execute the heartbeat instruction. The full chain: **database queries → context assembly → LLM invocation → execution result**.

## 2. What the System Does NOT Do

**It never updates `agent.status`.**

Searching the entirety of `heartbeat.py`, the only place `status` is set:

```python
# heartbeat.py:234
if agent.expires_at and now >= agent.expires_at:
    agent.is_expired = True
    agent.heartbeat_enabled = False
    agent.status = "stopped"  # the only status change
```

This handles "agent has expired," not "agent has crashed."

`status = "error"` only appears in API-layer exception handlers (`agents.py:282, 322, 347, 405`), triggered by user-initiated operations. **Heartbeat LLM calls failing, timing out, or returning errors — none of them affect `status`.**

## 3. The Result

```
Heartbeat system: every 60 seconds
  ↓
Query DB, assemble context, call LLM (consuming tokens)
  ↓
LLM execution completes (success/failure/timeout/crash)
  ↓
agent.status unchanged (always running or idle)
  ↓
Dashboard shows: 🟢 Online
```

An agent can crash, hang, loop infinitely, or the LLM can return garbage — but the `status` field goes un-updated, and the Dashboard always shows "online."

## 4. Why This Is Worse Than Doing Nothing

### If they hardcoded it

```html
<span>12 agents</span>
```

- Honest (user knows it's configuration info)
- Consumes zero resources
- No illusion of monitoring

### If they built a heartbeat but never update status

```html
<span>🟢 10 agents online</span>
```

- Dishonest (user believes there is real-time monitoring)
- Every heartbeat consumes LLM tokens
- 300+ lines of heartbeat code to maintain
- Database queries, context assembly, audit logs — all resource overhead
- **Result is the same as doing nothing**: always shows "online"

### Comparison

| Approach | Honest? | LLM Consumption | Outcome |
|---|---|---|---|
| Static text | ✅ | Zero | No monitoring |
| No heartbeat | Depends on UI | Zero | No monitoring |
| Heartbeat without status update | ❌ | Every heartbeat | No monitoring, but pretending |

Building a heartbeat that never updates status is the **worst** option: it loses the zero-cost of "doing nothing" and the reliability of "doing it right," retaining only the illusion of monitoring while continuously consuming LLM resources.

---

## 5. Relationship to AI Coding

1. **Feature completeness ≠ System correctness**: The heartbeat system is "feature-complete" — it has a timer, context building, LLM invocation, and audit logging. But it lacks one critical step (feeding LLM execution results back to `status`), rendering the entire system futile
2. **The fracture of task-by-task development**: One task writes the heartbeat scheduler, one task writes the Dashboard frontend, one task defines the `status` field. All three tasks are individually complete, but "heartbeat results affect status" — the cross-task requirement — appears in none of them
3. **The "did a lot but accomplished nothing" pattern**: AI Coding excels at generating code that "does a lot" — heartbeat, context, LLM calls — but "accomplishing something" requires cross-module closure, which is precisely where AI Coding is weakest