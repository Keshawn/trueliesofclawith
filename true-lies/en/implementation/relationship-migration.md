# Relationship Migration: An Unfinished Half-Product

> Category: Implementation (IMP)  
> Evidence Level: F (directly proven by source code)  
> Analysis Baseline: `4f843556`

---

## TL;DR

Agent relationship management was migrated from `relationships.py` (9 REST APIs) to `directory.py` (hardcoded `collaborator` + empty description), but the old API and ~700 lines of frontend component remain fully intact in the codebase—unused and uncleaned.

---

## 1. The Relationship Model Retains Full Fields

`backend/app/models/org.py:82-98`:

```python
class AgentAgentRelationship(Base):
    relation: Mapped[str] = mapped_column(String(50), nullable=False,
                                           default="collaborator")
    description: Mapped[str] = mapped_column(Text, default="")
    created_by_user_id: Mapped[uuid.UUID | None]
    updated_by_user_id: Mapped[uuid.UUID | None]
```

The fields support rich relationships like `peer`, `supervisor`, `assistant`—but the new path forces the simplest possible values.

---

## 2. The New Path: Hardcoded Degradation

`backend/app/api/directory.py:350-353`:

```python
db.add(AgentAgentRelationship(
    agent_id=agent_id,
    target_agent_id=payload.target_agent_id,
    relation="collaborator",    # ← hardcoded
    description="",             # ← forced empty
))
```

The rich organizational relationship model is degraded to a boolean ACL switch: "related" or "not related."

---

## 3. The Old API Still Runs in Full

All 9 endpoints in `backend/app/api/relationships.py` remain active:

| Line | Method | Path |
|---|---|---|
| 128 | GET | `/` |
| 185 | GET | `/member-candidates` |
| 295 | PUT | `/` |
| 377 | DELETE | `/{rel_id}` |
| 403 | GET | `/agent-candidates` |
| 444 | GET | `/agents` |
| 479 | GET | `/agents/candidates` |
| 494 | PUT | `/agents` |
| 539 | DELETE | `/agents/{rel_id}` |

All still accept requests and operate on the database, but the frontend no longer calls them. These are **orphan APIs**: running, but known to no one and maintained by no one.

---

## 4. Frontend Dead Code

`frontend/src/pages/agent-detail/AgentDetailPage.tsx:1507` defines `function RelationshipEditor()`, approximately 700 lines, containing complete React state, search debouncing, relationship dropdown menus, and API integration code.

This function is **never referenced or mounted** anywhere in the file. It is dead code.

---

## 5. Relationship to AI Coding

1. **"Adding a feature = creating a new file/function"**: Replacing `relationships.py` with `directory.py` instead of refactoring is typical AI task-by-task development behavior
2. **"Deleting old code" was never a task**: The AI won't proactively say "we should now clean up the old code"—that requires an architect's holistic judgment
3. **Hardcoded degradation**: `relation="collaborator"` + `description=""` is the simplest possible implementation—AI tends to make minimal changes to complete a task, rather than preserving design integrity