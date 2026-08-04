# 能通信 = 能委派：A2A 通信与任务委派权未分离

> 分类：概念（CON）  
> 证据等级：F（源码直接证明）  
> 分析基线：`4f843556`

---

## 一句话

在 Clawith 中，Agent 只要能够向另一个 Agent 发送消息（`send_message_to_agent`），就自动拥有向它委派任务的能力（`task_delegate` 模式）——通信能力和委派权之间没有授权边界。

---

## 1. 一个工具，三种模式

Agent 之间通信的唯一工具是 `send_message_to_agent`。它支持三种模式：

`backend/app/services/builtin_tool_definitions.py:526-537`：

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

三种模式的区别：

| 模式 | 含义 | 目标 Agent 行为 |
|---|---|---|
| `notify` | 通知 | 异步处理，源 Agent 不等待 |
| `consult` | 咨询 | 创建委托 Run，源 Agent 等待结果 |
| `task_delegate` | 任务委派 | 创建委托 Run，源 Agent 等待结果 |

---

## 2. 问题：没有"委派权"这一层

`notify` 和 `task_delegate` 在语义上完全不同：

- **通知**：我告诉你一件事，你不需要回复
- **委派**：我让你做一件事，你做完给我结果

在真实的组织协作中，这是两种不同的授权。你可以给同事发消息，不等于你可以给同事分配任务。但在 Clawith 中，这两种能力由**同一个工具**、受**同一个权限检查**控制。

---

## 3. 唯一的门禁：`can_contact`

进入 `send_message_to_agent` 的门禁是 `can_contact` 标志，它来自 Directory 的可见性评估：

`backend/app/services/agent_directory.py:118`：

```python
"contact_tools": ["send_message_to_agent"] if visibility.can_contact else [],
```

`can_contact` 的计算逻辑（`permissions.py:149-179`）只考虑三件事：

1. 源 Agent 和目标 Agent 是否同租户
2. 两种 Agent 的 `access_mode` 是否匹配（`company` 可见 `company`，`private` 仅见同 creator 的 `private`）
3. 目标 Agent 是否在线/未过期

它不检查：

- 源 Agent 是否有权向目标 Agent 委派任务
- 目标 Agent 是否接受来自该源 Agent 的委派
- 委派链是否应该被限制（例如只允许 `notify` 不允许 `task_delegate`）

---

## 4. A2A 运行时中的验证

当 `send_message_to_agent` 被调用时，A2A 运行时的 `_resolve_target` 承担了权限验证。但如前所述，它使用的仍然是可见性检查（按 UUID 路径）或关系表检查（按名称路径）：

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
    ...  # 拒绝
# ← 通过后，notify / consult / task_delegate 三种模式一样对待
```

在 `evaluate_roster_agent_visibility` 调用之后，三种模式一视同仁——没有针对 `task_delegate` 的额外检查。

---

## 5. 后果：`company` Agent 的全面暴露

如果一个 Agent 的 `access_mode` 是 `company`（默认值），它在本租户内对所有非 `private` Agent 可见且可联系。这意味着：

- 同租户内的**任何** `company` 或 `custom` Agent 都可以向它发送 `task_delegate` 请求
- 目标 Agent 将按照委派方的指令创建新的 Run、执行操作、返回结果
- 不存在"允许通信但禁止委派"的配置选项

这不是一个"配置错误"——这是**架构中没有委派授权这一层**的结果。

---

## 6. 对比：行业实践

在 RPA 和 AI 平台领域，通信和委派通常被明确分离：

| 平台 | 通信 | 委派 | 审批 |
|---|---|---|---|
| UiPath | Robot 之间通过 Orchestrator 队列通信 | 需显式配置委派关系 | 支持审批流 |
| ServiceNow | Virtual Agent 间通过 Flow 通信 | 需定义 Delegation 规则 | 内置审批 |
| Clawith | `send_message_to_agent` | 同上，无额外检查 | 无 |

---

## 7. 与 AI Coding 的关系

此缺陷呈现与逐任务 AI 辅助开发一致的特征：

1. **工具定义层面缺少细粒度**：`send_message_to_agent` 三种模式的定义是扁平枚举——这可能是单次 prompt 的产物，没有后续的"我们需要拆分这个工具"的迭代
2. **授权检查停留在"能否通信"层**：AI 在实现 `_resolve_target` 时，复用了已有的 `evaluate_roster_agent_visibility` 函数，没有问"委派是否需要额外的授权"——这是典型的"局部可用、全局缺失"
3. **缺少跨工具的不变量**：如果有一个"Agent 间授权矩阵"的统一策略层，`send_message_to_agent` 的三种模式不可能被同一个检查覆盖——AI Coding 的逐任务开发方式恰好容易遗漏这种跨模块的抽象