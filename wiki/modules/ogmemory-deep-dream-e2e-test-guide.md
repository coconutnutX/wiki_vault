---
type: module
title: "oG-Memory DeepDream E2E Test Guide"
status: active
created: 2026-06-04
updated: 2026-06-04
tags: [ogmemory, deepdream, testing, e2e, locomo]
related:
  - "[[oG-Memory Operations Guide]]"
  - "[[ogmemory-deep-dream-framework]]"
  - "[[oG-Memory Provenance Design RFC]]"
---

# oG-Memory DeepDream 端到端测试指南

从清空数据库到注入对话、调用 DeepDream、验证产出的完整流程。
适用于 LoCoMo small 数据集的快速验证。

---

## 1. 前置条件

| 依赖 | 状态检查 |
|------|----------|
| PostgreSQL | `systemctl status postgresql` |
| oG-Memory 服务 | `curl -s http://127.0.0.1:8090/api/v1/health` |
| Python conda 环境 | `conda activate py11` |

**健康检查期望输出**：
```json
{"agfs":false,"backend":"og-memory","llm":"gpt-4o-mini","sql":"connected","status":"ok","storage_backend":"sql"}
```

## 2. 启动 / 重启服务

每次改了代码后需要重启：

```bash
pkill -f "python server/app.py"
sleep 3
cd /data/Workspace2/oG-Memory
nohup python server/app.py > /tmp/ogmem_stdout.log 2>&1 &
sleep 10
curl -s http://127.0.0.1:8090/api/v1/health
```

> **注意**：日志重定向到 `/tmp/ogmem_stdout.log`。原始 `.ogmem_data/logs/context_engine.log` 不再使用。

## 3. 清空数据库

```bash
PGPASSWORD=ogmem123 psql -h 127.0.0.1 -U ogmem -d ogmemory -c "
TRUNCATE TABLE session_archives, context_nodes, vector_index, outbox_events, dream_recalls, relation_edges CASCADE;
"
```

验证清空：
```bash
PGPASSWORD=ogmem123 psql -h 127.0.0.1 -U ogmem -d ogmemory -c "
SELECT 'session_archives', COUNT(*) FROM session_archives;
SELECT 'context_nodes', COUNT(*) FROM context_nodes;
SELECT 'vector_index', COUNT(*) FROM vector_index;
"
```

期望输出：所有表 COUNT = 0。

## 4. 注入 LoCoMo Small 对话数据

LoCoMo small 包含 4 个 session（Caroline 和 Melanie 的对话）：

```bash
cd /data/Workspace2/oG-Memory
python << 'SCRIPT'
import json, requests, time

with open('/data/Workspace2/locomo_test/data/locomo_small.json') as f:
    data = json.load(f)

conv = data[0]['conversation']
speaker_a = conv['speaker_a']  # Caroline
speaker_b = conv['speaker_b']  # Melanie

API = "http://127.0.0.1:8090/api/v1"
account_id = "acct-demo"
user_id = "ogmem-small"

sessions = ['session_1', 'session_2', 'session_3', 'session_4']

for s_key in sessions:
    session_id = f"locomo-small-{s_key}"
    msgs = conv[s_key]

    # Convert: Caroline = user, Melanie = assistant
    formatted = []
    for m in msgs:
        role = "user" if m['speaker'] == speaker_a else "assistant"
        formatted.append({"role": role, "content": m['text']})

    # 1. Ingest batch
    resp = requests.post(f"{API}/ingest_batch", json={
        "sessionId": session_id,
        "messages": formatted,
        "userId": user_id,
        "accountId": account_id,
    })
    print(f"[ingest_batch] {s_key}: status={resp.status_code}")

    # 2. After_turn to trigger extraction
    resp = requests.post(f"{API}/after_turn", json={
        "sessionId": session_id,
        "messages": formatted,
        "userId": user_id,
        "accountId": account_id,
        "prePromptMessageCount": 0,
    })
    print(f"[after_turn] {s_key}: status={resp.status_code}")

    time.sleep(3)

print("\nAll sessions ingested.")
SCRIPT
```

期望输出：4 个 session 各 `status=200`，`ingested=True`，`ok=True`。

## 5. 等待后台抽取完成

```bash
sleep 120
PGPASSWORD=ogmem123 psql -h 127.0.0.1 -U ogmem -d ogmemory -c "
SET app.account_id = 'acct-demo';
SELECT category, COUNT(*) FROM context_nodes WHERE category NOT IN ('session_archive') GROUP BY category ORDER BY category;
"
```

期望：至少有 event、preference、profile 等类别，总记忆数 50+ 条。

如果数量不够，继续等待：
```bash
sleep 60
# 再次查询
```

## 6. 调用 DeepDream API

```bash
truncate -s 0 /tmp/ogmem_stdout.log   # 清空日志便于追踪

curl -s -X POST http://127.0.0.1:8090/api/v1/deepdream \
  -H "Content-Type: application/json" \
  -d '{"userId":"ogmem-small","accountId":"acct-demo","sessionId":"locomo-small-dream-test"}'
```

等待完成（约 2-5 分钟）：
```bash
sleep 180
```

## 7. 查看日志追踪

```bash
python -c "
with open('/tmp/ogmem_stdout.log', 'rb') as f:
    data = f.read()
text = data.decode('utf-8', errors='replace')
lines = [l.strip() for l in text.split('\n') if l.strip() and 'httpx' not in l and 'werkzeug' not in l]

for l in lines:
    if any(k in l for k in ['deepdream', 'DeepDream', 'DreamOutput', 'provenance',
        'acquire', 'process_promotion', 'completed', 'iteration',
        'ERROR', '_on_invalid', 'candidate', 'write_memory', 'created_uris']):
        print(l[:300])
"
```

关键日志行解读：

| 日志内容 | 含义 |
|----------|------|
| `acquire_recent: user=... uris=20` | 成功获取 20 条记忆 |
| `DreamOutput #1: topic=... provenance_ids=N` | 产出第 N 条 promotion |
| `assigned provenance_ids: visited_uris=X prov_ids=X` | provenance 已分配 |
| `write_memory result: target_uri=... action=create` | 成功写入数据库 |
| `deepdream: completed — status=... dream_outputs=N` | 最终结果 |

## 8. 查看 Dream 产出内容

### 8.1 查看 Dream 列表

```sql
SET app.account_id = 'acct-demo';

SELECT uri, category, abstract
FROM context_nodes
WHERE category = 'dream'
ORDER BY created_at;
```

> **重要**：必须先 `SET app.account_id` 否则 RLS 会返回 0 行。

### 8.2 查看 Dream 详细内容

```sql
SET app.account_id = 'acct-demo';

SELECT uri, abstract, content, metadata
FROM context_nodes
WHERE category = 'dream'
ORDER BY created_at;
```

`metadata` 字段（JSON）中包含：
- `provenance_ids`: 溯源 ID 数组
- `routing_key`: 路由键
- `confidence`: 信心值

### 8.3 解析 provenance_ids 为人类可读 URI

```bash
cd /data/Workspace2/oG-Memory
python -c "
import json
from core.provenance_resolver import ProvenanceResolver

# 从数据库获取 metadata
# 替换为实际的 metadata JSON
metadata = {/* 从 8.2 查询结果复制 */}

for pid in metadata.get('provenance_ids', []):
    parsed = ProvenanceResolver.parse_id(pid)
    human = ProvenanceResolver.display_id(pid)
    print(f'  Prov ID: {pid}')
    print(f'  Human:   {human}')
    print(f'  Type: {parsed[\"source_type\"]}, Source: {parsed[\"source_id\"]}')
    print()
"
```

### 8.4 查看 Dream 向量索引

```sql
SET app.account_id = 'acct-demo';

SELECT uri, level
FROM vector_index
WHERE uri LIKE '%/dream/%'
ORDER BY uri;
```

- `level=0`: 记忆节点向量
- `level=2`: content.md 子节点向量

### 8.5 查看所有记忆类别统计

```sql
SET app.account_id = 'acct-demo';

SELECT category, COUNT(*) FROM context_nodes
GROUP BY category ORDER BY category;
```

### 8.6 验证 Dream 记忆可以被搜索到

```bash
curl -s -X POST http://127.0.0.1:8090/api/v1/compose \
  -H "Content-Type: application/json" \
  -d '{"userId":"ogmem-small","accountId":"acct-demo","sessionId":"test-search","query":"What patterns does this user show?"}'
```

compose 结果中应该包含 dream 记忆。

## 9. 仅删除 Dream 数据（保留其他记忆）

如果只想重跑 DeepDream 但保留已有的抽取记忆：

```bash
PGPASSWORD=ogmem123 psql -h 127.0.0.1 -U ogmem -d ogmemory -c "
SET app.account_id = 'acct-demo';
DELETE FROM context_nodes WHERE category = 'dream';
"
PGPASSWORD=ogmem123 psql -h 127.0.0.1 -U ogmem -d ogmemory -c "
SET app.account_id = 'acct-demo';
DELETE FROM vector_index WHERE uri LIKE '%/dream/%';
"
```

然后从步骤 6 重新开始。

## 10. 一键脚本（完整测试流程）

```bash
#!/bin/bash
# deepdream-e2e-test.sh — 完整端到端测试
set -e

API="http://127.0.0.1:8090/api/v1"
DB="PGPASSWORD=ogmem123 psql -h 127.0.0.1 -U ogmem -d ogmemory"

echo "=== 1. 检查服务 ==="
curl -s $API/v1/health | python -m json.tool

echo "=== 2. 清空数据库 ==="
$DB -c "TRUNCATE TABLE session_archives, context_nodes, vector_index, outbox_events, dream_recalls, relation_edges CASCADE;"

echo "=== 3. 注入对话 ==="
cd /data/Workspace2/oG-Memory
python -c "
import json, requests, time
with open('/data/Workspace2/locomo_test/data/locomo_small.json') as f: data = json.load(f)
conv = data[0]['conversation']
speaker_a, speaker_b = conv['speaker_a'], conv['speaker_b']
API = 'http://127.0.0.1:8090/api/v1'
for s_key in ['session_1','session_2','session_3','session_4']:
    sid = f'locomo-small-{s_key}'
    formatted = [{'role': 'user' if m['speaker']==speaker_a else 'assistant', 'content': m['text']} for m in conv[s_key]]
    requests.post(f'{API}/ingest_batch', json={'sessionId':sid,'messages':formatted,'userId':'ogmem-small','accountId':'acct-demo'})
    requests.post(f'{API}/after_turn', json={'sessionId':sid,'messages':formatted,'userId':'ogmem-small','accountId':'acct-demo','prePromptMessageCount':0})
    time.sleep(3)
print('Ingest done')
"

echo "=== 4. 等待抽取 (120s) ==="
sleep 120
$DB -c "SET app.account_id = 'acct-demo'; SELECT category, COUNT(*) FROM context_nodes WHERE category NOT IN ('session_archive') GROUP BY category ORDER BY category;"

echo "=== 5. 调用 DeepDream ==="
truncate -s 0 /tmp/ogmem_stdout.log
curl -s -X POST $API/v1/deepdream \
  -H "Content-Type: application/json" \
  -d '{"userId":"ogmem-small","accountId":"acct-demo","sessionId":"locomo-small-dream-test"}'

echo ""
echo "=== 6. 等待 DeepDream 完成 (180s) ==="
sleep 180

echo "=== 7. 查看 Dream 结果 ==="
$DB -c "SET app.account_id = 'acct-demo'; SELECT uri, abstract FROM context_nodes WHERE category = 'dream';"
$DB -c "SET app.account_id = 'acct-demo'; SELECT uri, content, metadata FROM context_nodes WHERE category = 'dream';"
$DB -c "SET app.account_id = 'acct-demo'; SELECT category, COUNT(*) FROM context_nodes GROUP BY category ORDER BY category;"

echo "=== 完成 ==="
```

---

## 常见问题

### Q: psql 查询返回 0 行

忘记设 `SET app.account_id = 'acct-demo';`。RLS 会过滤掉所有数据。

### Q: acquire_recent 只返回 1 个 URI

SQL storage 中 flat scan 找到叶子节点（如 profile）后不走 known_categories fallback。已在代码中修复（v2+），fallback 在结果不足 limit 时自动补充。

### Q: LLM 超时 / 429 rate limit

`chatapi.littlewheat.com` 代理网关不稳定，会导致 LLM 调用超时或 rate limit。这不是代码问题，等待几分钟后重试即可。

### Q: content.md 子节点无法 read

content.md 是 vector_index 的 chunk 节点，不是 context_nodes 的可读条目。已在代码中过滤，不会出现在 acquire_recent/acquire_search 的返回结果中。