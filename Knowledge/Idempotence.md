---
title: Idempotence
tags: [knowledge, cs, api, http, design]
---

# ♻️ Idempotence

An operation is **idempotent** if applying it **multiple times produces the same result** as applying it once.

```
f(f(x)) = f(x)
```

The word comes from Latin: *idem* (same) + *potens* (power). It's a concept from mathematics adopted heavily in software design.

---

## 📐 Math Origin

In math, an idempotent function is one where repeated application doesn't change the result after the first call:

```
abs(abs(-5))  = abs(5) = 5       ✅ idempotent
sort(sort([3,1,2])) = [1,2,3]   ✅ idempotent
x + 1 applied twice: 5 → 6 → 7  ❌ not idempotent
```

---

## 🌐 Idempotence in HTTP

HTTP defines which methods are expected to be idempotent:

| Method | Idempotent? | Safe? | Notes |
|--------|------------|-------|-------|
| `GET` | ✅ | ✅ | Only reads — no side effects |
| `HEAD` | ✅ | ✅ | Same as GET but no body |
| `PUT` | ✅ | ❌ | Replace a resource — same result if repeated |
| `DELETE` | ✅ | ❌ | Deleting an already-deleted resource = still gone |
| `POST` | ❌ | ❌ | Creates new resource — repeated calls = duplicates |
| `PATCH` | ❌* | ❌ | Depends on implementation |

> **Safe** means "no server state changes." **Idempotent** means "repeated calls don't change state beyond the first."

### Why PUT is idempotent but POST is not

```http
PUT /users/42  { "name": "Ana" }   → always sets user 42's name to "Ana"
POST /users    { "name": "Ana" }   → creates a new user each time
```

---

## 🗃️ Idempotence in Databases

### Idempotent SQL patterns

```sql
-- ❌ Not idempotent: runs twice = balance goes negative twice
UPDATE accounts SET balance = balance - 100 WHERE id = 1;

-- ✅ Idempotent: running twice gives same result
UPDATE accounts SET balance = 0 WHERE id = 1;

-- ✅ INSERT OR IGNORE / UPSERT — safe to run multiple times
INSERT OR IGNORE INTO users (id, email) VALUES (1, 'a@b.com');
```

### Migrations
Database migrations must be idempotent if they can be re-run:
```sql
CREATE TABLE IF NOT EXISTS users (...);  -- ✅ safe to repeat
ALTER TABLE users ADD COLUMN IF NOT EXISTS bio TEXT;  -- ✅
```

---

## 🔁 Idempotence in Distributed Systems

In distributed systems, **network failures** mean a request might be sent more than once (retry logic). If the operation isn't idempotent, you get:
- Duplicate payments
- Double-created records
- Corrupted state

### Idempotency keys
A common pattern: the client generates a unique key per intent, and the server deduplicates by it.

```http
POST /payments
Idempotency-Key: uuid-abc-123
{ "amount": 50.00, "to": "user_99" }
```

If the server already processed `uuid-abc-123`, it returns the original response without charging again. Stripe, PayPal, and most payment APIs use this pattern.

---

## ⚙️ Idempotence in Infrastructure / DevOps

Tools like **Terraform**, **Ansible**, and **Chef** are built around idempotence:

> "Describe the desired state, and the tool will make it so — regardless of current state."

Run `terraform apply` 10 times → same infrastructure. It's safe to re-run because the tool checks what already exists before making changes.

---

## 🧪 Why It Matters for Reliability

| Scenario | Without idempotence | With idempotence |
|----------|--------------------|--------------------|
| Network retry | Duplicate charge | Safe — same result |
| Re-deployed migration | Error or duplicate | No-op |
| Queued job runs twice | Double email sent | Single action |
| Crash mid-operation | Partial state | Recoverable |

---

## 🔗 Relacionado
- [[Knowledge/Abstraction|Abstraction]] — clean contracts between components
- [[Backend/Laravel/Queues|Queues]] — job queues need idempotent handlers for retries
- [[CS/DATABASE|Databases]] — SQL patterns for safe re-runs

## ↩️ Navegación
- [[Knowledge/Knowledge|🧠 Knowledge]] → [[HOME|🏠 HOME]]
