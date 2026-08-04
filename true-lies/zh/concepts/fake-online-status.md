# 比不做更坏：Dashboard 的虚假在线

> 分类：概念（CON）  
> 证据等级：F（源码直接证明）  
> 分析基线：`4f843556`

---

## 一句话

Dashboard 显示"X 个 Agent 在线"，背后是一整套心跳系统（`heartbeat.py` + `heartbeat_runtime.py`）：每 60 秒循环一次，查 50 条活动记录、10 条未读通知、组装上下文、调用 LLM——但不论 LLM 执行结果如何，`agent.status` 永远是 `running` 或 `idle`。Dashboard 永远显示"在线"。如果直接写死一个静态数字，至少不浪费 LLM 资源。

---

## 1. 系统做了什么

`backend/app/services/heartbeat.py` — 300+ 行：

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
            # 检查过期、时区、活跃时间、间隔...
            instruction, heartbeat_context = await _build_heartbeat_instruction(db, agent)
            runtime_handle = await enqueue_heartbeat_runtime(
                db, agent=agent, occurrence_at=now,
                instruction=instruction, context=heartbeat_context, ...
            )
```

`_build_heartbeat_instruction` 做了什么：

```python
# 查 50 条最近活动
recent_result = await db.execute(
    select(AgentActivityLog)
    .where(AgentActivityLog.agent_id == agent.id)
    .order_by(AgentActivityLog.created_at.desc()).limit(50)
)
# 查 10 条未读通知
notification_result = await db.execute(
    select(Notification)
    .where(Notification.agent_id == agent.id, Notification.is_read.is_(False))
    .limit(10)
)
```

然后 `enqueue_heartbeat_runtime` 触发一次 LLM 调用，执行心跳指令。整个链路：**数据库查询 → 上下文组装 → LLM 调用 → 执行结果**。

## 2. 系统没做什么

**没有更新 `agent.status`**。

搜遍整个 `heartbeat.py`，唯一设置 `status` 的地方是：

```python
# heartbeat.py:234
if agent.expires_at and now >= agent.expires_at:
    agent.is_expired = True
    agent.heartbeat_enabled = False
    agent.status = "stopped"  # 唯一的状态变更
```

——这处理的是"Agent 到期了"，不是"Agent 挂了"。

`status = "error"` 只出现在 API 层的异常处理中（`agents.py:282, 322, 347, 405`），是用户手动操作时报错。**心跳 LLM 调用失败、超时、返回错误——都不影响 `status`。**

## 3. 结果

```
心跳系统：每 60 秒
  ↓
查数据库、组装上下文、调用 LLM（消耗 token）
  ↓
LLM 执行完毕（无论成功/失败/超时/崩溃）
  ↓
agent.status 不变（永远是 running 或 idle）
  ↓
Dashboard 显示：🟢 在线
```

Agent 可能已经崩溃、hang 住、无限循环、LLM 返回垃圾——但 `status` 字段无人更新，Dashboard 永远显示"在线"。

## 4. 为什么比不做更坏

### 如果直接写死

```html
<span>12 个 Agent</span>
```

- 诚实（用户知道这是配置信息）
- 不消耗任何资源
- 没有监控的假象

### 如果做了心跳但不更新状态

```html
<span>🟢 10 个 Agent 在线</span>
```

- 不诚实（用户以为有实时监控）
- 每次心跳消耗 LLM token
- 心跳代码 300+ 行，维护成本
- 查询数据库、组装上下文、审计日志——全是资源开销
- **结果和不做一样**：永远显示"在线"

### 对比

| 方案 | 诚实度 | LLM 消耗 | 结果 |
|---|---|---|---|
| 静态文本 | ✅ | 零 | 没监控 |
| 不做心跳 | 取决于 UI | 零 | 没监控 |
| 做心跳但不更新状态 | ❌ | 每次心跳 | 没监控，但假装有 |

做心跳但不更新状态是**最坏的**：它同时失去了"不做"的零成本和"真做"的可靠性，只保留了"看起来在监控"的幻觉，并且为此持续消耗 LLM 资源。

---

## 5. 与 AI Coding 的关系

1. **功能完整性 ≠ 系统正确性**：心跳系统"功能完整"——有定时器、有上下文构建、有 LLM 调用、有审计日志。但缺少一个关键步骤（把 LLM 执行结果反馈到 `status`），导致整个系统徒劳
2. **逐任务开发的断裂**：一个任务写心跳调度，一个任务写 Dashboard 前端，一个任务定义 `status` 字段。三个任务各自完成，但"心跳结果影响 status"这个跨任务的需求没有出现在任何任务中
3. **"做了很多但没有效果"模式**：AI Coding 擅长生成"做了很多"的代码——心跳、上下文、LLM 调用——但"有效果"需要跨模块的闭环，这恰恰是 AI Coding 最薄弱的