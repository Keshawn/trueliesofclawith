# 数字员工，还是个人助理？

> 分类：概念（CON） + 架构（ARC）  
> 证据等级：F（源码直接证明）  
> 分析基线：`4f843556`

---

## 一句话

README 宣称 "Digital Employees … just like a new hire"，但代码中 Agent 的身份由 `creator_id` 外键派生、RBAC 无 `agent` 角色、配额挂在 User 表上——架构上是个人助理，不是组织同事。

---

## 1. 官方承诺

Clawith 的 README 在开篇即定义了自己的产品定位（[README.md:49-50](https://github.com/Keshawn/clawith/blob/4f843556/README.md#L49)）：

> Clawith agents are **digital employees of your organization**. Every agent understands the full org chart, can send messages, delegate tasks, and build real working relationships — **just like a new hire joining a team.**

即：Agent 是组织中的数字员工，像新同事一样理解组织架构、建立工作关系。

---

## 2. 行业基准：什么是"数字员工"

"数字员工"（Digital Worker）并非 Clawith 独创的概念。在行业实践中，这一概念已有明确的定义边界：

### 2.1 身份独立性

**非人类身份（Non-Human Identity, NHI）** 是现代 IAM 体系的基础概念。Gartner 在身份优先安全框架中明确要求：非人类身份必须作为一等实体管理，拥有独立的身份生命周期，不应随人类创建者的离职而失效[^1]。

Google BeyondCorp / Zero Trust 架构的核心原则：每一个访问主体——无论人类或非人类——都必须拥有自己独立的、可验证的身份[^2]。

在 RPA 和 AI 平台领域，UiPath 的 Robot 和 ServiceNow 的 Digital Worker 均拥有：

- 独立的机器身份（AD/AAD 中的 Robot 账号，不与人类绑定）
- 独立的许可证分配
- 审计日志归属于 Robot 本身，而非其注册者

对比之下，Clawith 的 Agent 通过 `creator_id` 外键派生身份——Agent 的"存在"依赖于创建者。如果创建者被删除，Agent 的归属关系直接断裂。

### 2.2 授权模型

NIST SP 800-162（ABAC 指南）将系统中的实体区分为**主体（Subject）**和**对象（Object）**[^3]：

- **主体**：发起访问请求的主动实体，拥有独立的属性集和安全标识
- **对象**：被访问的被动实体

数字员工在其职责范围内必须是**主体**——它自己发起消息、调用工具、访问资源。但 Clawith 的 RBAC 枚举中没有 `agent` 角色，Agent 在权限模型中是从属于 `agent_admin` 的对象。

OWASP 非人类身份 Top 10（2025 草案）将"非人类身份缺乏独立授权模型"列为 CHER-1（Credential Handling and Entity Resolution）[^4] 的核心风险。

### 2.3 可问责性

ISO/IEC 22989:2022（AI 术语标准）定义 AI 系统的可问责性为："一个实体对其行为及后果可以被问责的状态"[^5]。

如果 Agent 的工作记录、配额消耗、成本归属全部绑定在创建者（人类）身上，则 Agent 没有被问责的架构基础。真正的数字员工应有独立的：

- 成本中心归属
- 绩效指标
- 工作记录（无论创建者是否在职）

---

上述行业实践共同定义了数字员工的**最低架构标准**：独立身份、按岗位授权、独立担责。这三条不是本文的发明，而是行业共识的提炼。

[^1]: Gartner, "Identity-First Security: The New Perimeter," 2023.
[^2]: Google Cloud, "BeyondCorp: A New Approach to Enterprise Security," 2014.
[^3]: NIST SP 800-162, "Guide to Attribute Based Access Control (ABAC) Definition and Considerations," 2014.
[^4]: OWASP, "Non-Human Identity Top 10 (Draft)," 2025.
[^5]: ISO/IEC 22989:2022, "Information technology — Artificial intelligence — Artificial intelligence concepts and terminology."

---

## 3. 三个判据

如果 Agent 真的是"数字员工"而不仅仅是"我的助手"，它至少需要满足三个判据：

| 判据 | 含义 | 行业依据 |
|---|---|---|
| 独立身份 | 不依赖创建者而存在，有自己的组织身份 | Non-Human Identity (Gartner), BeyondCorp (Google) |
| 按岗位授权 | 权限来自岗位/角色，而非创建者是谁 | NIST ABAC Subject/Object 区分, OWASP NHI Top 10 |
| 独立担责/预算 | 有自己的配额、成本中心、工作记录 | ISO/IEC 22989 可问责性定义 |

这三条是"数字化同事"区别于"数字化助手"的边界，每一条都有对应的行业标准作为依据。

---

## 4. 判据一：独立身份 ❌

`backend/app/models/agent.py:43-44`：

```python
creator_id: Mapped[uuid.UUID] = mapped_column(
    UUID(as_uuid=True), ForeignKey("users.id"), nullable=False
)
tenant_id: Mapped[uuid.UUID | None] = mapped_column(
    UUID(as_uuid=True), ForeignKey("tenants.id")
)
```

Agent 的身份是一个带 `creator_id` 非空外键的数据库记录。Agent 没有：

- 独立的密码学身份（密钥对、证书）
- 独立的组织岗位（position/title/department）
- 独立的委托链（谁授权了这个 Agent 代表谁？）

它的身份是"某个人名下的一条记录"。

---

## 5. 判据二：按岗位授权 ❌

`backend/app/models/user.py:75-76`：

```python
Enum("platform_admin", "org_admin", "agent_admin", "member",
     name="user_role_enum"),
```

四个角色是：平台管理员、组织管理员、Agent 管理员、普通成员。全部是**人类角色**。

没有 `agent` 角色。在 Clawith 的权限体系里，Agent 不是一等公民，而是被 `agent_admin` 管理、被 `member` 使用的**对象**。

---

## 6. 判据三：独立担责/预算 ❌

`backend/app/models/user.py:92,96`：

```python
quota_message_limit: Mapped[int] = mapped_column(Integer, default=50)
quota_max_agents: Mapped[int] = mapped_column(Integer, default=2)
```

配额主体是 **User**（消息额度、可创建 Agent 数量）。Agent 层只有运行时限速：

```python
max_llm_calls_per_day: Mapped[int] = mapped_column(Integer, default=1000)
```

Agent 没有独立的预算、编制或成本中心。你能创建几个 Agent、每个 Agent 能发多少消息，取决于**你是谁**——这是"我的助理"的治理逻辑，不是"数字同事"的编制模型。

---

## 7. 能力上界

```
Agent 实际能力 = creator 的权限 ∩ 租户 RBAC ∩ 配额上限
```

Agent 永远不会超过创建它的人的能力。这是个人助理的治理公式。

---

## 8. 对比：三类实体

| 实体 | 层级 | 核心特征 | Clawith 的 Agent |
|---|---|---|---|
| 我的助手 | 个人级 | 身份派生自创建者，权限不超出创建者 | ✅ 匹配 |
| 数字员工 | 组织级 | 独立身份、按岗位授权、独立担责 | ❌ 不匹配 |
| 自主 Agent | 独立级 | 独立目标、自主决策、独立资源 | ❌ 不匹配 |

Clawith 的 Agent 在架构上是**第一类**（个人助理），但 README 用**第二类**（数字员工）的语言包装。

---

## 9. 与 AI Coding 的关系

**这不是 AI Coding 的问题——这是产品定位决策。** 但 AI Coding 的参与体现在：

1. 代码忠实地实现了"个人助理"模型（`creator_id` 外键、User 配额、人类 RBAC），从未偏离——AI 没有"纠正"错误的产品定位，它只是忠实地执行了它
2. README 和代码之间的鸿沟暗示：营销语言和工程实现由不同的人/流程产出，AI 只是其中一方的执行者
3. 这是"AI Coding 的工程后果"研究中的**基准线**：定位谎言是项目起源问题，AI Coding 的贡献是**忠实实现了一个错误的定位**，并在实现过程中放大了架构裂缝