---
type: repo
status: active
created: 2026-05-13
updated: 2026-05-13
tags: [ogmemory, extraction, storage, architecture, analysis]
verified: true
---

# oG-Memory 抽取与存储模块分析

oG-Memory (ContextEngine) CLI Agent 上下文生命周期引擎的抽取和存储模块深度分析。重点关注阶段④ afterTurn — 抽取到存储的完整链路。

> 代码位置已对照源码验证 (2026-05-13)。具体数据流实例见 [[oG-Memory Extraction and Storage Example]]。

## 1. 整体架构：六个拦截点

| 阶段 | 时机 | 抽取/存储相关操作 |
|------|------|-------------------|
| ① 消息到达 | Agent 推理前 | 预取候选上下文 |
| ② 推理准备 | 组装上下文窗口 | 冷启动注入/分层加载 |
| ③ 工具调用 | 工具执行前后 | 结果压缩/事实抽取 |
| **④ 轮次结束** | Agent 完成一轮后 | **增量抽取/关系构建/异步索引** |
| ⑤ 压缩管理 | 上下文接近满时 | 信号打分/摘要链 |
| ⑥ 会话关闭 | 会话结束 | 任务归档/状态快照 |

## 2. HTTP API 入口

- **主入口**: `server/app.py` — Flask REST API 服务
- **服务层**: `server/memory_service.py` — MemoryService 类

| 端点 | 方法 | 位置 | 说明 |
|------|------|------|------|
| `/api/v1/after_turn` | POST | `app.py:163-174` | **主要写入入口**：抽取 + 持久化 |
| `/api/v1/ingest` | POST | `app.py:177-188` | 单条消息写入 session buffer |
| `/api/v1/ingest_batch` | POST | `app.py:191-201` | 批量消息写入 |
| `/api/v1/compose` | POST | `app.py:148-160` | 搜索记忆，组装上下文 |
| `/api/v1/sessions/{id}/commit` | POST | `app.py:346-356` | 提交 session，触发抽取 |

调用链: `HTTP Request → app.py → MemoryService.after_turn() → SessionManager → TopicBuffer → threshold 达到 → MemoryWriteAPI.commit_session()`

## 3. 抽取模块

### 目录结构

```
extraction/
├── tools.py              # Extractor 类 — 两阶段抽取
├── react_loop.py         # ExtractionReActLoop — ReAct 循环
├── prefetch.py           # MemoryPrefetcher — 预取已有记忆
├── tool_builder.py       # 动态工具生成（从 YAML schema）
├── tool_schemas.py       # Legacy 硬编码工具定义
├── tool_collector.py     # 工具调用统计收集
├── prompts/              # LLM 提示模板
│   └── manager.py        # PromptManager
└── schemas/              # YAML Schema 定义
    ├── registry.py       # SchemaRegistry
    ├── models.py         # MemoryTypeSchema, SchemaField
    └── definitions/      # YAML 文件 (profile, entity, event, preference, pattern, case, skill, tool)
```

### 两阶段抽取

**Phase 1: Span Identification** (`tools.py:295-343`)
- 廉价 JSON-mode LLM 调用识别包含可抽取信息的消息范围
- 输出: `[{"start": int, "end": int, "reason": str, "categories": []}]`

**Phase 2: Span Structuring** (`tools.py:349-397`)
- 对每个 span 执行 focused tool-call extraction
- 支持 **eager mode**（预取 + 单次抽取, `tools.py:555-589`）和 **lazy mode**（ReAct 循环, `tools.py:591-627`）
- **Dual-run 机制**: 每个 span 执行两次（默认 temperature + temperature=0），合并结果

### Extractor 核心 (`tools.py:143-290`)

```
extract(messages, ctx, session_time, session_summary, tool_stats_text):
    1. _detect_language(messages)
    2. _identify_spans(messages, session_time, session_summary)
    3. _structure_spans(spans, messages, ...)
    4. 过滤 confidence < 0.5
```

### ReAct 循环 (`react_loop.py:109-318`)

LLM 拥有工具 `read(uri)`, `list(uri)`, `get_relations(uri)`, `extract_*()` — 在循环中执行。工具从 SchemaRegistry 动态生成。

### YAML Schema 系统

**SchemaRegistry** (`schemas/registry.py:14-149`) 从 `definitions/*.yaml` 加载。Schema 属性：

| 属性 | 作用 |
|------|------|
| `operation_mode` | `upsert` / `add_only` — 决定写入策略 |
| `owner_scope` | `user` / `agent` — 决定 URI 前缀 |
| `directory` | URI 模板 |
| `filename_template` | 文件名模板 |
| `fields` | 字段定义 + 校验规则 |

## 4. 存储模块（DB-first）

### 目录结构

```
fs/sql_adapter/
├── sql_context_fs.py    # SQLContextFS — PostgreSQL 存储
├── schema.py            # 数据库 schema
└── pool.py              # 连接池

commit/
├── context_writer.py    # ContextWriter — 写入编排器
├── policy_router.py     # PolicyRouter — 策略路由
├── merge_policies.py    # MergePolicy (Profile/Aggregate/Append)
├── sql_outbox_store.py  # SQLOutboxStore
├── archive_builder.py   # ArchiveBuilder → ContextNode
└── candidate_pipeline.py # CandidatePipeline

index/
├── outbox_worker.py     # OutboxWorker — 异步索引
├── index_record_builder.py # 构建向量索引记录
├── sql_notify_listener.py  # SQL NOTIFY 监听
├── directory_summarizer.py # 目录摘要
└── repair_job.py        # 自修复
```

### SQLContextFS (`sql_context_fs.py`)

PostgreSQL 单表 (`context_nodes`)，原子 upsert，乐观锁 version 字段，RLS 租户隔离。

| 方法 | 位置 |
|------|------|
| `write_node` | `:326-363` |
| `write_node_with_outbox` | `:365-445` |
| `read_node` | `:447-541` |
| `exists` | `:543-560` |
| `delete_node` | `:562-589` |
| `archive_node` | `:591-628` |
| `list_children` | `:630-669` |
| `move_node` | `:671-909` |

`write_node_with_outbox` 核心流程：单事务内 INSERT/UPDATE `context_nodes` + INSERT `outbox_events` + `pg_notify('ogmem_outbox')`

### ContextWriter (`context_writer.py:22-212`)

写入编排: `PolicyRouter.plan() → WritePlan` → `ArchiveBuilder.build() → ContextNode` → `SQLContextFS.write_node_with_outbox()`

### PolicyRouter (`policy_router.py:26-199`)

Schema-driven 策略路由:

| operation_mode | is_single_file | 策略 |
|----------------|----------------|------|
| `upsert` + single | True | ProfilePolicy (固定 URI, 合并) |
| `upsert` + multi | False | AggregateTopicPolicy (按 slug) |
| `add_only` | - | AppendOnlyPolicy (唯一 ID) |

### 异步索引 (Outbox 模式)

**OutboxWorker** (`outbox_worker.py:56-912`):
- `UPSERT_CONTEXT`: Embed → Upsert L0/L1/L2 向量
- `UPSERT_DIRECTORY`: 生成目录摘要 → Upsert
- `DELETE_CONTEXT` / `ARCHIVE_CONTEXT`: 删除向量记录
- SQL NOTIFY 实时唤醒 + `FOR UPDATE SKIP LOCKED` 多进程并发

## 5. 核心数据模型

### CandidateMemory (抽取结果)

```python
@dataclass
class CandidateMemory:
    category: str          # profile, entity, event, preference, ...
    routing_key: str       # slug 标识符
    abstract: str          # L0 摘要 (~100 token)
    overview: str          # L1 概述 (~500 token)
    content: str           # L2 完整内容
    confidence: float      # 0.0-1.0
    when: str | None
    who: str | None
    where: str | None
    owner_scope: str       # user/agent
```

### ContextNode (持久化节点)

```python
@dataclass
class ContextNode:
    uri: str               # ctx://{account}/users/{user}/memories/{cat}/{slug}
    context_type: str      # MEMORY/SKILL/...
    category: str
    level: int             # 0=L0, 1=L1, 2=L2
    owner_space: str       # user:{user_id} / agent:{agent_id}
    abstract: str
    overview: str
    content: str
    metadata: dict
```

### OutboxEvent / IndexRecord

- `OutboxEvent`: event_id, event_type (UPSERT_CONTEXT/DELETE...), uri, payload, status (PENDING/PROCESSING/DONE/DLQ), retry_count
- `IndexRecord`: id ({uri}:{level}), uri, level (0/1/2), text, filters, metadata

## 6. DB-first vs AGFS 对比

| 特性 | AGFS (旧) | SQL (DB-first) |
|------|-----------|----------------|
| 原子性 | 4 步写入 | 单事务 ON CONFLICT |
| 索引唤醒 | 轮询 (5s) | pg_notify 实时 |
| 并发控制 | 依赖 AGFS | 乐观锁 version |
| 租户隔离 | 路径强制 | SQL RLS + account_id |
| 向量存储 | 外部向量库 | pgvector 同库 |

## 7. 完整数据流 (after_turn)

```
用户消息 → app.py:handle_after_turn()
  → MemoryService.after_turn()
    → SessionManager.add_message() → TopicBuffer 累积
    → threshold 检查 (默认 200 tokens)
    → MemoryWriteAPI.commit_session()
      → CandidatePipeline.extract()
        → Extractor.extract()
          → Phase 1: _identify_spans() → spans[]
          → Phase 2: _structure_spans() → candidates[] (eager/lazy)
        → filter_by_confidence() → deduplicate()
      → ContextWriter.write_candidates()
        → PolicyRouter.plan() → WritePlan
        → ArchiveBuilder.build() → ContextNode
        → SQLContextFS.write_node_with_outbox() (单事务)
        → SQLOutboxStore.register_directory()
    (后台) SQLNotifyListener → OutboxWorker.run_once()
      → embed → vector_index.upsert → 向量索引完成
```
