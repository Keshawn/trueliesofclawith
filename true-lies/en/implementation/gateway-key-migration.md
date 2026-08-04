# Gateway API Key: Old and New Logic Coexist

> Category: Implementation (IMP)  
> Evidence Level: F (directly proven by source code)  
> Analysis Baseline: `4f843556`

---

## TL;DR

Gateway API Key creation uses SHA256 hashing, but verification tries plaintext first—the code is in a migration intermediate state, with storage and verification logic inconsistent, and no migration completion plan.

---

## 1. Creation Path: Hashed Storage

`backend/app/api/agents.py:550,1242`:

```python
raw_key = f"oc-{secrets.token_urlsafe(32)}"
agent.api_key_hash = hashlib.sha256(raw_key.encode()).hexdigest()
```

New keys are stored as SHA256 hashes in the `api_key_hash` column. This is secure—keys cannot be directly recovered from a database breach.

---

## 2. Verification Path: Plaintext First

`backend/app/api/gateway.py:45-48`:

```python
# First try plaintext (new behavior)
result = await db.execute(
    select(Agent).where(
        Agent.api_key_hash == api_key,    # ← plaintext direct comparison
```

Verification tries plaintext matching first, falling back to hash only on failure. The comment labels plaintext as "new behavior."

---

## 3. Logical Contradiction

| Path | Behavior | Status |
|---|---|---|
| Creation (agents.py) | SHA256 hash | Old behavior |
| Verification (gateway.py) | Plaintext first, hash fallback | Claimed "new behavior" |

Creation and verification are inconsistent. If creation is ever changed to plaintext, existing hashed keys will become unverifiable (unless the fallback is kept). If creation is never changed, the plaintext-first verification path is dead code—but it occupies the primary execution path.

---

## 4. Risks of the Migration Intermediate State

- Two paths coexist, increasing the attack surface
- Comments suggest "plaintext = new behavior," guiding future developers to complete the migration (i.e., delete the hash path)
- If migration is completed, all keys will be stored in plaintext—a database breach means total compromise

---

## 5. Relationship to AI Coding

This is the classic "migration task split across prompts" pattern:

1. First prompt: "Change API key verification to plaintext comparison" → Done (gateway.py)
2. Second prompt: "Change API key creation to plaintext storage" → Not executed or not completed
3. No one is responsible for "checking whether the migration is complete and cleaning up the old path"

The comments ("new behavior" / "legacy behavior") document the migration intent, but no one tracks whether the migration is complete.

---

## 6. Related Findings

See [Gateway API Key: Plaintext Storage & Non-Constant-Time Comparison](../security/gateway-key-plaintext.md).