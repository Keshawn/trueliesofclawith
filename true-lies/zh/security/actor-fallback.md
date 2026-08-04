# actor 缺失时回退 Agent 创建者身份

> 分类：安全（SEC）  
> 证据等级：F（源码直接证明）  
> 分析基线：`4f843556`

---

## 一句话

工具执行时，若 `context.actor_user_id` 为 None（匿名/未认证/委托执行），系统静默回退到 `agent.creator_id`（通常是管理员），形成混淆代理人漏洞。

---

## 1. 回退点一：文件删除

`backend/app/services/agent_runtime/tool_step_service.py:494`：

```python
actor_user_id = context.actor_user_id or str(agent.creator_id)
```

此回退用于 `delete_file` 和 `GROUP_DELETE_WORKSPACE_FILE` 的权限判断。当 `actor_user_id` 为 None 时，以 Agent 创建者（通常是管理员）的身份执行删除。

---

## 2. 回退点二：通用工具执行

`tool_step_service.py:1879`：

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

同模式——`actor_user_id` 为 None 时回退到 `agent.creator_id`。

---

## 3. actor_user_id 可为 None

`backend/app/services/agent_runtime/state.py:214`：

```python
class RuntimeContext:
    actor_user_id: str | None = None
```

`actor_user_id` 的类型就是 `str | None`，系统设计上允许它为 None。

---

## 4. 触发场景

`actor_user_id` 可能为 None 的场景包括：

- **委托执行**（delegated run）：A2A 委托、OpenClaw gateway 消息
- **群聊上下文**：群聊中 Agent-to-Agent `@` 调用，原始用户上下文可能丢失
- **定时任务/触发器**：scheduler 触发的执行无人类用户上下文
- **匿名执行**：通过 API 直接调用 run 的入口

---

## 5. 混淆代理人攻击链

```
普通员工 Jerry 在群聊中 @ 管理员 Tom 创建的 Agent
  → Agent 处理群聊消息，context.actor_user_id 可能为 None
  → 工具执行回退到 agent.creator_id = Tom (管理员)
  → Jerry 以 Tom 的权限执行文件操作/工具调用
```

---

## 6. 与 AI Coding 的关系

这种 `or` 回退是典型的"让代码能跑"开发模式：

- 开发者在某个路径上遇到 `actor_user_id` 为 None 导致失败
- 加上 `or str(agent.creator_id)` 作为 "fallback" 使其工作
- 未意识到这会授予非授权用户管理员级别的权限
- 两个回退点（line 494 和 line 1879）独立实现，暗示同一问题被修补了两次，每次以相同方式