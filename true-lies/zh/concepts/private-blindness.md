# 既瞎又聋的私有 Agent：`private` 模式的隔离悖论

> 分类：概念（CON）  
> 证据等级：F（源码直接证明）  
> 分析基线：`4f843556`

---

## 一句话

`private` 模式的 Agent 只能看到同 creator 的其他 `private` Agent——它对公司的 `company` 和 `custom` Agent 完全不可见。结果是：用户必须在"安全但无用"和"有用但全裸"之间二选一。

---

## 1. 产品承诺：多 Agent 协作

Clawith 的 README 宣称 Agent 之间可以通过 A2A 机制互相通信、委派任务、协同工作。理想场景：

> 你的"私人秘书"自动将请假请求委派给公司的"HR 助手"，然后处理回复，把结果总结给你。

这要求 `private` Agent 至少能**看到** `company` Agent。

---

## 2. 代码现实：`private` 完全致盲

`backend/app/core/permissions.py:149-179`：

```python
def evaluate_roster_agent_visibility(source_agent, target_agent, ...):
    source_mode = _agent_access_mode(source_agent)
    target_mode = _agent_access_mode(target_agent)
    visible = False

    if source_mode == "private":
        visible = (
            target_mode == "private"
            and getattr(source_agent, "creator_id", None) == getattr(target_agent, "creator_id", None)
        )
    else:
        visible = target_mode == "company" or (target_mode == "custom" and authorized_custom_target)
```

当源 Agent 是 `private` 时：

```python
visible = (target_mode == "private" and source.creator_id == target.creator_id)
```

它**只能**看到同一个人创建的、同样是 `private` 的其他 Agent。`company` 和 `custom` Agent 被彻底排除在外。

---

## 3. 后果矩阵

| 源 Agent | 目标 Agent | 可见？ |
|---|---|---|
| `private` | `private`（同 creator） | ✅ |
| `private` | `private`（不同 creator） | ❌ |
| `private` | `company` | ❌ |
| `private` | `custom` | ❌ |
| `company` | `private` | ❌ |
| `company` | `company` | ✅ |

两个世界完全隔离——`private` 和 `company` 之间**没有任何交集**。

---

## 4. 为什么这样设计：连接即信任

在 Clawith 的 A2A 模型中，一旦 Agent A 能看到 Agent B，A 就能向 B 委派任务（参见 CON-005）。系统没有在"可见"和"可委派"之间设立额外的授权层。

如果 `private` Agent 能看到 `company` Agent，反向也成立的话——`company` Agent 就能看到 `private` Agent，进而向它委派任务、获取其私有数据。这将是跨用户的 Confused Deputy 攻击。

开发者没有实现细粒度的 Agent 间授权，而是直接**切断了 `private` 和 `company` 之间的所有连接**。

---

## 5. 悖论：安全 vs 有用

这个设计制造了一个无法调和的矛盾：

```
private Agent → 安全（别人看不到你）→ 但无法与任何 company Agent 协作 → 废物
company Agent → 能与所有 company Agent 协作 → 但全公司都能看到你 → 裸奔
```

用户被迫在"安全但无用"和"有用但全裸"之间二选一。不存在"安全且有用"的中间地带。

---

## 6. 与 `custom` 模式的关系

结合 CON-*（custom 模式形同虚设），三个模式的实际效果：

| 模式 | 能看见谁 | 谁能看见你 | 能协作吗 |
|---|---|---|---|
| `private` | 只有同 creator 的 private | 只有同 creator 的 private | ❌ 无法与 company 协作 |
| `company` | 所有 company + 已授权的 custom | 所有非 private | ✅ 但全裸 |
| `custom` | 所有 company + 已添加的 custom | 任何非 private（等于 company） | ✅ 但全裸 |

唯一能保护隐私的 `private` 模式，恰恰是唯一无法参与多 Agent 协作的模式。

---

## 7. 与 AI Coding 的关系

此缺陷呈现与逐任务 AI 辅助开发一致的特征：

1. **安全修复的局部最优**：发现"连接即信任"漏洞后，最直接的修复是"切断连接"——这是单次 prompt 能想到的最快方案。但它没有考虑这个修复对产品功能的影响
2. **缺少跨关注点的权衡**：安全（不能越权）和功能（多 Agent 协作）被当作两个独立的任务处理，没有人在架构层面追问"两者能否共存？"
3. **`private` 的语义退化**：`private` 本应意味着"我的数据只有我能访问"，但代码把它变成了"我的 Agent 与世隔绝"——这是概念模型在实现中被简化的典型案例