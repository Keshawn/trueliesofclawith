# The `custom` Mode Illusion: Visibility Control That Doesn't Exist

> Category: Concept (CON)  
> Evidence Level: F (directly provable from source)  
> Analysis Baseline: `4f843556`

---

## TL;DR

Clawith offers three `access_mode` values (`private` / `company` / `custom`), but the `custom` mode is architecturally incapable of preventing other Agents from seeing you — it only controls who *you* can see, not who can see *you*. In practice, `custom` is equivalent to `company`.

---

## 1. What the Three Modes Promise

Agent `access_mode` in `backend/app/models/agent.py` has three values:

| Mode | Product Semantics | User Expectation |
|---|---|---|
| `private` | Private | Only I can see and use it |
| `company` | Company-visible | Everyone in the company can see it |
| `custom` | Custom | I can fine-tune who can see it |

The existence of `custom` implies that users can **selectively** make their Agent visible to some and invisible to others.

---

## 2. How `custom` Actually Works

`custom` visibility is controlled by the `AgentAgentRelationship` table. `_agent_directory_conditions` (`agent_directory.py:213-216`):

```python
def _custom_agent_authorized_condition(source_agent_id: uuid.UUID):
    return exists().where(
        AgentAgentRelationship.agent_id == source_agent_id,       # ← who is looking
        AgentAgentRelationship.target_agent_id == AgentModel.id,  # ← who is being looked at
    )
```

This table's structure is `(agent_id, target_agent_id)` — the **viewer's ID comes first**, the **viewed Agent's ID comes second**. Its semantics are "who I can see," not "who can see me."

---

## 3. Relationships Can Be Created Arbitrarily

There are two entry points for creating `AgentAgentRelationship` entries. Neither requires the viewed Agent's consent.

### Entry Point 1: Custom Directory API

`backend/app/api/directory.py:321-356`:

```python
@router.post("/custom/agents")
async def add_custom_directory_agent(agent_id, payload, current_user, db):
    target = (await db.execute(
        select(Agent).where(
            Agent.id == payload.target_agent_id,
            Agent.access_mode != "private",   # ← the only restriction
            ...
        )
    )).scalar_one_or_none()
    db.add(AgentAgentRelationship(
        agent_id=agent_id,                    # ← viewer
        target_agent_id=payload.target_agent_id,  # ← viewed
    ))
```

### Entry Point 2: Legacy Relationships API

`backend/app/api/relationships.py:495-527`:

```python
@router.put("/agents")
async def save_agent_relationships(agent_id, data, current_user, db):
    for r in data.relationships:
        target_result = await db.execute(
            build_visible_agents_query(current_user, ...)  # ← visible to the user, not the agent
        )
        db.add(AgentAgentRelationship(
            agent_id=agent_id,              # ← viewer
            target_agent_id=target_id,      # ← viewed
        ))
```

Both entry points share the same characteristic: **they only check who the viewer is and whether the target exists. They never ask whether the target consents.**

---

## 4. Consequence: `custom` = `company`

Any non-`private` Agent can be added to any other non-`private` Agent's Directory by any admin:

```
Agent A (custom)
    │
    │  A's Directory only contains B
    │  A believes it is only visible to B
    │
    │  Admin calls:
    │  POST /agents/D/directory/custom/agents { target_agent_id: A }
    │  POST /agents/E/directory/custom/agents { target_agent_id: A }
    │  POST /agents/F/directory/custom/agents { target_agent_id: A }
    │
    ▼
D can see A ✅
E can see A ✅
F can see A ✅
```

A's `custom` configuration is completely meaningless. The architecture provides no mechanism to prevent being viewed.

---

## 5. The Real Effect of the Three Modes

| Claimed | Actual |
|---|---|
| `private` | ✅ Same-creator private Agents can see each other; external addition is blocked |
| `company` | ✅ Company-wide visibility |
| `custom` | ❌ **Equivalent to `company`** — any admin can add you from any non-private Agent's perspective |

The only effective protection is `private`. `custom` is a **false middle option** — it gives users the UI illusion of fine-grained control that the architecture cannot deliver.

---

## 6. Root Cause: Visibility Is a Viewer Property, Not a Viewed-Entity Property

In a correct authorization model, "who can see me" should be controlled by the entity being viewed:

```
Correct model:  A's visibility_policy = [B, C]  →  B and C can see A
Clawith:        D's directory = [A, B, C]       →  D can see A, B, C
```

The `AgentAgentRelationship` table is **the viewer's configuration**, not **the viewed entity's policy**. This is a fundamental conceptual error — `custom` mode is architecturally incapable of implementing "control who can see me."

---

## 7. Relationship to AI Coding

This defect exhibits patterns consistent with task-by-task AI-assisted development:

1. **Locally coherent, globally missing semantics**: `AgentAgentRelationship` is reasonable in the "who I can see" context — you need a list to manage your Directory. The problem is that no one asked: "does the viewed Agent need to consent?"
2. **UI-driven development**: The `custom` mode UI offers an "add Agent" button, and the backend faithfully implements it. But the cross-Agent invariant — "each Agent's visibility is controlled by itself" — was never modeled
3. **The `private` exception proves the rule's absence**: `private` works because it's hardcoded in two places (`access_mode != "private"`), not because there's a unified visibility policy layer in the architecture