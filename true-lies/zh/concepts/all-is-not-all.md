# 全部 ≠ 全部：系统性的"全量"语义不一致

> 分类：概念（CON）  
> 证据等级：F（源码直接证明）  
> 分析基线：`4f843556`

---

## 一句话

Clawith 中多处声称"全部"的功能，实际上都不包含全部。这不是孤立的 bug，而是一个贯穿整个系统的模式：**"全部"这个词在 UI 和后端之间的语义从未被对齐过。**

---

## 1. 例一：Activity Feed 的"全部活动"

Dashboard 的 "Recent Activity" 面板展示 Agent 的活动记录，数据来自 `AgentActivityLog` 表。

`backend/app/models/activity_log.py` 中 `action_type` 的枚举值：

```python
action_type: Enum(
    "chat_reply", "tool_call", "feishu_msg_sent", "agent_msg_sent",
    "web_msg_sent", "task_created", "task_updated", "file_written",
    "error", "schedule_run", "heartbeat", "plaza_post",
)
```

**缺失**：`a2a_delegated`、`consult`、`notify`、`task_delegated`。

任何通过 A2A 委派完成的工作——Agent A 委托 Agent B 执行任务，Agent B 完成并返回结果——**不会出现在活动日志中**。Dashboard 上的"全部活动"实际上排除了所有 Agent 间协作。

---

## 2. 例二：Directory 的"全部 Agent"

`query_directory` 工具返回"当前租户下的 Agent 列表"，但 `_agent_directory_conditions` 按 `access_mode` 过滤：

- `private` 源 Agent 只能看到同 creator 的 `private` Agent
- `company` 源 Agent 看不到 `private` Agent

"全部"实际意味着"你的 access_mode 允许你看到的那个子集"。

---

## 3. 例三：`custom` 的"自定义可见性"

`custom` 模式承诺用户可以自定义"谁可以看见我的 Agent"。但实际上：

- `AgentAgentRelationship` 是**查看方配置**（`agent_id` = 谁在看），不是被查看方策略
- 任何管理员可以从任何非 private Agent 的视角添加你

"自定义"实际意味着"任何人都可以自定义看到你"。

---

## 4. 这三个例子不是孤立的

| 功能 | 声称 | 实际 |
|---|---|---|
| Activity Feed | 全部活动 | 缺 A2A 委派 |
| Directory | 全部 Agent | 缺跨 access_mode 的 Agent |
| `custom` 模式 | 自定义可见性 | 查看方自定义，被查看方失控 |

这三个例子来自不同的模块（Activity Log、Directory、Permissions），但呈现相同的模式：

```
UI/产品层面：承诺"全部"
    ↓
代码层面：按局部上下文实现了"当前条件下可见的子集"
    ↓
结果："全部"的语义从未被全局定义和强制执行
```

---

## 5. 根因：没有一个全局的"全部"定义

在正确的系统中，"全部"需要被明确定义：

```
全部活动 = 所有 action_type 的活动
全部 Agent = 所有租户内的 Agent（按可见性过滤是另一层）
全部可见性 = 被查看方控制的可见性策略
```

但 Clawith 没有这个全局定义。每个模块各自理解"全部"的含义，前端按自己的理解展示，后端按自己的理解过滤。**"全部"这个词的语义散落在每个模块的局部实现中，从未被对齐。**

---

## 6. 与 AI Coding 的关系

1. **局部语义自洽，全局概念漂移**：每个模块的"全部"在其局部上下文中有自己的定义（Activity Log 的"全部" = 枚举中列出的类型；Directory 的"全部" = 当前 access_mode 可见的 Agent）。单独看每个模块都"正确"，但跨模块时"全部"的语义不再一致
2. **没有全局概念注册表**：AI Coding 按任务生成代码时，不会有人追问"等一下，'全部'在这个系统里到底是什么意思？"——每个任务各自定义"全部"的范围
3. **UI 文本与后端逻辑从未交叉验证**：前端写"全部活动"，后端写 `select * from agent_activity_logs`——两个局部都正确，但没有人检查"后端返回的真的是全部吗？"