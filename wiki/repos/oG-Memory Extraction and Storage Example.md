---
type: repo
status: active
created: 2026-05-13
updated: 2026-05-13
tags: [ogmemory, extraction, storage, example, data-flow]
verified: true
---

# oG-Memory 抽取与存储实例详解

以具体例子展示消息从原始对话到抽取再到存储的完整数据流。

> 代码位置已对照源码验证 (2026-05-13)。架构概览见 [[oG-Memory Extraction and Storage Analysis]]。

## 1. 输入：原始对话

```json
{
  "sessionId": "session-abc123",
  "userId": "alice",
  "accountId": "acme-corp",
  "messages": [
    {"role": "user", "content": "我叫 Alice，是一名后端工程师，住在伦敦"},
    {"role": "assistant", "content": "你好 Alice！很高兴认识你。"}
  ]
}
```

入口: `server/app.py:163-174` → `/api/v1/after_turn`

## 2. Phase 1: Span Identification

**代码**: `extraction/tools.py:295-343` (`_identify_spans`)

使用 `_SPAN_PROMPT` (`tools.py:110-136`)，JSON-mode LLM 识别包含可抽取信息的消息范围。

**输出**:
```json
{
  "spans": [
    {"start": 0, "end": 0, "reason": "User introduces themselves...", "categories": ["profile"]}
  ]
}
```

## 3. Phase 2: Span Structuring

**代码**: `tools.py:349-397` → 单 span 走 `_structure_span_eager` (`tools.py:555-589`)

### Prompt 构成

- **System Prompt**: `extraction/prompts/templates/extraction.yaml:18-87` — 抽取指令，含 SPEAKER AWARENESS / TEMPORAL PRECISION / CATEGORY RULES
- **Few-shot Examples**: `extraction.yaml:89-260` — 每个 category 的示例

### Schema 约束

`extraction/schemas/definitions/profile.yaml`:

```yaml
memory_type: profile
operation_mode: upsert          # 合并更新
owner_scope: user
directory: "ctx://{{ account_id }}/users/{{ user_id }}/memories/profile"
filename_template: "content.md"
fields:
  - routing_key (required)
  - abstract (required, ≤200 chars)
  - overview (required)
  - content (required)
  - confidence (required, 0.0-1.0)
  - evidence_quote (required, verbatim quote)
  - attributed_speaker (required)
  - attribution_basis (required, enum: self_first_person/self_named/other_named)
```

### Tool 定义（动态生成）

`extraction/tool_builder.py:51-93` 从 Schema 自动生成 OpenAI function-calling 格式。

### Dual-run 机制

抽取执行两次（默认 temperature + temperature=0），合并去重。

### LLM 输出

```json
[
  {"name": "extract_profile", "input": {"routing_key": "name", "abstract": "用户名叫 Alice", "confidence": 0.95, "evidence_quote": "我叫 Alice", ...}},
  {"name": "extract_profile", "input": {"routing_key": "occupation", "abstract": "后端工程师", ...}},
  {"name": "extract_profile", "input": {"routing_key": "location", "abstract": "住在伦敦", ...}}
]
```

## 4. Tool Call → CandidateMemory

**代码**: `extraction/tool_builder.py:171-243` (`parse_tool_call`)

```python
candidate = CandidateMemory(
    category="profile", routing_key="name",
    abstract="用户名叫 Alice", overview="## 姓名\n- Alice",
    content="用户的名字是 Alice。", confidence=0.95,
    owner_scope="user"
)
```

## 5. 策略路由

**代码**: `commit/policy_router.py:64-125`

profile 是 `upsert + single_file` → **ProfilePolicy** → 固定 URI 合并。

**ProfilePolicy** (`commit/merge_policies.py:38-76`):
1. 构建 URI: `ctx://acme-corp/users/alice/memories/profile/name`
2. 检查是否存在 → 已存在则 merge (乐观锁 version)，不存在则 create

## 6. ArchiveBuilder → ContextNode

**代码**: `commit/archive_builder.py:40-134`

```python
ContextNode(
    uri="ctx://acme-corp/users/alice/memories/profile/name",
    context_type="MEMORY", category="profile", level=3,
    owner_space="user:alice",
    abstract="用户名叫 Alice", overview="## 姓名\n- Alice",
    content="用户的名字是 Alice。",
    metadata={"created_at": "2024-01-15T10:30:00Z", ...}
)
```

## 7. SQLContextFS 写入

**代码**: `fs/sql_adapter/sql_context_fs.py:365-445`

**create** — 单事务:
```sql
INSERT INTO context_nodes (...) VALUES (...)
ON CONFLICT (uri) DO UPDATE SET content=EXCLUDED.content, ...;
INSERT INTO outbox_events (...) VALUES ('UPSERT_CONTEXT', ...);
SELECT pg_notify('ogmem_outbox', ...);
```

**merge** — 乐观锁:
```sql
UPDATE context_nodes SET ..., version = version + 1
WHERE uri = '...' AND version = 1;
```

version 不匹配 → `ConcurrentModificationError` → 重试。

## 8. 异步索引

**投递** (`sql_context_fs.py:365-445`): 同一事务内 INSERT outbox_events + pg_notify

**IndexRecord** (`index/index_record_builder.py:54-138`): 每个 ContextNode 展开为 3 条 (L0=abstract, L1=overview, L2=content)

**OutboxWorker** (`outbox_worker.py:145-183`): `_extract_records → embedder.embed_texts → vector_index.upsert`

## 9. 原始消息存储：Session Archive

**表**: `session_archives` (`session/sql_archive_store.py:92-170`)

```sql
CREATE TABLE session_archives (
    archive_id TEXT, session_id TEXT, account_id TEXT,
    abstract TEXT, overview TEXT,
    messages JSONB,        -- 完整原始消息 JSON 数组（不分块）
    metadata JSONB,
    PRIMARY KEY (account_id, session_id, archive_id)
);
```

**SessionCompressor** (`session/compressor.py:55-120`) 用 LLM 生成 overview + abstract。

关键: 原始消息**不分块**，完整存储在 `messages` JSONB。检索按 session_id 索引，非向量检索。

## 10. 关键问题

| 问题 | 答案 |
|------|------|
| 原始消息是否同步存储？ | 是，但存到 `session_archives` 表（非 `context_nodes`） |
| 存储格式？ | context_nodes: L0/L1/L2 三层；session_archives: 完整 JSONB |
| 是否分块？ | 抽取记忆有 L0/L1/L2 分层；原始消息不分块 |
| Prompt 约束在哪？ | `_SPAN_PROMPT` (tools.py:110-136), `extraction.yaml`, `definitions/*.yaml` |
