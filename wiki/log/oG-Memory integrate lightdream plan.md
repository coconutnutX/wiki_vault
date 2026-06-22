# 整合 B (dev_lightdream) 的有效部分到 A (dev_0608) 的设计方案

  

## 背景

  

- **实现A** (dev_0608): `98c56569` + `b8b6a2b3` — 简洁的 recall 统计

  - `search_recall` 表记录 HTTP API 搜索召回事件

  - `RecallLogger` 写事件 + `query_recall_stats` 聚合查询

  - `acquire_search_recall_stats` operational tool 给 DeepDream LLM 用

  - 总量: 531行代码 + 1张表

  - 整合后表将重命名为 `recall_events`，列 `recalled_at` 改为 `event_at`

  

- **实现B** (dev_lightdream): `c816dea4` + `4db0a4c5` — 类似 openclaw 的复杂信号系统

  - `dream_signals` + `dream_signal_query_seen` + `dream_staged_candidates` + `dream_runs` 4张表

  - `DreamSignalRecorder` 记录 write/recall/session 三种信号

  - `SQLDreamSignalStore` 带评分排序、去重、分阶段运行、独立进程

  - 总量: 1136行 sql_signal_store.py + 311行 recorder.py + 88行 light.py + 79行 operational_tools.py + 161行 archive_chunks.py + 独立进程脚本

  

**核心问题**: B 引入了大量 openclaw 特有的复杂性（4张表、分阶段运行、独立进程、advisory lock），但其中有些有效逻辑（snippet 规范化/哈希、Jaccard 去重、write/session 信号源）值得保留。A 更简洁但只有 recall 统计，缺少 write 信号和 snippet 哈希去重能力。

  

## 分析：B 中哪些值得保留、哪些应该删减

  

### ✅ 保留（有实际价值）

  

| 功能 | 原因 | 来源文件 |

|------|------|---------|

| **snippet 规范化 + quote_hash** | 去除 markdown 标记 → 形成稳定的文本指纹，用于去重和变更检测 | recorder.py `normalize_snippet` / `quote_hash_for_snippet` |

| **Jaccard 去重** | 近重复片段合并，防止同一内容的不同格式被重复晋升 | sql_signal_store.py `_jaccard_similarity` / `_dedupe_candidates` |

| **write 信号源** | 记忆被写入本身也是信号，不只是搜索召回 | recorder.py `record_write_node` |

| **session_archive 信号源** | 会话归档内容也可作为 Dream 原料 | recorder.py `record_session_archive` |

| **dream 源过滤** | 防止 dream 自身输出被循环巩固 | recorder.py `_is_dream_source` |

| **snippet 最短长度检查** | CJK ≥8字符，Latin ≥20字符 — 防止碎片噪声 | recorder.py `_snippet_is_long_enough` |

| **canonical_context_snippet** | 从 ContextNode/RetrievedBlock 提取稳定的短摘要 | recorder.py `canonical_context_snippet` |

  

### ❌ 删减（过度复杂或不适用）

  

| 功能 | 原因 | 来源文件 |

|------|------|---------|

| **`dream_staged_candidates` 表** | openclaw 需要 markdown 分阶段写入，oG-Memory 用数据库直接操作，不需要中间 staging 表 | db_schema.sql |

| **`dream_runs` 表** | 独立进程追踪运行状态，oG-Memory 的 Dream 由 LLM ReAct loop 驱动，不需要 | db_schema.sql |

| **`stage_light_run` 方法** | 分阶段运行编排（lock → aggregate → list → dedupe → stage → finish_run），与 oG-Memory 的即时 ReAct loop 不匹配 | sql_signal_store.py |

| **`_try_light_scope_lock` / advisory lock** | 独立进程并发控制，oG-Memory 不需要 | sql_signal_store.py |

| **`_insert_run` / `_finish_run`** | 运行状态追踪，与 ReAct loop 重复 | sql_signal_store.py |

| **`LightDreamer` / `run_active_scopes`** | 独立进程的调度逻辑 | light.py |

| **`scripts/run_dream_light_service.py`** | 独立进程脚本 | scripts/ |

| **`DreamOperationalTools` (B 版)** | acquire_staged_candidates/mark_promoted 是为 staging 表服务的，A 版的 operational tools 更直接 | operational_tools.py |

| **`get_session_signal_snapshot`** | 依赖 staging 表查询 session chunk 的已记录 snippet，不需要 | sql_signal_store.py |

| **`session/archive_chunks.py`** | 会话分块逻辑，复杂且与 Dream core 无直接关系 | session/archive_chunks.py |

| **`dream_signal_query_seen` 表** | 替代方案：A 已用 SQL `COUNT(DISTINCT query_text)` 和 `COUNT(DISTINCT date_trunc('day', recalled_at))` 实现了同样的聚合，不需要单独表 | db_schema.sql |

| **`list_active_scopes`** | 为独立进程发现哪些 scope 有变化，不需要 | sql_signal_store.py |

  

## 整合策略

  

### 核心思路：把 B 的有效逻辑做成 **scoring 工具**，注入 A 的现有流程

  

A 的现有流程是：

1. 用户搜索 → `RecallLogger.log_recalls()` 写 `search_recall` 表

2. DeepDream ReAct loop → LLM 调用 `acquire_search_recall_stats` → `query_recall_stats` 查聚合统计

  

整合后新增：

1. 用户搜索 → `RecallLogger.log_recalls()` 写 `recall_events` 表（表名更新）

2. 写入记忆 → `RecallLogger.log_write_signal()` 写 `recall_events` 表（新增 write 信号类型）

3. 会话归档 → `RecallLogger.log_session_signal()` 写 `recall_events` 表（新增 session 信号类型）

4. DeepDream → LLM 调用 `acquire_recall_scoring` → 获取增强版统计 + OpenClaw 六维度评分排名（**完全替代原 `acquire_search_recall_stats`**）

5. 原 `acquire_search_recall_stats` 工具被移除，其功能合并到 `acquire_recall_scoring`

  

### 具体设计

  

#### 1. 增强 `search_recall` 表（替代 B 的 4 张表）

  

A 当前表结构：

```sql

search_recall (id, uri, account_id, owner_space, query_text, score, category, recalled_at)

```

  

重命名并扩展为 `recall_events`：

```sql

recall_events (id, uri, account_id, owner_space, query_text, score, category, event_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

               signal_type VARCHAR(16) DEFAULT 'recall',  -- 'recall' | 'write' | 'session'

               snippet TEXT,                               -- 规范化后的摘要

               quote_hash VARCHAR(64))                     -- snippet SHA256 摘要

```

  

重命名原因：表不再仅记录搜索召回事件，而是记录所有与记忆交互的活动事件（recall/write/session），`recall_events` 更准确地反映了"记忆提取/交互事件"的含义。同时 `recalled_at` 列改为 `event_at`，因为写入和归档事件不是"recall"。

  

这样一张表统一记录三种信号，`query_recall_stats` 查询时用 `signal_type` 区分聚合维度。**不需要 `dream_signal_query_seen` 表**，SQL 的 `COUNT(DISTINCT)` 已经实现了唯一查询和跨天计数。

  

#### 2. 从 B 提取纯函数到 `dream/snippet_utils.py`

  

从 `recorder.py` 中提取不依赖 store/Protocol 的纯函数：

  

- `normalize_snippet(text, max_chars=300)` — snippet 规范化

- `quote_hash_for_snippet(snippet)` — SHA256 摘要指纹

- `snippet_is_long_enough(snippet)` — 最短长度检查（CJK ≥8，Latin ≥20）

- `canonical_context_snippet(source_node, retrieved_block)` — 从 ContextNode/RetrievedBlock 提取稳定摘要

- `jaccard_similarity(left, right)` — Jaccard 去重

- `is_dream_source(category, source_uri, source_kind)` — dream 源过滤

  

这些纯函数无依赖，可以直接复用。

  

#### 3. 增强 `RecallLogger` — 新增 write/session 信号记录

  

在 `RecallLogger` 中新增方法：

  

- `log_write_signal(node, ctx)` — 记录写入事件，signal_type='write'

- `log_session_signal(session_id, archive_id, abstract, overview, ctx)` — 记录会话归档，signal_type='session'

  

两者都调用 `canonical_context_snippet` / `snippet_is_long_enough` / `is_dream_source` 进行过滤，最终 INSERT 到 `recall_events` 表。

  

#### 4. 增强 `query_recall_stats` — 多维度聚合

  

当前 A 的 `query_recall_stats` 只统计 recall 信号。扩展为：

  

```sql

SELECT

    r.uri,

    MAX(r.category) AS category,

    MAX(r.snippet) AS snippet,                     -- 新增

    MAX(r.quote_hash) AS quote_hash,               -- 新增

    SUM(CASE WHEN r.signal_type='recall' THEN 1 ELSE 0 END) AS recall_count,

    SUM(CASE WHEN r.signal_type='write' THEN 1 ELSE 0 END) AS write_count,

    SUM(CASE WHEN r.signal_type='session' THEN 1 ELSE 0 END) AS session_count,

    COUNT(*) AS total_signal_count,                -- 新增

    COUNT(DISTINCT r.query_text) AS unique_queries,

    COUNT(DISTINCT date_trunc('day', r.event_at)) AS recall_days,

    AVG(CASE WHEN r.signal_type='recall' THEN r.score ELSE NULL END) AS avg_score, -- 新增

    MIN(r.event_at) AS first_recalled_at,

    MAX(r.event_at) AS last_recalled_at

FROM recall_events r

WHERE ...

GROUP BY r.uri

HAVING COUNT(*) >= min_recall_count

ORDER BY total_signal_count DESC, unique_queries DESC, recall_days DESC

```

  

这样一张表就实现了 B 的 `dream_signals` 表的大部分聚合功能（write_count, session_count, recall_count, unique_queries, recall_days, avg_score），**无需额外表**。

  

#### 5. 新增 `acquire_recall_scoring` operational tool — 替代 `acquire_search_recall_stats` + OpenClaw 六维度评分

  

这是整合的核心新增项。将原 `acquire_search_recall_stats` 和 OpenClaw 的 `rankShortTermPromotionCandidates` 评分逻辑合并为一个 operational tool，让 DeepDream LLM 可以获取代码计算的统计 + 评分排名，而不需要 LLM 自己做评分判断。原 `acquire_search_recall_stats` 工具被移除。

  

新文件 `dream/recall_scoring_tool.py`：

  

```python

@operational_tool

def acquire_recall_scoring(

    min_total_signals: int = 3,

    days: int = 14,

    min_score: float = 0.75,

    limit: int = 20,

    ctx: RequestContext = None,

    read_api: ReadAPI = None,

) -> dict:

    """Rank memories by OpenClaw-style six-dimension scoring for dream consolidation.

  

    Returns scored candidates with per-component breakdowns and raw statistics,

    enabling the LLM to understand WHY a memory is a strong candidate.

  

    **This replaces acquire_search_recall_stats** — it returns all the same

    statistics (recall_count, unique_queries, recall_days) PLUS scoring.

  

    **Scoring dimensions** (all code-calculated, no LLM involved):

    - frequency: log1p(total_signal_count) / log1p(10) — how often seen

    - relevance: average search score across recall events — how relevant

    - diversity: max(unique_queries, recall_days) / 5 — how broad

    - recency: exponential decay with 14-day half-life — how recent

    - consolidation: max(spacing × days_span, write_count/3) — how durable

    - conceptual: unique_tokens_in_snippet / 30 — how information-rich

  

    :param min_total_signals: Minimum total signal count (recall+write+session) threshold.

    :param days: Look-back window in days.

    :param min_score: Minimum composite score to include a candidate.

    :param limit: Max number of scored candidates to return.

    """

```

  

评分逻辑参考 OpenClaw 的 `rankShortTermPromotionCandidates`，但简化：

- 不需要 `conceptTags`（oG-Memory 没有此概念），conceptual 维度从 snippet 的 unique token 数量近似（`unique_tokens / 30`）

- 不需要 `phaseBoost`（oG-Memory 没有 light/rem 分阶段信号）

- 用 SQL 聚合替代 B 的 `dream_signal_query_seen` 表

- `acquire_search_recall_stats` 工具完全被替代，不再保留

  

权重参考 OpenClaw：

```

frequency=0.24, relevance=0.30, diversity=0.15, recency=0.15, consolidation=0.10, conceptual=0.06

```

  

#### 6. 在 DeepDream ReAct loop 中注入新工具

  

修改 `dream/operational_tools.py` 的 `DEEP_DREAM_MVP_OPERATIONAL` 列表：

- 移除 `acquire_search_recall_stats`（被 `acquire_recall_scoring` 替代）

- 加入 `acquire_recall_scoring`

  

#### 7. 在 WriteAPI / ReadAPI 中调用 `log_write_signal`

  

在 `WriteAPI.write_memory()` 成功写入后，调用 `RecallLogger.log_write_signal()` 记录 write 信号。这使 Dream 能发现哪些记忆是被主动写入的（不只是搜索召回的）。

  

## 实施步骤

  

### Step 1: Cherry-pick B 的两个 commit 到工作分支

  

```bash

git checkout -b dev_lightdream_integrate dev_0608

git cherry-pick c816dea4 4db0a4c5

```

  

保留历史记录（taoying 的作者信息不变）。

  

### Step 2: 删除/回退所有 B 引入的不需要文件和修改

  

Cherry-pick 后会带入 B 的所有文件变更，需要删除或回退以下内容：

  

**需要完全删除的文件**（B 新增的，A 分支上不存在）：

- `dream/light.py` — 独立进程调度器

- `dream/recorder.py` — 会被重构到 `snippet_utils.py` + `RecallLogger`

- `dream/sql_signal_store.py` — 会被重构到 `RecallLogger` + `recall_scoring_tool.py`

- `scripts/run_dream_light_service.py` — 独立进程脚本

- `session/archive_chunks.py` — 过度复杂的会话分块逻辑

- `session/sql_archive_store.py` — 与 Dream core 无关

- `docs/dream_light_deepdream_pr96_integration_design.md` / `docs/dream_light_design.md` — B 的设计文档（不需要保留）

- `tests/unit/dream/test_light.py`

- `tests/unit/dream/test_sql_signal_store.py`

- `tests/unit/dream/test_recorder.py`

- `tests/unit/dream/test_operational_tools.py`（B 版）

- `tests/unit/scripts/test_run_dream_light_service.py`

- `tests/unit/service/test_dream_light_service.py`

- `tests/unit/service/test_read_api_dream_recorder.py`

- `tests/unit/service/test_write_api_dream_recorder.py`

- `tests/unit/session/test_sql_archive_store.py`

  

**需要回退修改的文件**（A 分支已有，B 改动了但改动不需要）：

- `config/env.example` — 回退 B 添加的 DREAM_LIGHT 环境变量

- `config/ogmem.reference.yaml` — 回退 B 添加的 dream_light 配置段

- `deploy/deploy.env` / `deploy/deploy.sh` / `deploy/ogmemory.example.yaml` / `deploy/README.md` — 回退 B 添加的 dream_light service 相关部署配置

- `docker/Dockerfile.standalone` / `docker/entrypoint-standalone.sh` — 回退 B 添加的 dream_light 进程启动逻辑

- `docs/OGMEMORY_ENV.md` / `docs/deployment/` 文件 — 回退 B 添加的 dream_light 文档段

- `providers/unified_config.py` — 回退 B 添加的 dream_light 配置解析

- `session/archive_store.py` — 回退 B 对 archive_store 的修改

- `dream/__init__.py` — 回退 B 添加的 light/recorder/sql_signal_store 导出

  

**保留修改的文件**（B 的修改是有价值的或需要保留后进一步修改）：

- `fs/sql_adapter/db_schema.sql` — 保留 dream_signals 等表的 schema 添加，后续会删除不需要的表，只保留需要的 `recall_events` 表

- `dream/recall_logger.py` — 保留 A 版本，后续增强

- `dream/recall_stats_tool.py` — 保留当前版本，后续会被替换为 `recall_scoring_tool.py`

- `providers/llm/openai_llm.py` — 检查 B 的修改是否与 dream 有关，如果是则回退

  

### Step 3: 创建 `dream/snippet_utils.py`

  

从删除的 `recorder.py` 中提取纯函数：

- `normalize_snippet`

- `quote_hash_for_snippet`

- `snippet_is_long_enough`

- `canonical_context_snippet`

- `canonical_session_snippet`

- `is_dream_source`

- `jaccard_similarity`（从 `sql_signal_store.py` 提取）

  

### Step 4: 重命名 `search_recall` → `recall_events` 并扩展 schema

  

在 `db_schema.sql` 中：

- 重命名 `search_recall` 表为 `recall_events`

- 重命名 `recalled_at` 列为 `event_at`

- 添加 `signal_type VARCHAR(16) DEFAULT 'recall'`, `snippet TEXT`, `quote_hash VARCHAR(64)` 列

- 更新所有相关索引名和 RLS policy 名

  

### Step 5: 增强 `RecallLogger`

  

- 新增 `log_write_signal(node, ctx)`

- 新增 `log_session_signal(...)`

- 增强 `query_recall_stats` — 多维度聚合（write_count, session_count, total_signal_count, avg_score, snippet, quote_hash）

  

### Step 6: 创建 `dream/recall_scoring_tool.py`

  

移植 OpenClaw 六维度评分逻辑为 operational tool。

  

### Step 7: 更新注入点

  

- `dream/operational_tools.py` → 加入 `acquire_recall_scoring`

- `server/memory_service.py` → WriteAPI 写入后调用 `log_write_signal`

- `service/api.py` → ReadAPI 注入新方法

  

### Step 8: 更新测试

  

- 新增 `tests/unit/dream/test_snippet_utils.py`

- 新增 `tests/unit/dream/test_recall_scoring_tool.py`

- 更新 `tests/unit/dream/test_recall_logger.py`（write/session 信号，表名变更）

- 移除 `dream/recall_stats_tool.py`（被 `recall_scoring_tool.py` 替代）

- 更新 `tests/unit/dream/test_recall_stats_tool.py` → 改为 `test_recall_scoring_tool.py`

- 删除 B 的冗余测试文件

  
  

## 关键设计决策

  

1. **一张表 vs 四张表**: 用 `recall_events` 的 `signal_type` 列替代 B 的 `dream_signals + dream_signal_query_seen + dream_staged_candidates + dream_runs` 四张表。SQL `COUNT(DISTINCT)` 替代 `dream_signal_query_seen` 的 query/day 去重功能。原 `search_recall` 表重命名为 `recall_events`，`recalled_at` 列重命名为 `event_at`。

  

2. **评分作为 operational tool**: OpenClaw 的评分逻辑是代码计算的，放在 server 端做成 tool 让 LLM 调用，而不是 LLM 自己判断。这保持了评分的确定性和可预测性。

  

3. **snippet 去重作为 post-query 步骤**: `query_recall_stats` 返回聚合结果后，在 Python 层用 `jaccard_similarity` 去重近重复候选，而不是在 SQL 层或用单独的 staging 表。同时 `RecallLogger` 类名保留不变（即使底层表改为 `recall_events`），因为它仍然承担"记录和查询 recall 相关事件"的职责。

  

4. **不引入独立进程**: oG-Memory 的 Dream 由 DeepDream ReAct loop 驱动，信号采集在主进程的 HTTP API 调用中完成，不需要独立进程。