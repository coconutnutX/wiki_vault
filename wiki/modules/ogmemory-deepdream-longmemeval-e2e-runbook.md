---
name: ogmemory-deepdream-longmemeval-e2e-runbook
description: DeepDream LongMemEval 端到端测试操作手册（环境、注入、before QA、dream、after QA）
metadata:
  type: reference
---

# DeepDream LongMemEval 端到端操作手册

> ⚠️ **过时提示**：本文档中提到的 `search_recall` 表、`RecallLogger`、`acquire_search_recall_stats` 已被替代。当前代码使用 `signal_record` 表（通过 `SignalRecorder` 类写入）和 `query_signal_stats` 方法。5 张废弃表（`search_recall`、`dream_runs`、`dream_signals`、`dream_staged_candidates`、`dream_signal_query_seen`）已从数据库中删除，无需创建或 TRUNCATE。最新流程规范见 [[longmemeval-test-standardization]]。

从环境准备到注入数据、before-dream QA、调用 DeepDream、after-dream QA 的完整操作流程。

**核心设计**：先跑 before-dream QA（compose 积累 search_recall 统计），再调 DeepDream（recall_stats 有数据可用），最后跑 after-dream QA。这样 dream 的 `acquire_search_recall_stats` 能看到 compose 真实使用产生的召回频率，产出更有价值的 consolidation。

## 1. 环境准备

### 目录

| 项目 | 路径 |
|------|------|
| oG-Memory 代码 | `/data/Workspace2/oG-Memory` |
| longmemeval 测试框架 | `/data/Workspace2/repos/longmemeval` |
| 数据文件 | `/data/Workspace2/repos/longmemeval/data/` |
| 配置文件 | `/data/Workspace2/repos/longmemeval/configs/` |

### conda 环境

```bash
conda activate py11
```

所有脚本（runner、deepdream、QA eval、服务）都必须在 py11 环境下运行。

### 配置文件层级

longmemeval 框架的配置合并顺序：默认值 → `configs/env.toml` → 实验配置（如 `configs/dream-36-ingest.toml`）。

`configs/env.toml` 包含全局共享参数（api_url、account_id 等），实验配置覆盖特定参数（data_file、user_prefix 等）。

**当前 `env.toml` 关键参数**：

```toml
[ogmem]
api_url = "http://127.0.0.1:8090"
docker_container = ""          # 空 = 直接用 host 端口，不走 docker
account_id = "longmemeval"
```

## 2. 启动 / 确认 oGMemory 服务

### 确认服务状态

```bash
curl -s http://127.0.0.1:8090/api/v1/health
```

期望输出：

```json
{"agfs":false,"backend":"og-memory","llm":"...","sql":"connected","status":"ok","storage_backend":"sql"}
```

### 启动服务

**关键：必须从 oG-Memory 目录启动**，否则 `dream/prompts/dream.yaml` 和 `dream/tool_specs/` 按相对路径找不到，导致 dream 的 prompt-driven tool 不加载（`prompt_driven_specs=[]`），LLM 只会用 operational tools 而不产出 dream。

```bash
cd /data/Workspace2/oG-Memory
conda activate py11
nohup python -u server/app.py > /tmp/ogmem_stdout.log 2>&1 &
disown
```

验证 cwd 正确：

```bash
ls -la /proc/<PID>/cwd
# 应指向 /data/Workspace2/oG-Memory
```

### 日志

| 日志 | 路径 | 说明 |
|------|------|------|
| 服务 stdout/stderr | `/tmp/ogmem_stdout.log` | 需要 `python -u` 重定向到此文件才能实时看到 |
| context_engine.log | `.ogmem_data/logs/context_engine.log` | 旧日志，不再实时写入（仅 history） |

> **注意**：如果服务是后台启动且没有重定向到文件（stdout 是 pipe），日志不会持久化。务必用 `nohup python -u ... > /tmp/ogmem_stdout.log 2>&1 &` 启动。

### 重启服务（改代码后）

```bash
pkill -f "python -u server/app.py"
sleep 3
cd /data/Workspace2/oG-Memory
nohup python -u server/app.py > /tmp/ogmem_stdout.log 2>&1 &
disown
sleep 10
curl -s http://127.0.0.1:8090/api/v1/health
```

## 3. account_id 与数据库隔离

### account_id 作用

oGMemory 使用 PostgreSQL Row-Level Security (RLS) 按 `account_id` 隔离数据。所有操作（注入、compose、dream、psql 查询）必须使用同一个 `account_id`。

**当前约定**：`account_id = "longmemeval"`

### psql 查询前必须 SET

```sql
SET app.account_id = 'longmemeval';
```

不设则 RLS 返回 0 行。这是最常见的坑。

### 是否需要清空数据库

| 场景 | 操作 |
|------|------|
| 首次测试 / 完全干净重跑 | 清空所有表 |
| 只重跑 dream，保留抽取记忆 | 只删 dream 行 + search_recall 行 |
| 做 before-dream QA（无 dream） | 只删 dream 行 + 向量索引 |
| 不清空，直接追加 | 不需要任何操作 |

**清空所有表**（完全重跑）：

```sql
TRUNCATE TABLE session_archives, context_nodes, vector_index, outbox_events, dream_recalls, relation_edges CASCADE;
```

**只删除 dream 数据**（保留抽取记忆）：

```sql
SET app.account_id = 'longmemeval';
DELETE FROM context_nodes WHERE category = 'dream';
DELETE FROM vector_index WHERE uri LIKE '%/dream/%';
```

**删除 dream + search_recall**（重跑 before-dream QA 和 dream）：

```sql
SET app.account_id = 'longmemeval';
DELETE FROM context_nodes WHERE category = 'dream';
DELETE FROM vector_index WHERE uri LIKE '%/dream/%';
DELETE FROM search_recall;
DELETE FROM dream_recalls;
```

验证清空：

```sql
SET app.account_id = 'longmemeval';
SELECT 'context_nodes', COUNT(*) FROM context_nodes;
SELECT 'vector_index', COUNT(*) FROM vector_index;
SELECT 'search_recall', COUNT(*) FROM search_recall;
```

### user_id 语义

LongMemEval 每条题目 = 一个独立虚拟用户。runner 为每题分配独立 `user_id`：`{user_prefix}-{run_id}-{qid}`。

**按题目隔离是正确的评测语义**——题目间不应互相干扰。compose 召回时只应看到该虚拟用户的记忆。

## 4. 数据集

### 已提交可用

| 文件                                          |    大小 |  题目 | 唯一 session | 题型               | 用途                    |
| ------------------------------------------- | ----: | --: | ---------: | ---------------- | --------------------- |
| `longmemeval_s_dream_openclaw_smoke_9.json` | 4.7MB |   9 |        440 | ms, tr, ku       | 最小冒烟                  |
| `longmemeval_s_dream_18.json`               | 8.9MB |  18 |        855 | ms, tr, ku       | 低成本主测                 |
| `longmemeval_s_dream_36.json`               |  18MB |  36 |       1638 | ms, tr, ku       | 正式主测 ← **当前使用**       |
| `longmemeval_s_dream_smoke_30.json`         |  16MB |  30 |       1442 | ms, tr, ku       | 中规模冒烟                 |
| `longmemeval_s_dream_abstention.json`       |  13MB |  24 |       1135 | ms, tr, ku       | 拒答护栏                  |
| `longmemeval_s_dream_all_abstention.json`   |  16MB |  30 |       1426 | ms, tr, ku, ss-u | 全部拒答                  |
| `longmemeval_oracle.json`                   |  15MB | 500 |        940 | 全 6 种            | 官方 oracle（非 dream 专用） |

题型缩写：ms=multi-session, tr=temporal-reasoning, ku=knowledge-update, ss-u=single-session-user

所有 dream 数据集每题 answer_session_ids ≥ 2（跨 session 证据），适合测 dream 的跨 session consolidation/correction 定位。

oracle 系列不适合测 dream——每题只有 2-6 个纯证据 session（无干扰），per-user 记忆太少无法支撑 dream consolidation。

### 未提交（需下载）

| 文件 | 大小 | 获取方式 |
|------|------:|----------|
| `longmemeval_s_dream_core.json` | 164MB | Git LFS（当前本地是指针未下载） |
| `longmemeval_s_cleaned.json` | 277MB | HuggingFace 下载 |

## 5. 注入数据

### 用 longmemeval runner 注入

```bash
cd /data/Workspace2/repos/longmemeval
conda activate py11
python -m longmemeval_test.cli run configs/dream-36-ingest.toml --only health_check,run
```

runner 会为每题分配独立 `user_id`（`dream-36-{run_id}-{qid}`），逐 session 调 `/api/v1/after_turn`，然后调 `compose` 验证召回（reader=none 时只注入不回答）。

### 注入配置关键参数

```toml
[general]
name = "dream-36-ingest"
env_file = "env.toml"
data_file = "data/longmemeval_s_dream_36.json"
modes = "ogmemory-compose"    # 注入 + compose

[reader]
kind = "none"                 # 只注入不回答

[ogmem]
user_prefix = "dream-36"     # ← 后续脚本都依赖此值
post_ingest_wait = 60.0       # 注入后等 60 秒让后台抽取完成

[steps]
health_check = true
run = true                    # 只跑注入
official_eval = false
stats = false
```

### 注入耗时估算

dream_36（36 题 × ~50 session）：约 60-90 分钟。

### 获取 run_id（后续脚本都需要）

注入完成后，从 runner 输出目录读取：

```bash
python3 -c "
import json
d = json.load(open('output/dream-36-ingest/run_config.json'))
print(f'run_id:      {d[\"run_id\"]}')
print(f'user_prefix: {d[\"user_prefix\"]}')
"
```

### 验证注入成功

```sql
SET app.account_id = 'longmemeval';
SELECT category, COUNT(*) FROM context_nodes
WHERE category NOT IN ('session_archive')
GROUP BY category ORDER BY category;
```

期望：entity, event, preference, profile 等类别，总行数 300+（36 题 × 每题 ~50-70 条抽取记忆）。

## 6. Before-dream QA 评测（积累 search_recall）

### 为什么先跑 before-dream QA

compose 的每次向量搜索命中会写入 `search_recall` 表（通过 RecallLogger）。Dream 的 `acquire_search_recall_stats` 从 `search_recall` 表聚合统计，找出被多次召回、跨 query、跨天的记忆。

**如果先跑 dream 再跑 compose**，`search_recall` 表是空的（没经过 compose），`acquire_search_recall_stats` 返回空结果，LLM 只能靠 acquire_recent + read 找 consolidation 机会——这不是设计意图的完整流程。

**先跑 before-dream QA**，让 compose 自然积累 search_recall 统计数据，dream 就能基于真实使用频率做精准的 consolidation 判断。

### 脚本：`scripts/run_qa_eval.py`

对已有的 user_id 调 compose（召回记忆）→ reader（回答问题）→ judge（官方判分），不做注入。同时 RecallLogger 写入 search_recall 统计。

每条 detail 记录包含完整链路：question → compose（召回的记忆正文）→ reader hypothesis → judge 判分。

输出文件：
- `results.json`：聚合 JSON，每题一条，含 question、memory_context、hypothesis、expected、result、compose_stats
- `details.jsonl`：每题一行 JSON（同数据，行分隔）
- `ogmemory-compose.hypotheses.jsonl`：LongMemEval 兼容答案文件

```bash
cd /data/Workspace2/repos/longmemeval
conda activate py11

python scripts/run_qa_eval.py \
  --runner-output output/dream-36-ingest \
  --data-file data/longmemeval_s_dream_36.json \
  --api-url http://127.0.0.1:8090 \
  --account-id longmemeval \
  --reader-model gpt-4o-mini \
  --judge-model gpt-4o-mini \
  --tag before-dream
```

输出目录：`output/before-dream-<run_id>/`

### 验证 search_recall 数据已积累

```sql
SET app.account_id = 'longmemeval';
SELECT COUNT(*) AS total_recall_events FROM search_recall;

-- 查看哪些记忆被高频召回
SELECT uri, COUNT(*) AS recall_count,
  COUNT(DISTINCT query_text) AS unique_queries,
  COUNT(DISTINCT date_trunc('day', recalled_at)) AS recall_days,
  MAX(category) AS category
FROM search_recall
GROUP BY uri
HAVING COUNT(*) >= 3
ORDER BY recall_count DESC
LIMIT 20;
```

### 耗时估算

compose ~5s + reader ~10-30s + judge ~5s = 每题约 20-40s。36 题 ≈ 12-24 分钟。

## 7. 调用 DeepDream

### 脚本：`scripts/run_deepdream.py`

从 runner 的 `run_config.json` 读 run_id 和 user_prefix，按题维度为每题的 user_id 调 `/api/v1/deepdream`。

**此时 search_recall 已有数据**，`acquire_search_recall_stats` 能返回真实召回统计。

```bash
cd /data/Workspace2/repos/longmemeval
truncate -s 0 /tmp/ogmem_stdout.log   # 清空日志便于追踪

python scripts/run_deepdream.py \
  --runner-output output/dream-36-ingest \
  --data-file data/longmemeval_s_dream_36.json \
  --api-url http://127.0.0.1:8090 \
  --account-id longmemeval \
  --pause 30 \
  --timeout 600
```

### 关键参数

| 参数 | 说明 |
|------|------|
| `--runner-output` | runner 输出目录（读 `run_config.json` 获取 run_id） |
| `--data-file` | 同注入用的数据文件（构造与 runner 相同的 qid 列表） |
| `--account-id` | 与注入一致 |
| `--pause` | 题间暂停秒数（30s 避免连续 LLM 调用触发 rate limit） |
| `--timeout` | 单次 dream 调用超时（600s） |
| `--limit` | 只处理前 N 题 |

### 耗时估算

- 单题：2-5 分钟
- 36 题：约 80-150 分钟（1.5-2.5 小时）

### 产出保存

结果 JSON 保存到 `output/<runner-output>/deepdream_results.json`。

### 查看日志

```bash
strings /tmp/ogmem_stdout.log | grep -E "(deepdream|process_|DreamOutput|write_memory|completed|prompt_driven|recall_stats)" | grep -v "httpx|werkzeug|outbox_worker|embeddings"
```

关键日志行：

| 内容 | 含义 |
|------|------|
| `prompt_driven_specs=['process_consolidation', 'process_correction']` | semantic tool 加载成功 |
| `acquire_search_recall_stats` | LLM 用了 recall 统计（如果出现说明 search_recall 有数据） |
| `DreamOutput #1: topic=... sub_type=merge confidence=0.9` | dream 产出详情 |
| `write_memory result: action=create` | 成功写入 DB |

## 8. After-dream QA 评测

Dream 产出的 consolidation/correction 记忆已写入数据库。现在对同样的 user_id 跑 compose + answer + judge，看 dream 是否提升了召回或准确率。

```bash
cd /data/Workspace2/repos/longmemeval
conda activate py11

python scripts/run_qa_eval.py \
  --runner-output output/dream-36-ingest \
  --data-file data/longmemeval_s_dream_36.json \
  --api-url http://127.0.0.1:8090 \
  --account-id longmemeval \
  --reader-model gpt-4o-mini \
  --judge-model gpt-4o-mini \
  --tag after-dream
```

输出目录：`output/after-dream-<run_id>/`

**不需要删除 dream 数据**——after-dream QA 就是要在有 dream 的情况下测试 compose 召回效果。

### QA eval 脚本关键参数

| 参数 | 说明 |
|------|------|
| `--runner-output` | runner 输出目录（读 `run_config.json` 获取 run_id） |
| `--data-file` | 同注入用的数据文件 |
| `--reader-model` | 回答问题的 LLM 模型 |
| `--judge-model` | 官方判分的 LLM 模型（可用同一模型） |
| `--tag` | 输出目录名称标记 |
| `--skip-judge` | 跳过判分步骤 |
| `--pause` | 题间暂停（默认 5s） |
| `--limit` | 只处理前 N 题 |

### reader/judge API 配置

```bash
# 环境变量方式（推荐）
export OPENAI_API_KEY="sk-..."
export OPENAI_BASE_URL="https://chatapi.littlewheat.com/v1"
```

### 查看评测结果

```bash
python3 -c "
import json
run_id = json.load(open('output/dream-36-ingest/run_config.json'))['run_id']
for tag in ['before-dream', 'after-dream']:
    path = f'output/{tag}-{run_id}/results.json'
    d = json.load(open(path))
    print(f'{tag}: accuracy={d[\"accuracy\"]} ({d[\"correct\"]}/{d[\"graded\"]})')
    for qtype, stats in d['by_question_type'].items():
        qacc = stats.get('accuracy','n/a')
        print(f'  {qtype}: {qacc} ({stats[\"correct\"]}/{stats[\"graded\"]})')
    print()
"
```

### 查看 per-question 详情（debug）

```bash
python3 -c "
import json
run_id = json.load(open('output/dream-36-ingest/run_config.json'))['run_id']
for tag in ['before-dream', 'after-dream']:
    path = f'output/{tag}-{run_id}/results.json'
    d = json.load(open(path))
    print(f'=== {tag} ===')
    for r in d['results']:
        cats = r.get('compose_stats', {}).get('hit_categories', [])
        has_dream = 'dream' in cats
        print(f'  {r[\"question_id\"]} {r[\"question_type\"]} result={r.get(\"result\",\"?\")} has_dream={has_dream} chars={r.get(\"retrieved_chars\",0)}')
    print()
"
```

### 评测指标

| 指标 | 说明 |
|------|------|
| `accuracy` | CORRECT / graded（整体准确率） |
| `by_question_type.accuracy` | 分题型准确率 |
| `hit_count` | compose 向量搜索命中数 |
| `hit_categories` | compose 召回了哪些类别（看是否包含 `dream`） |
| `retrieved_chars` | compose 返回证据字符数 |

### 耗时估算

36 题 ≈ 12-24 分钟。

## 9. 查看 Dream 结果（SQL）

### 列表 + 核心字段

```sql
SET app.account_id = 'longmemeval';

SELECT
  uri,
  abstract,
  content,
  metadata->'tool_stats'->>'sub_type'    AS sub_type,
  metadata->'tool_stats'->>'confidence'  AS confidence,
  metadata->>'routing_key'               AS routing_key,
  jsonb_array_length(metadata->'provenance_ids') AS prov_count
FROM context_nodes
WHERE category = 'dream'
ORDER BY created_at;
```

### sub_type 分布统计

```sql
SET app.account_id = 'longmemeval';

SELECT
  metadata->'tool_stats'->>'sub_type' AS sub_type,
  COUNT(*)
FROM context_nodes WHERE category = 'dream'
GROUP BY metadata->'tool_stats'->>'sub_type';
```

### 查看 provenance_ids 溯源内容

```sql
SET app.account_id = 'longmemeval';

SELECT uri, metadata->'provenance_ids' AS provenance_ids
FROM context_nodes WHERE category = 'dream';
```

每条 provenance_id 格式：`prov:1:memory:ctx%3A%2F%2F...`（URL-encoded URI）。

### 查看所有记忆类别统计

```sql
SET app.account_id = 'longmemeval';

SELECT category, COUNT(*) FROM context_nodes
WHERE category NOT IN ('session_archive')
GROUP BY category ORDER BY category;
```

### 验证 dream 能被向量搜索召回

```sql
SET app.account_id = 'longmemeval';

SELECT uri, level FROM vector_index
WHERE uri LIKE '%/dream/%' ORDER BY uri;
```

### 查看 search_recall 统计

```sql
SET app.account_id = 'longmemeval';

SELECT uri, COUNT(*) AS recall_count,
  COUNT(DISTINCT query_text) AS unique_queries,
  COUNT(DISTINCT date_trunc('day', recalled_at)) AS recall_days,
  MAX(category) AS category
FROM search_recall
GROUP BY uri
ORDER BY recall_count DESC
LIMIT 30;
```

## 10. 注意事项与踩坑总结

### ⚠️ 必须用 dev_0608 分支

`dev_0608` 包含完整的 dream enhancement：process_correction + process_consolidation + acquire_search_recall_stats。其他分支（如 dev_0603）缺少关键功能，dream prompt 是旧版（process_promotion），没有 recall_stats tool。

### ⚠️ cwd 必须是 oG-Memory 目录

服务进程的 cwd 必须是 `/data/Workspace2/oG-Memory`。否则 dream 的 prompt-driven tool 不加载。症状：`prompt_driven_specs=[]` 和 `Template not found: dream/prompts/dream.yaml`。

验证：`ls -la /proc/<PID>/cwd`

### ⚠️ psql 查询前必须 SET app.account_id

否则 RLS 返回 0 行。

### ⚠️ 流程顺序：先 before QA → dream → after QA

这是设计意图的正确流程：
1. Before-dream QA：compose 积累 search_recall 统计 + 得到 baseline accuracy
2. Dream：recall_stats 有数据，dream 能精准判断哪些记忆值得 consolidation
3. After-dream QA：dream 产出已写入 DB，compose 会召回 dream 类别记忆

如果反过来（先 dream → QA），search_recall 为空，dream 只能靠 acquire_recent 而没有召回频率信号。

### ⚠️ Before-dream QA 不需要删 dream

因为首次跑 before-dream QA 时数据库里根本没有 dream 数据（还没跑 dream），所以不需要删除操作。

### ⚠️ run_id 同步

注入 runner、dream 脚本、QA eval 脚本的 `run_id` 必须一致。所有脚本从同一个 `--runner-output` 目录的 `run_config.json` 读取。

### ⚠️ source_uris 可能是逗号分隔字符串

LLM 填写 `source_uris`/`affected_uris` 时可能返回逗号分隔字符串而非 JSON 数组。代码已修复（`deep_dream_react_loop.py`），字符串会先 split(",") 再 strip。

### ⚠️ LLM 行为不可控

DeepDream 的 ReAct loop 让 LLM 自主决定是否调用 process_consolidation/correction。某些题目 LLM 可能不产出 dream。有 search_recall 统计后，LLM 更可能发现高频召回的模式并产出 consolidation。

### ⚠️ LLM 超时 / rate limit

代理网关不稳定，dream 调用可能超时。遇到时重试即可。

### ⚠️ OutboxWorker 自动运行

不需要单独启动 index worker。OutboxWorker 在 server/app.py 进程内自动运行。

### ⚠️ compose 召回 dream 不一定提升准确率

dream consolidation 的内容与 LongMemEval 题目需要的答案证据不一定匹配。这是正常的——dream 的目的是跨 session 合并/纠偏，不是专门为某道题生成答案。

## 11. 完整流程速查（从零开始，dream_36）

```bash
# ==============================
# Step 0: 环境 + 分支
# ==============================
conda activate py11
cd /data/Workspace2/oG-Memory && git checkout dev_0608

# ==============================
# Step 1: 清空数据库
# ==============================
PGPASSWORD=ogmem123 psql -h 127.0.0.1 -U ogmem -d ogmemory -c \
  "TRUNCATE TABLE session_archives, context_nodes, vector_index, outbox_events, dream_recalls, relation_edges CASCADE;"

# ==============================
# Step 2: 启动服务（确保 cwd 是 oG-Memory）
# ==============================
cd /data/Workspace2/oG-Memory
nohup python -u server/app.py > /tmp/ogmem_stdout.log 2>&1 &
disown
sleep 10 && curl -s http://127.0.0.1:8090/api/v1/health

# ==============================
# Step 3: 注入数据（36 题）
# ==============================
cd /data/Workspace2/repos/longmemeval
python -m longmemeval_test.cli run configs/dream-36-ingest.toml --only health_check,run

# ==============================
# Step 4: Before-dream QA 评测（积累 search_recall）
# ==============================
python scripts/run_qa_eval.py \
  --runner-output output/dream-36-ingest \
  --data-file data/longmemeval_s_dream_36.json \
  --api-url http://127.0.0.1:8090 \
  --account-id longmemeval \
  --reader-model gpt-4o-mini \
  --judge-model gpt-4o-mini \
  --tag before-dream

# ==============================
# Step 5: 调 DeepDream（recall_stats 有数据）
# ==============================
truncate -s 0 /tmp/ogmem_stdout.log
python scripts/run_deepdream.py \
  --runner-output output/dream-36-ingest \
  --data-file data/longmemeval_s_dream_36.json \
  --api-url http://127.0.0.1:8090 \
  --account-id longmemeval \
  --pause 30 \
  --timeout 600

# ==============================
# Step 6: After-dream QA 评测
# ==============================
python scripts/run_qa_eval.py \
  --runner-output output/dream-36-ingest \
  --data-file data/longmemeval_s_dream_36.json \
  --api-url http://127.0.0.1:8090 \
  --account-id longmemeval \
  --reader-model gpt-4o-mini \
  --judge-model gpt-4o-mini \
  --tag after-dream

# ==============================
# Step 7: 对比结果
# ==============================
python3 -c "
import json
run_id = json.load(open('output/dream-36-ingest/run_config.json'))['run_id']
for tag in ['before-dream', 'after-dream']:
    path = f'output/{tag}-{run_id}/results.json'
    d = json.load(open(path))
    print(f'{tag}: accuracy={d[\"accuracy\"]} ({d[\"correct\"]}/{d[\"graded\"]})')
    for qtype, stats in d['by_question_type'].items():
        print(f'  {qtype}: {stats.get(\"accuracy\",\"n/a\")} ({stats[\"correct\"]}/{stats[\"graded\"]})')
    print()
"
```

## 相关文档

- [[ogmemory-deepdream-longmemeval-test-plan]] — 测试计划（数据集选择、user_id 隔离分析）
- [[ogmemory-deep-dream-e2e-test-guide]] — locomo 数据的端到端测试指南
- [[ogmemory-deep-dream-framework]] — deepdream 框架设计