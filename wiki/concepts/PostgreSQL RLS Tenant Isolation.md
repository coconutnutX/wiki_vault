---
created: 2026-05-22
tags:
  - postgresql
  - rls
  - tenant-isolation
  - og-memory
  - security
---

# PostgreSQL RLS Tenant Isolation

## Concept

Row Level Security (RLS) 是 PostgreSQL 的多租户数据隔离机制。oG-Memory 使用强制 RLS 保护 `context_nodes` 表，确保不同租户的数据完全隔离。

## How It Works

### Table Configuration

```sql
-- Check RLS status
SELECT relname, relrowsecurity, relforcerowsecurity 
FROM pg_class WHERE relname = 'context_nodes';

-- Result: relrowsecurity=t, relforcerowsecurity=t
-- "forced row security enabled" means even table owner must pass RLS
```

### Policy Definition

```sql
POLICY "tenant_isolation"
  USING (((account_id)::text = current_setting('app.account_id'::text, true)))
  WITH CHECK (((account_id)::text = current_setting('app.account_id'::text, true)))
```

- **USING**: 读取时过滤，只返回匹配 tenant 的行
- **WITH CHECK**: 写入时验证，只允许写入匹配 tenant 的行

### Tenant Binding

应用代码必须在每个事务开始时设置 tenant context：

```python
# fs/sql_adapter/pool.py
@staticmethod
def bind_tenant(conn, account_id: str) -> None:
    """Bind tenant identity to the current transaction for RLS."""
    with conn.cursor() as cur:
        cur.execute("SET LOCAL app.account_id = %s", (account_id,))
```

使用 `SET LOCAL` 确保绑定在事务结束时自动重置，防止连接池泄露 tenant identity。

## Practical Implications

### Direct psql Queries

直接用 `psql` 查询看不到任何数据（RLS 过滤掉所有行）：

```bash
# ❌ 看不到数据
PGPASSWORD=ogmem123 psql -h 127.0.0.1 -U ogmem -d ogmemory \
  -c "SELECT count(*) FROM context_nodes;"
# Result: 0

# ✅ 必须设置 tenant context
PGPASSWORD=ogmem123 psql -h 127.0.0.1 -U ogmem -d ogmemory \
  -c "SET app.account_id = 'acct-demo'; SELECT count(*) FROM context_nodes;"
# Result: 159
```

### Application Queries

应用代码通过 `bind_tenant()` 自动设置 context，无需额外处理：

```python
conn = pool.get_connection()
pool.bind_tenant(conn, ctx.account_id)  # 设置 RLS context
# ... 所有后续查询自动被 RLS 保护
```

## Tables with RLS

| Table | RLS Enabled | Policy |
|-------|-------------|--------|
| `context_nodes` | ✓ (forced) | `tenant_isolation` |
| `outbox_events` | ✓ | tenant filtering |
| `vector_index` | ✓ | tenant filtering |

## Security Benefits

1. **防御性安全**: 即使应用代码遗漏 tenant 检查，RLS 在数据库层面强制隔离
2. **连接池安全**: `SET LOCAL` 自动重置，防止 tenant identity 泄露
3. **合规性**: 满足多租户数据隔离的安全要求

## Related

- [[oG-Memory SQL-only Headless Start]]
- oG-Memory storage layer architecture