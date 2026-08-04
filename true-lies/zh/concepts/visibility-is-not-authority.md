# 可见 ≠ 可管理：被混淆的"可见"语义

> 分类：概念（CON）  
> 证据等级：F（源码直接证明）  
> 分析基线：`4f843556`

---

## 一句话

在 Clawith 的 Directory 模型中，"能看见一个 Agent"同时承担了"能发现它""能联系它""能向它委派任务"三层语义——但代码里没有区分这三件事。

---

## 1. 问题：一个"可见"承载了太多

Clawith 的 Agent Directory（花名册）是 Agent 之间互相发现和通信的基础设施。一个 Agent 在 Directory 中能"看到"哪些其他 Agent，由 `evaluate_roster_agent_visibility` 决定。

但这个函数返回的不仅是一个"是否可见"的布尔值：

`backend/app/core/permissions.py:18-23`：

```python
@dataclass(frozen=True)
class RosterVisibility:
    """Visibility result for roster-driven agent and human lookup."""

    visible: bool
    can_contact: bool
    unavailable_reason: str | None = None
```

它把两件事打包在一起：**能不能看见**（`visible`）和**能不能联系**（`can_contact`）。在代码逻辑中，`visible` 决定 Agent 是否出现在 Directory 列表中，而 `can_contact` 决定 `send_message_to_agent` 工具是否可用。

`backend/app/services/agent_directory.py:118`：

```python
"contact_tools": ["send_message_to_agent"] if visibility.can_contact else [],
```

---

## 2. 它混淆了什么

在真实的组织协作中，以下四件事通常需要不同的授权决策：

| 概念 | 含义 | 典型授权条件 |
|---|---|---|
| **可发现** | 能在 Directory 中搜到 | 同租户/同部门 |
| **可通信** | 能发消息 | 需要双向关系或审批 |
| **可委派** | 能向它布置任务 | 需要管理权限或委派授权 |
| **可管理** | 能修改它的配置 | 需要管理员角色 |

但在 Clawith 中，这四件事的边界被模糊了：

- `visible` 承担了"可发现"和"可通信"（`can_contact` 由同一个函数计算）
- `can_contact` 承担了"可通信"和"可委派"（`send_message_to_agent` 同时支持 `notify`、`consult`、`task_delegate` 三种模式）

---

## 3. A2A 委派直接使用可见性检查

最关键的证据在 A2A 运行时。当一个 Agent 通过 UUID 调用另一个 Agent 时，`_resolve_target` 使用 `evaluate_roster_agent_visibility` 来验证权限：

`backend/app/services/agent_runtime/a2a_runtime.py:417-431`：

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

这条代码的意思很清楚：**"能在 Directory 里看到 + 能联系 = 能委派任务"**。

没有单独的检查问："源 Agent 是否有权向目标 Agent 委派任务？目标 Agent 是否接受来自该源 Agent 的委派？"

---

## 4. 两种解析路径的不一致

`_resolve_target` 实际上有两条路径，使用了不同的授权逻辑：

| 路径 | 触发条件 | 授权检查 | 位置 |
|---|---|---|---|
| 按 UUID 解析 | 调用方传了 `target_agent_id` | `evaluate_roster_agent_visibility`（可见性检查） | `a2a_runtime.py:417-431` |
| 按名称解析 | 调用方传了 `agent_name` | `AgentAgentRelationship` 表 + `evaluate_agent_relationship_status` | `a2a_runtime.py:442-462` |

同一个委派操作，两种入口，两种授权逻辑。这本身也是"实现不一致"的一个例子。

---

## 5. "可见"的语义矩阵

`evaluate_roster_agent_visibility` 的可见性规则（`permissions.py:149-179`）：

```python
def evaluate_roster_agent_visibility(
    source_agent: Agent,
    target_agent: Agent,
    *,
    authorized_custom_target: bool = False,
) -> RosterVisibility:
```

| 源 Agent 模式 | 目标 Agent 模式 | 可见？ | 可联系？ |
|---|---|---|---|
| `private` | `private` 且同 creator | ✅ | 取决于状态 |
| `private` | `company` | ❌ | — |
| `company` | `company` | ✅ | 取决于状态 |
| `company` | `custom` 且已授权 | ✅ | 取决于状态 |
| `custom` | `company` | ✅ | 取决于状态 |
| `custom` | `custom` 且已授权 | ✅ | 取决于状态 |

这里的关键是：`company` 模式的 Agent 可以看到所有 `company` 和已授权的 `custom` Agent。这意味着**一个"公司可见"的 Agent 可以被同租户内的任何其他 Agent 联系和委派任务**——只要目标 Agent 在运行。

---

## 6. 系统性后果

当"可见"同时承担"可发现""可通信""可委派"三层语义时：

1. **无法实施细粒度授权**：你无法让某个 Agent "能被看到但不能被委派"——因为授权检查中没有这个维度
2. **`company` 模式 Agent 暴露面过大**：任何同租户 Agent 都可以向它委派任务，这不是权限模型的设计选择，而是**缺少权限模型**的结果
3. **新增授权维度需要改动多处**：因为 `RosterVisibility` 的语义已嵌入 Directory、A2A、工具定义等多个模块

---

## 7. 与 AI Coding 的关系

此缺陷呈现与逐任务 AI 辅助开发一致的特征：

1. **局部概念自洽**：`RosterVisibility` 在 Directory 上下文中是合理的——你需要知道"能看到哪些 Agent 以及它们是否在线"。问题出在这个概念被 **不加修改地复用** 到 A2A 委派授权中
2. **跨模块语义漂移**：Directory 模块定义的"可见可联系"被 A2A 模块当作"可委派"使用——两个模块由不同 prompt/任务开发时，这种语义借用很容易发生
3. **缺少全局授权层**：如果有一个统一的"Agent 间授权"策略层，`RosterVisibility` 不可能被误用为委派检查——AI Coding 的逐任务开发方式恰恰容易遗漏这种跨模块的抽象层