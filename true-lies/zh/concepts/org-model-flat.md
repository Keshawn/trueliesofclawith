# 没有任何权限控制的系统：`company` 模式下的全公司同权

> 分类：概念（CON）  
> 证据等级：F（源码直接证明）  
> 分析基线：`4f843556`

---

## 一句话

`custom` 等于 `company` 不是最可怕的。最可怕的是 `company` 模式下，组织内所有用户的权限**完全相同**——没有部门、没有角色层级、没有能力差异。而表面上唯一的区分（`use` vs `manage`）在 A2A 委派中形同虚设。

---

## 1. `company` 模式：全公司 = 一个人

`backend/app/core/permissions.py:44-58`：

```python
def can_use_agent_static(user: User, agent: Agent) -> bool:
    access_mode = _agent_access_mode(agent)
    if access_mode == "company":
        return True  # ← 任何用户，只要在租户内，就可以使用
```

对 `company` Agent 而言，用户身份只检查一件事：**是不是这个租户的**。没有其他任何区分。

### 这意味着什么？

```
Alice (CEO)         → 可以使用公司 HR Agent
Bob (实习生)         → 可以使用公司 HR Agent
Charlie (外部顾问)    → 只要在租户内，也可以使用公司 HR Agent
```

三个人权限完全相同。没有"实习生只能查看，不能委派"、"顾问只能咨询，不能执行"这样的粒度。

---

## 2. `use` vs `manage`：唯一的区分，且只在管理端有意义

系统唯一的权限区分是 `use` 和 `manage`：

```python
# can_manage_agent (permissions.py:95-126)
def can_manage_agent(db, user, agent):
    if _is_admin(user) and access_mode != "private":
        return True
    # custom 模式下检查 AgentPermission 表
    ...
```

| 角色 | 对 `company` Agent 的权限 |
|---|---|
| 普通用户 | `use`（可以使用） |
| Admin | `manage`（可以配置） |
| Creator | `manage`（可以配置） |

但这个区分只影响**人类用户直接操作 Agent 配置**的能力。在 Agent 之间的 A2A 通信中，它完全不起作用。

---

## 3. 为什么 `use` vs `manage` 在 A2A 中形同虚设

当 Agent A 通过 A2A 委派任务给 Agent B 时，执行流程是：

`backend/app/services/agent_runtime/a2a_runtime.py:772-830`：

```python
async def execute(self, ...):
    owner_user_id = (
        source_run.origin_user_id    # 发起方 run 的原始用户
        or actor_user_id             # 或 actor
        or source_agent.creator_id   # 或源 Agent 的创建者
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

目标 Agent B 执行任务时，`origin_user_id` 被设为源 Agent 的创建者。这意味着：

1. Bob（有 `use` 权限）创建 Agent B
2. Bob 通过 Agent B 向公司的 HR Agent A（创建者 Alice）委派任务
3. HR Agent A 执行任务时，`origin_user_id` = Bob
4. 任务以 Alice 的 Agent 的身份执行，拥有 Alice 的 Agent 的全部能力

**`use` 权限的 Bob 通过委派，让 Alice 的 Agent 执行了只有 `manage` 权限才能做的事。** `use` vs `manage` 的区分在 A2A 路径上被完全绕过。

---

## 4. 缺失的企业组织结构

对比真实企业软件应有的权限模型：

| 真实企业 | Clawith |
|---|---|
| 部门（HR、财务、工程…） | 无 |
| 角色层级（Admin > Manager > User > Viewer） | 只有 Admin / User 两层 |
| 按角色授予能力（可查看 / 可使用 / 可委派 / 可管理） | 只有 `use` / `manage` |
| 跨部门隔离（HR 看不到财务数据） | 全公司可见 |
| 委派链审计（谁委派了什么给谁） | 无 |

整个组织模型被压缩成了一张**完全扁平的表**：

```
┌─────────────────────────────────────────┐
│  Tenant                                │
│                                        │
│  Admin: 可以 manage 所有非 private Agent │
│  User:  可以 use 所有 company Agent     │
│                                        │
│  (两者在 A2A 委派中无区别)               │
└─────────────────────────────────────────┘
```

---

## 5. 三个模式的完整真相

结合前四篇概念发现，三个 `access_mode` 的完整图景：

| 模式 | 能看见谁 | 谁能看见你 | 权限差异 | 能协作吗 |
|---|---|---|---|---|
| `private` | 同 creator 的 private | 同 creator 的 private | 无 | ❌ 与世隔绝 |
| `company` | 所有 company + 已授权的 custom | 所有非 private | 全公司一样 | ✅ 全裸 |
| `custom` | 所有 company + 被添加的任何人 | 任何非 private（= company） | 等于 company | ✅ 全裸 |

三种模式实际上只有两个有效状态：**与世隔绝**（`private`）和**全公司一样**（`company` / `custom`）。不存在"有限协作"、"部门隔离"、"角色分级"这些企业软件的基本概念。

---

## 6. 与 AI Coding 的关系

1. **没有领域建模**：企业权限模型（RBAC、ACL、组织树、部门、角色）是成熟的领域知识，但代码中完全没有体现——这是 AI 逐任务生成代码的典型特征：每个任务只解决眼前的局部问题，没有人推动跨任务的领域建模
2. **`use`/`manage` 是 UI 驱动的产物**：Agent 详情页需要一个"权限"标签，所以有了 `use` 和 `manage` 两个值。但这个设计从来没有被放到 A2A 委派的上下文中重新审视
3. **扁平化是默认行为**：AI 编码助手在没有明确约束的情况下，天然倾向于"能跑就行"的扁平实现——`if access_mode == "company": return True` 是最短路径