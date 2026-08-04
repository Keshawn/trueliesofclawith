# relationship 迁移：未收口的半成品

> 分类：实现（IMP）  
> 证据等级：F（源码直接证明）  
> 分析基线：`4f843556`

---

## 一句话

Agent 关系管理从 `relationships.py`（9 个 REST API）迁移到 `directory.py`（硬编码 `collaborator` + 空描述），但旧 API 和 ~700 行前端组件仍完整保留在代码库中，无人调用也无人清理。

---

## 1. 关系模型保留完整字段

`backend/app/models/org.py:82-98`：

```python
class AgentAgentRelationship(Base):
    relation: Mapped[str] = mapped_column(String(50), nullable=False,
                                           default="collaborator")
    description: Mapped[str] = mapped_column(Text, default="")
    created_by_user_id: Mapped[uuid.UUID | None]
    updated_by_user_id: Mapped[uuid.UUID | None]
```

字段支持 `peer`、`supervisor`、`assistant` 等丰富关系——但新路径强制使用最简值。

---

## 2. 新路径：硬编码降级

`backend/app/api/directory.py:350-353`：

```python
db.add(AgentAgentRelationship(
    agent_id=agent_id,
    target_agent_id=payload.target_agent_id,
    relation="collaborator",    # ← 硬编码
    description="",             # ← 写死为空
))
```

丰富的组织关系模型被降维成布尔 ACL 开关："有关联"或"无关联"。

---

## 3. 旧 API 仍完整运行

`backend/app/api/relationships.py` 中 9 个端点全部活跃：

| 行号 | 方法 | 路径 |
|---|---|---|
| 128 | GET | `/` |
| 185 | GET | `/member-candidates` |
| 295 | PUT | `/` |
| 377 | DELETE | `/{rel_id}` |
| 403 | GET | `/agent-candidates` |
| 444 | GET | `/agents` |
| 479 | GET | `/agents/candidates` |
| 494 | PUT | `/agents` |
| 539 | DELETE | `/agents/{rel_id}` |

全部仍可接受请求、操作数据库，但前端不再调用。这些是**孤儿 API**：运行中，但无人知晓、无人维护。

---

## 4. 前端死代码

`frontend/src/pages/agent-detail/AgentDetailPage.tsx:1507` 定义了 `function RelationshipEditor()`，约 700 行，包含完整的 React State、搜索框防抖、关系下拉菜单和接口联调代码。

该函数在文件中**从未被引用或挂载**。它是死代码。

---

## 5. 与 AI Coding 的关系

1. **"添加新功能 = 新建文件/函数"**：用 `directory.py` 替代 `relationships.py` 而非重构，是 AI 逐任务开发的典型行为
2. **"删除旧代码"从未被列入任务**：AI 不会主动说"现在应该清理旧代码"——这需要架构师的整体判断
3. **硬编码降级**：`relation="collaborator"` + `description=""` 是最简单的实现——AI 倾向于最小改动完成任务，而非保持设计完整性