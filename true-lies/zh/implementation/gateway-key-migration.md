# Gateway API Key：新旧逻辑并存

> 分类：实现（IMP）  
> 证据等级：F（源码直接证明）  
> 分析基线：`4f843556`

---

## 一句话

Gateway API Key 的创建使用 SHA256 哈希，验证却优先明文比对——代码处于迁移中间态，存储和验证逻辑不一致，且无迁移完成计划。

---

## 1. 创建路径：哈希存储

`backend/app/api/agents.py:550,1242`：

```python
raw_key = f"oc-{secrets.token_urlsafe(32)}"
agent.api_key_hash = hashlib.sha256(raw_key.encode()).hexdigest()
```

新 key 以 SHA256 哈希存入 `api_key_hash` 列。这是安全的做法——数据库泄露时无法直接恢复 key。

---

## 2. 验证路径：明文优先

`backend/app/api/gateway.py:45-48`：

```python
# First try plaintext (new behavior)
result = await db.execute(
    select(Agent).where(
        Agent.api_key_hash == api_key,    # ← 明文直接比对
```

验证先尝试明文匹配，失败后才回退到哈希。注释称明文为"new behavior"。

---

## 3. 逻辑矛盾

| 路径 | 行为 | 状态 |
|---|---|---|
| 创建（agents.py） | SHA256 哈希 | 旧行为 |
| 验证（gateway.py） | 明文优先，哈希回退 | 声称"新行为" |

创建和验证不一致。如果将来创建改为明文，已存储的哈希 key 将无法验证（除非保留回退）。如果永远不改创建，明文优先的验证路径就是死代码——但占据主要执行路径。

---

## 4. 迁移中间态的风险

- 两条路径同时存在，增加攻击面
- 注释暗示"明文 = 新行为"，引导未来的开发者完成迁移（即删除哈希路径）
- 如果迁移完成，所有 key 将明文存储——数据库泄露即全泄露

---

## 5. 与 AI Coding 的关系

这是典型的"迁移任务被拆分执行"模式：

1. 第一个 prompt："把 API key 验证改为明文比对" → 完成（gateway.py）
2. 第二个 prompt："把 API key 创建改为明文存储" → 未执行或未完成
3. 无人负责"检查迁移是否完成并清理旧路径"

代码中的注释（"new behavior" / "legacy behavior"）记录了迁移意图，但没有人跟踪迁移是否完成。

---

## 6. 关联

详见 [Gateway API Key 的明文存储与非恒定时间比对](../security/gateway-key-plaintext.md)。