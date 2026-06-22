---
type: module
title: "LongMemEval Dream_36 重测进度 (0622)"
created: 2026-06-22
updated: 2026-06-22
tags: [longmemeval, deepdream, test, in-progress, schema-migration, doubao-quota]
related:
  - longmemeval-dream36-test-report
  - ogmemory-deepdream-longmemeval-e2e-runbook
---

# LongMemEval Dream_36 重测进度 (2026-06-22)

> ⚠️ **过时提示**：本文档中提到的 `search_recall` 表已废弃。当前代码使用 `signal_record` 表替代。5 张废弃表已从数据库中删除。下次重测请按 [[longmemeval-test-standardization]] 的新数据库方案执行，不再需要 TRUNCATE 或清空 `search_recall`。

用户要求：全部用 doubao 提取和 dream（之前用 gpt4o 提取），清空数据库从头重跑。但因 doubao Volcengine API 配额耗尽（7月3日重置），中途调整方案。

关联：[[longmemeval-dream36-test-report]]（旧测试报告）、[[ogmemory-deepdream-longmemeval-e2e-runbook]]（操作手册）

## 1. 最终模型分配方案

| 角色 | 模型 | API | Key | 状态 |
|------|------|-----|-----|------|
| **提取**（注入阶段） | doubao-seed-2-0-code-preview-260215 | `https://ark.cn-beijing.volces.com/api/coding/v3` | `beb569ea-db9e-47a3-8d40-ff3fcbabd9cf` | ✅ 已完成注入 |
| **compose / dream**（服务端） | gpt-4o-mini | `https://chatapi.littlewheat.com/v1` | `sk-sxWGh4hWeExbe8sqZEkgBi4E9l8E53oaAaoYEzjxbzR5IOgk` | ✅ 服务已切换 |
| **reader / judge**（QA评测） | GLM-5 | `http://113.46.219.251:8080/v1` | `sk--hAux09V3aXYBMsaZG_k2w` | ✅ API可用 |
| **embedding** | text-embedding-ada-002 | `https://chatapi.littlewheat.com/v1` | `sk-sxWGh4hWeExbe8sqZEkgBi4E9l8E53oaAaoYEzjxbzR5IOgk` | ✅ |

**注意**：Volcengine doubao API 配额已耗尽，报 `AccountQuotaExceeded`，重置时间 **2026-07-03 23:59:59 +0800 CST**。

## 2. 已完成步骤

### Step 0: 清空数据库 ✅

```sql
TRUNCATE TABLE session_archives, context_nodes, vector_index, outbox_events, dream_recalls,
  search_recall, relation_edges, dream_signals, dream_staged_candidates, dream_runs CASCADE;
```

### Step 1: 注入数据 ✅

| 参数 | 值 |
|------|-----|
| run_id | `00080bf4` |
| user_prefix | `dream-36` |
| user_id 格式 | `dream-36-00080bf4-{qid}` |
| account_id | `longmemeval` |
| agent_id | `openclaw-longmemeval` |
| 提取模型 | doubao-seed-2-0-code-preview-260215 |
| 数据文件 | `data/longmemeval_s_dream_36.json` |
| runner 配置 | `configs/dream-36-ingest.toml` |
| runner 输出目录 | `output/dream-36-ingest/` |

注入完成后的数据库统计：

| 表 | 记录数 |
|----|--------|
| context_nodes | 13,481 |
| vector_index | 23,662 |
| search_recall | 629（来自失败的 before-dream QA 尝试） |

context_nodes 按类别分布：

| category | count |
|----------|-------|
| case | 70 |
| entity | 1,725 |
| event | 4,212 |
| pattern | 102 |
| preference | 2,911 |
| profile | 22 |
| session_summary | 1,006 |
| skill | 16 |
| state | 1,641 |
| tool | 135 |

**对比上次（gpt4o 提取）**：上次 22,875 context_nodes / 32,935 vector_index，本次 doubao 提取量约少 41%。doubao-seed 的提取风格更精简。

### Step 1a: 服务端 LLM 切换 ✅

注入完成后因 doubao 配额耗尽，把服务端 LLM 从 doubao 切为 gpt-4o-mini：

```yaml
# config/ogmem.yaml 变更
llm:
  api_key: "sk-sxWGh4hWeExbe8sqZEkgBi4E9l8E53oaAaoYEzjxbzR5IOgk"  # 从 beb569ea 改
  base_url: "https://chatapi.littlewheat.com/v1"                   # 从 volcengine 改
  model: "gpt-4o-mini"                                              # 从 doubao-seed 改
```

服务重启确认：
- cwd: `/data/Workspace2/oG-Memory` ✅
- health: `{"llm":"gpt-4o-mini","status":"ok"}` ✅

## 3. 阻塞问题：Schema Migration

### Before-dream QA 尝试失败 ❌

before-dream QA 跑了 34 题，但全部失败：
- 29 题 HTTP 500（compose 报 RLS 错误）
- 5 题 Connection refused（服务重启期间）

### 根因分析

compose 500 错误有两个子问题：

**问题 A：`node_uri_aliases` 表不存在**

- 服务端新代码引用 `node_uri_aliases` 表（URI alias / rename 功能）
- 该表定义在 `fs/sql_adapter/db_schema.sql` 中
- 但数据库里没有这张表（可能 TRUNCATE CASCADE 没删表，但 ensure_schema 的 DO block 里有条件检查，可能没触发创建）

**修复进展**：手动执行了 `db_schema.sql`，`node_uri_aliases` 表已创建 ✅

**问题 B：`context_nodes.node_id` 全为 NULL**

- schema migration 要求 `node_id UUID NOT NULL`，但所有 13,481 行的 node_id = NULL
- UPDATE 被 RLS 阻止（即使 `SET app.account_id` 也不够，因为 RLS FORCE 模式下 owner 也受限）
- 修复：`ALTER TABLE context_nodes NO FORCE ROW LEVEL SECURITY` → UPDATE → 重新 FORCE

**修复进展**：
- ✅ `ALTER TABLE context_nodes NO FORCE ROW LEVEL SECURITY`
- ✅ `UPDATE context_nodes SET node_id = gen_random_uuid() WHERE node_id IS NULL`（13,481 行更新）
- ✅ `ALTER TABLE context_nodes ALTER COLUMN node_id SET NOT NULL`
- ✅ `ALTER TABLE context_nodes FORCE ROW LEVEL SECURITY`（恢复）

**问题 C：`vector_index` 缺少 `space_id` / `owner_space` / `node_uri` 等新列**

- db_schema.sql 的 P1A migration block 会添加这些列并 backfill
- 但这个 DO block 在 schema application 时可能因为问题 B 的 NOT NULL 约束失败而整体失败
- 需要重新执行这个 migration block 来补齐 vector_index 的列

**状态**：正在由另一个 agent 修复 schema migration 🔄

## 4. 待完成步骤

### Step 3: Before-dream QA 评测（待修复 schema 后重跑）

```bash
cd /data/Workspace2/repos/longmemeval
OPENAI_API_KEY="sk--hAux09V3aXYBMsaZG_k2w" \
OPENAI_BASE_URL="http://113.46.219.251:8080/v1" \
python scripts/run_qa_eval.py \
  --runner-output output/dream-36-ingest \
  --data-file data/longmemeval_s_dream_36.json \
  --api-url http://127.0.0.1:8090 \
  --account-id longmemeval \
  --reader-model GLM-5 \
  --judge-model GLM-5 \
  --tag before-dream
```

⚠️ **search_recall 表已有 629 条脏数据**（来自失败的 before-dream 尝试），在正式跑 before-dream 前需要清空：
```sql
DELETE FROM search_recall;
```

### Step 4: DeepDream（gpt-4o-mini 驱动）

```bash
cd /data/Workspace2/repos/longmemeval
truncate -s 0 /tmp/ogmem_stdout.log

python scripts/run_deepdream.py \
  --runner-output output/dream-36-ingest \
  --data-file data/longmemeval_s_dream_36.json \
  --api-url http://127.0.0.1:8090 \
  --account-id longmemeval \
  --pause 30 \
  --timeout 600
```

### Step 5: After-dream QA 评测（GLM-5 reader/judge）

同 Step 3，改 `--tag after-dream`。

### Step 6: 对比结果

## 5. 关键文件路径

| 文件 | 路径 | 说明 |
|------|------|------|
| oG-Memory 代码 | `/data/Workspace2/oG-Memory` | dev_0608 分支 |
| 服务端配置 | `/data/Workspace2/oG-Memory/config/ogmem.yaml` | 当前 LLM = gpt-4o-mini |
| schema SQL | `/data/Workspace2/oG-Memory/fs/sql_adapter/db_schema.sql` | 含 P0 + P1A migration |
| ensure_schema | `/data/Workspace2/oG-Memory/fs/sql_adapter/schema.py` | 启动时自动执行 |
| longmemeval 代码 | `/data/Workspace2/repos/longmemeval` | 测试框架 |
| 注入配置 | `/data/Workspace2/repos/longmemeval/configs/dream-36-ingest.toml` | |
| env 配置 | `/data/Workspace2/repos/longmemeval/configs/env.toml` | |
| 数据文件 | `/data/Workspace2/repos/longmemeval/data/longmemeval_s_dream_36.json` | 36题数据集 |
| QA eval 脚本 | `/data/Workspace2/repos/longmemeval/scripts/run_qa_eval.py` | |
| DeepDream 脚本 | `/data/Workspace2/repos/longmemeval/scripts/run_deepdream.py` | |
| run_config | `/data/Workspace2/repos/longmemeval/output/dream-36-ingest/run_config.json` | run_id=00080bf4 |
| 失败的 before-dream | `/data/Workspace2/repos/longmemeval/output/before-dream-00080bf4/` | 34题全部失败，需删除重跑 |
| 服务 stdout | `/tmp/ogmem_stdout.log` | |
| DB 连接 | `host=127.0.0.1 port=5432 dbname=ogmemory user=ogmem password=ogmem123` | |

## 6. owner_space 分布（本次注入）

37 个 owner_space（36 user + 1 agent）：

- `user:dream-36-00080bf4-{qid}`：每个 qid 一个
- `agent:openclaw-longmemeval`：agent 级别（含 case 记忆）

每个 user 的 vector_index 记录数 78–1344 条，差异较大。

## 7. 与上次测试的差异总结

| 维度 | 上次（0611，gpt4o 提取） | 本次（0622，doubao 提取） |
|------|-------------------------|-------------------------|
| run_id | `84325a8b` | `00080bf4` |
| 提取 LLM | gpt-4o | doubao-seed-2-0-code-preview-260215 |
| context_nodes 总数 | 22,875 | 13,481 |
| vector_index 总数 | 32,935 | 23,662 |
| compose/dream LLM | doubao（后来改 gpt-4o-mini） | gpt-4o-mini |
| reader/judge LLM | doubao-seed | GLM-5 |
| schema 问题 | 无（旧 schema 没有 node_id/node_uri_aliases） | 有（需 migration） |
| owner_space 泄漏 bug | 发现并修复 | 已修复（代码已含修复） |
