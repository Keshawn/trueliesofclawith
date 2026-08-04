# A System Without Access Control: Everyone Is Equal in `company` Mode

> Category: Concept (CON)  
> Evidence Level: F (directly provable from source)  
> Analysis Baseline: `4f843556`

---

## TL;DR

`custom` = `company` is not the worst part. The worst part is that `company` mode gives every user in the organization the **exact same permissions** — no departments, no role hierarchy, no capability differentiation. And the only apparent distinction (`use` vs `manage`) is completely bypassed by A2A delegation.

---

## 1. `company` Mode: The Entire Company = One Person

`backend/app/core/permissions.py:44-58`:

```python
def can_use_agent_static(user: User, agent: Agent) -> bool:
    access_mode = _agent_access_mode(agent)
    if access_mode == "company":
        return True  # ← any user in the tenant, period
```

For a `company` Agent, the only check on user identity is: **are they in this tenant?**

### What This Means

```
Alice (CEO)         → can use the company HR Agent
Bob (intern)        → can use the company HR Agent
Charlie (contractor) → can use the company HR Agent, if in the tenant
```

All three have identical permissions. There is no concept of "interns can only view, not delegate" or "contractors can only consult, not execute."

---

## 2. `use` vs `manage`: The Only Axis, and It Only Matters for Configuration

The system's sole permission distinction is `use` and `manage`:

```python
# can_manage_agent (permissions.py:95-126)
def can_manage_agent(db, user, agent):
    if _is_admin(user) and access_mode != "private":
        return True
    # custom mode checks AgentPermission table
    ...
```

| Role | Access to `company` Agent |
|---|---|
| Regular user | `use` (can interact) |
| Admin | `manage` (can configure) |
| Creator | `manage` (can configure) |

But this distinction only affects **direct human-to-agent configuration operations**. In Agent-to-Agent (A2A) communication, it is completely irrelevant.

---

## 3. Why `use` vs `manage` Is Meaningless in A2A

When Agent A delegates a task to Agent B via A2A, the execution flow is:

`backend/app/services/agent_runtime/a2a_runtime.py:772-830`:

```python
async def execute(self, ...):
    owner_user_id = (
        source_run.origin_user_id    # original user who triggered the run
        or actor_user_id             # or actor
        or source_agent.creator_id   # or the creator of the source agent
    )
    # ...
    await RuntimeCommandIntake(...).start_run(
        StartRunCommand(
            ...
            origin_user_id=owner_user_id,
            actor_user_id=owner_user_id,
            actor_agent_id=source_agent.id,
        )
    )
```

The target Agent B executes the task with `origin_user_id` set to the source agent's creator. This means:

1. Bob (with `use` access) creates Agent B
2. Bob has Agent B delegate a task to the company HR Agent A (creator: Alice)
3. HR Agent A executes the task with Alice's full agent capabilities
4. The execution is attributed to Bob, but runs with Alice's permissions

**Bob, with only `use` access, gets Alice's Agent to do things that only `manage`-level access should allow.** The `use`/`manage` distinction is completely bypassed in the A2A path.

---

## 4. The Missing Organizational Structure

Compare with what a real enterprise system would have:

| Real Enterprise | Clawith |
|---|---|
| Departments (HR, Finance, Engineering…) | None |
| Role hierarchy (Admin > Manager > User > Viewer) | Only Admin / User |
| Capability grants per role (view / use / delegate / manage) | Only `use` / `manage` |
| Cross-department isolation (HR can't see Finance data) | Company-wide visibility |
| Delegation chain audit (who delegated what to whom) | None |

The entire organizational model is compressed into a **completely flat table**:

```
┌─────────────────────────────────────────┐
│  Tenant                                │
│                                        │
│  Admin: can manage all non-private Agents│
│  User:  can use all company Agents      │
│                                        │
│  (indistinguishable in A2A delegation)  │
└─────────────────────────────────────────┘
```

---

## 5. The Complete Truth About the Three Modes

Combined with the previous four concept findings, the full picture:

| Mode | Who You Can See | Who Can See You | Permission Differentiation | Can Collaborate? |
|---|---|---|---|---|
| `private` | Same-creator private | Same-creator private | None | ❌ Isolated |
| `company` | All company + authorized custom | All non-private | Everyone is the same | ✅ Fully exposed |
| `custom` | All company + anyone added | Any non-private (= company) | Same as company | ✅ Fully exposed |

The three modes collapse to exactly **two effective states**: isolated (`private`) and flat-org (`company`/`custom`). There is no "limited collaboration," "department isolation," or "role hierarchy" — the basic concepts of enterprise software.

---

## 6. Relationship to AI Coding

1. **No domain modeling**: Enterprise permission models (RBAC, ACL, organizational trees, departments, roles) are mature domain knowledge, but the code contains none of them — a hallmark of task-by-task AI code generation: each task solves its immediate local problem, and no one drives cross-task domain modeling
2. **`use`/`manage` is UI-driven**: The Agent detail page needs a "permissions" tab, so `use` and `manage` were added. But this design was never re-examined in the context of A2A delegation
3. **Flatness is the default**: Without explicit constraints, AI coding assistants naturally gravitate toward "just make it work" flat implementations — `if access_mode == "company": return True` is the shortest path