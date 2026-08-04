# `custom` 模式形同虚设：可见性控制的幻觉

> 分类：概念（CON）  
> 证据等级：F（源码直接证明）  
> 分析基线：`4f843556`

---

## 一句话

Clawith 提供了三种 `access_mode`（`private` / `company` / `custom`），但 `custom` 模式在架构上**完全无法阻止其他 Agent 看到自己**——它只是一个"我能看到谁"的配置，不是"谁可以看我"的控制。`custom` 的实际效果等同于 `company`。

---

## 1. 三种模式的承诺

`backend/app/models/agent.py` 中 Agent 的 `access_mode` 字段有三个值：

| 模式 | 产品语义 | 用户期望 |
|---|---|---|
| `private` | 私有 | 只有我能看到和使用 |
| `company` | 公司可见 | 全公司都能看到 |
| `custom` | 自定义 | 我可以精细控制谁可见 |

`custom` 模式的存在暗示用户可以**选择性地**让某些 Agent 可见、某些不可见。

---

## 2. `custom` 的实际工作机制

`custom` 的可见性由 `AgentAgentRelationship` 表控制。`_agent_directory_conditions`（`agent_directory.py:213-216`）：

```python
def _custom_agent_authorized_condition(source_agent_id: uuid.UUID):
    return exists().where(
        AgentAgentRelationship.agent_id == source_agent_id,    # ← 谁在看
        AgentAgentRelationship.target_agent_id == AgentModel.id,  # ← 在看谁
    )
```

这张表的结构是 `(agent_id, target_agent_id)`——**查看方的 ID 在前，被查看方的 ID 在后**。它的语义是"我能看到谁"，而不是"谁可以看我"。

---

## 3. 关系可以被任意创建

`AgentAgentRelationship` 有两个创建入口，都不需要被查看方同意：

### 入口 1：Custom Directory API

`backend/app/api/directory.py:321-356`：

```python
@router.post("/custom/agents")
async def add_custom_directory_agent(agent_id, payload, current_user, db):
    target = (await db.execute(
        select(Agent).where(
            Agent.id == payload.target_agent_id,
            Agent.access_mode != "private",   # ← 唯一限制
            ...
        )
    )).scalar_one_or_none()
    db.add(AgentAgentRelationship(
        agent_id=agent_id,                    # ← 查看方
        target_agent_id=payload.target_agent_id,  # ← 被查看方
    ))
```

### 入口 2：Legacy Relationships API

`backend/app/api/relationships.py:495-527`：

```python
@router.put("/agents")
async def save_agent_relationships(agent_id, data, current_user, db):
    for r in data.relationships:
        target_result = await db.execute(
            build_visible_agents_query(current_user, ...)  # ← 对用户可见
        )
        db.add(AgentAgentRelationship(
            agent_id=agent_id,              # ← 查看方
            target_agent_id=target_id,      # ← 被查看方
        ))
```

两个入口的共同点：**只检查查看方是谁、被查看方是否存在，从不检查被查看方是否同意。**

---

## 4. 后果：`custom` = `company`

任何非 `private` 的 Agent，都可以被任何管理员添加到任何其他非 `private` Agent 的 Directory 中：

```
Agent A (custom)
    │
    │  A 的 Directory 配置里只有 B
    │  A 认为自己只对 B 可见
    │
    │  管理员调用：
    │  POST /agents/D/directory/custom/agents { target_agent_id: A }
    │  POST /agents/E/directory/custom/agents { target_agent_id: A }
    │  POST /agents/F/directory/custom/agents { target_agent_id: A }
    │
    ▼
D 能看见 A ✅
E 能看见 A ✅
F 能看见 A ✅
```

A 的 `custom` 配置完全无效。架构上不存在"阻止被查看"的机制。

---

## 5. 三种模式的实际效果

| 宣称 | 实际 |
|---|---|
| `private` | ✅ 同 creator 的 private Agent 互见，无法被外部添加 |
| `company` | ✅ 全公司可见 |
| `custom` | ❌ **等于 `company`**——任何管理员可以从任何非 private Agent 的视角添加你 |

唯一有效的保护是设为 `private`。`custom` 是一个**虚假的中间选项**——它给了用户"精细控制"的 UI 假象，但架构上完全无法兑现。

---

## 6. 根因：可见性是查看方属性，不是被查看方属性

在正确的权限模型中，"谁可以看见我"应该由被查看方控制：

```
正确模型： A 的 visibility_policy = [B, C]  →  B 和 C 可以看见 A
Clawith：   D 的 directory = [A, B, C]     →  D 可以看见 A、B、C
```

`AgentAgentRelationship` 表是**查看方的配置**，不是**被查看方的策略**。这是概念模型层面的根本性错误——`custom` 模式从设计上就无法实现"控制谁可以看见我"。

---

## 7. 与 AI Coding 的关系

此缺陷呈现与逐任务 AI 辅助开发一致的特征：

1. **局部概念自洽、全局语义缺失**：`AgentAgentRelationship` 在"我能看到谁"的语境下是合理的——你需要一个列表来管理你的 Directory。问题在于，没有人问"被查看方是否需要同意？"
2. **UI 驱动开发**：`custom` 模式的 UI 给了用户一个"添加 Agent"的按钮，后端忠实地实现了这个按钮的功能。但跨 Agent 的全局不变量——"每个 Agent 的可见性由它自己控制"——从来没有被建模
3. **`private` 的例外恰好证明了规则的缺失**：`private` 之所以有效，是因为它在两个地方被硬编码拒绝（`access_mode != "private"`），而不是因为架构上有一个统一的"可见性策略层"