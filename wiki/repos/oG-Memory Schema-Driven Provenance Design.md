---
type: repo
status: active
created: 2026-05-13
updated: 2026-05-13
tags: [ogmemory, provenance, schema-driven, design, architecture]
---

# oG-Memory Schema-Driven Provenance 设计

> 深入分析如何在 oG-Memory 中实现 schema-driven 溯源系统。
> 初版三方案对比见 [[oG-Memory Provenance Design Analysis]]。
> 相关调研见 [[Memory Provenance in Agentic Systems]]。

---

## 0. 前置修复：当前 Pipeline 的溯源缺陷

在实现 Provenance 系统之前，需要先修复现有的数据链路断裂。这些问题是独立于 Provenance 设计的 bugfix 级改进。

### 缺陷 1：extraction pipeline 丢弃消息身份（致命）

`_build_incremental_extraction_state` (`memory_service.py:1928-1931`) 将 SessionMessage 转为匿名 dict：

```python
extraction_messages = [
    {"role": message.role, "content": message.content}   # id 和 created_at 被丢弃
    for message in incremental
    if message.role != "assistant"
]
```

同时 `archive_snapshot = list(incremental)` 保留了完整的 SessionMessage（包含 assistant 消息）。后果：

```
incremental:        [user_m3, assistant_m4, user_m5]
extraction_messages: [user_m3, user_m5]              ← 索引 0, 1
archive_snapshot:    [user_m3, assistant_m4, user_m5] ← 索引 0, 1, 2
```

extraction 中的 span 索引和 archive 中的消息索引**永远对不上**。这是所有溯源方案的根源性障碍。

**修复**：在 `extraction_messages` 中保留 `id` 字段（~1 行改动）。

### 缺陷 2：archive 不是不可变的

`session_archives` 使用 `ON CONFLICT DO UPDATE`，`messages` 字段可以被覆写：

```sql
ON CONFLICT (account_id, session_id, archive_id) DO UPDATE SET
    messages = EXCLUDED.messages,   -- 消息可被覆写
```

如果 archive 内容变了，所有溯源引用失效。实际上 archive_id 含时间戳+UUID，碰撞概率极低。

**修复**：改为 `DO NOTHING`（1 行改动），archive 写入即不可变。

### 缺陷 3：session 按 threshold 切成多个 archive

一次对话被 threshold 切成多个 archive：

```
session_archives:
  (acme, s-abc, 20260513_100000_a1b2) → messages[0..5]
  (acme, s-abc, 20260513_103000_d4e5) → messages[6..9]
  (acme, s-abc, 20260513_110000_g7h8) → messages[10..15]
```

这不是 bug，是设计选择。溯源应以 **archive_id + message_id** 为锚点，而非 session_id + span 索引。查询完整对话只需 `SELECT ... WHERE session_id = %s ORDER BY created_at`，不需要额外的 manifest 结构。

---

## 1. 为什么 Schema-Driven 是正确选择

oG-Memory 的核心设计理念是 **schema-driven**：记忆类型、字段定义、写入策略、URI 模板全部由 YAML schema 声明，代码只处理通用逻辑。

当前 schema 系统已经做到：
- 新增记忆类型 = 新增一个 YAML 文件，零代码改动
- `operation_mode` 决定写入策略（upsert/add_only）
- `owner_scope` 决定 URI 前缀（user/agent）
- `fields` 定义 LLM 抽取的输入输出

溯源理应遵循同一模式：**在 schema 中声明溯源需求，框架自动处理**。这不仅是一致性问题——如果溯源是每个记忆类型都需要的能力，它就不应该在每个类型里重复实现。

---

## 2. Provenance 是一个通用问题

### 2.1 不只是 context_nodes → session_archives

溯源不仅存在于"抽取记忆 → 原始消息"这一条路径。放眼整个系统：

| 生产者 (Producer) | 产出物 (Artifact) | 需要溯源到 |
|--------------------|--------------------|------------|
| Extraction | context_node (profile/entity/event) | session_archive 中的原始消息 |
| Extraction | context_node 中的 evidence_quote | 具体消息中的句子 |
| Dreaming (规划中) | dream_output (强化/合并后的记忆) | 参与的 context_nodes |
| Graph DB (规划中) | graph_node / graph_edge | 产生该关系的 context_node |
| Vector Index | IndexRecord (L0/L1/L2) | 对应的 context_node |
| Session Compressor | session_archive.overview | session 中的原始消息 |

这是一个 **有向无环图 (DAG)**：

```
原始消息 (session_archives)
    ↓ extraction
context_nodes (抽取记忆)
    ↓ dreaming (规划中)
dream_outputs (强化后的记忆)
    ↓ graph construction (规划中)
graph_nodes / graph_edges
    ↓ indexing
vector_records (IndexRecord)
```

每一层都需要知道"我来自哪里"。

### 2.2 当前 metadata JSONB 的局限

所有这些溯源信息目前都塞在 `context_nodes.metadata` JSONB 中：

```json
{
  "when": "2024-01-15",
  "who": "alice",
  "where": "london",
  "usage_count": 3,
  "tool_stats": {...},
  "_relations": [...]
}
```

问题：
1. **无 schema 约束**：metadata 是自由形式的 dict，任何代码都可以塞任何 key
2. **查询受限**：JSONB 支持 `->` 索引查询，但不如专用列高效，且语义不明确
3. **扩展性差**：随着溯源需求增长，metadata 会变成垃圾场
4. **跨组件不通用**：context_nodes 的 metadata 结构与 session_archives 的 metadata 结构完全不同

---

## 3. 设计：Provenance 作为一等公民

### 3.1 Provenance ID 方案

基于 message ID（而非 span 索引）设计统一 ID 体系：

```
prov:{source_type}:{source_id}:{detail}
```

具体示例：

| 来源 | Provenance ID |
|------|---------------|
| Archive 中的特定消息 | `prov:archive:20260513_100000_a1b2:msgs:msg_a3f8,msg_b7c1` |
| Archive 整体 | `prov:archive:20260513_100000_a1b2:all` |
| 另一个 context_node | `prov:memory:ctx://acme/users/alice/memories/profile/name:v3` |
| Dream 输出 | `prov:dream:dream-20260513-001:phase:reinforce` |
| Graph 构建 | `prov:graph:build-20260513:edge:alice-knows-bob` |

为什么用 `archive_id` 而非 `session_id`：
- 一个 session 有多个 archive（按 threshold 切分）
- archive 是不可变的写入单元，session 不是
- 溯源目标是稳定的锚点，archive_id 天然满足

为什么用 `message_id` 而非 span 索引：
- span 索引基于过滤后的 extraction_messages（无 assistant），archive 索引包含 assistant，两者永远对不上
- message_id 是稳定唯一标识，不受数组过滤/重排影响

### 3.2 SourceRef 数据模型

```python
@dataclass
class SourceRef:
    """统一溯源引用。"""
    source_type: str        # "archive" | "memory" | "dream" | "graph"
    source_id: str          # archive_id / memory_uri / dream_id / ...
    detail: str | None      # "msgs:msg_a3f8,msg_b7c1" / "v3" / None
    timestamp: str          # ISO8601，创建时间

    def to_provenance_id(self) -> str:
        parts = [f"prov:{self.source_type}:{self.source_id}"]
        if self.detail:
            parts.append(self.detail)
        return ":".join(parts)

    @staticmethod
    def from_provenance_id(pid: str) -> "SourceRef":
        # prov:archive:20260513_100000_a1b2:msgs:msg_a3f8,msg_b7c1
        _, source_type, source_id, *detail_parts = pid.split(":")
        return SourceRef(
            source_type=source_type,
            source_id=source_id,
            detail=":".join(detail_parts) if detail_parts else None,
            timestamp="",
        )
```

### 3.3 Schema 扩展设计

#### YAML Schema 新增 `provenance` 块

```yaml
# extraction/schemas/definitions/profile.yaml
memory_type: profile
description: "Extract individual user profile attributes..."
directory: "ctx://{{ account_id }}/users/{{ user_id }}/memories/profile"
filename_template: "content.md"
operation_mode: upsert
owner_scope: user
enabled: true

# 新增：溯源声明
provenance:
  enabled: true
  sources:
    - source_type: "archive"
      auto_inject: true                # 自动注入，不让 LLM 填
      granularity: "messages"          # 精确到消息 ID
      multi_source: true              # merge 后追加，保留完整溯源链
  metadata_key: "provenance_ids"
```

不同 schema 可以有不同的溯源策略：

```yaml
# event.yaml — 事件类型，精确到消息
provenance:
  enabled: true
  sources:
    - source_type: "archive"
      auto_inject: true
      granularity: "messages"
      multi_source: false
  metadata_key: "provenance_ids"

# skill.yaml — 技能累积，可能溯源到其他记忆
provenance:
  enabled: true
  sources:
    - source_type: "archive"
      auto_inject: true
      granularity: "messages"
    - source_type: "memory"
      auto_inject: false              # 需要代码逻辑注入
      granularity: "uri"
  metadata_key: "provenance_ids"
```

#### Schema 模型扩展

```python
@dataclass(frozen=True)
class MemoryTypeSchema:
    # ... 现有字段 ...
    provenance: ProvenanceConfig | None = None

@dataclass(frozen=True)
class ProvenanceConfig:
    enabled: bool = False
    sources: list[ProvenanceSource] = field(default_factory=list)
    metadata_key: str = "provenance_ids"

@dataclass(frozen=True)
class ProvenanceSource:
    source_type: str       # "archive" | "memory" | "dream" | "graph"
    auto_inject: bool = True
    granularity: str = "messages"  # "messages" | "uri" | "version"
    multi_source: bool = False
```

### 3.4 框架层：ProvenanceManager

```python
class ProvenanceManager:
    """管理所有溯源操作的通用组件。"""

    def inject_provenance(
        self,
        candidate: CandidateMemory,
        schema: MemoryTypeSchema,
        context: ExtractionContext,
    ) -> None:
        """根据 schema 的 provenance 声明，自动注入溯源信息。"""
        if not schema.provenance or not schema.provenance.enabled:
            return

        source_refs = []
        for source_decl in schema.provenance.sources:
            if not source_decl.auto_inject:
                continue

            if source_decl.source_type == "archive":
                ref = SourceRef(
                    source_type="archive",
                    source_id=context.archive_id,
                    detail=f"msgs:{','.join(context.message_ids)}"
                           if context.message_ids else None,
                    timestamp=datetime.now(UTC).isoformat(),
                )
                source_refs.append(ref)

        candidate.provenance_ids = [r.to_provenance_id() for r in source_refs]

    def build_metadata(self, candidate, schema, plan) -> dict:
        """将溯源信息写入 metadata。"""
        metadata = {}
        if hasattr(candidate, "provenance_ids") and candidate.provenance_ids:
            key = schema.provenance.metadata_key if schema.provenance else "provenance_ids"
            metadata[key] = candidate.provenance_ids
        return metadata

    def merge_provenance(
        self,
        existing_node: ContextNode,
        candidate: CandidateMemory,
        schema: MemoryTypeSchema,
    ) -> list[str]:
        """Merge 时合并溯源列表（追加而非覆盖）。"""
        key = schema.provenance.metadata_key
        existing_ids = existing_node.metadata.get(key, [])
        new_ids = getattr(candidate, "provenance_ids", [])
        # 去重合并，保留顺序
        return list(dict.fromkeys(existing_ids + new_ids))
```

### 3.5 全链路改动点

#### 前置修复（P0）：extraction 保留 message ID

```python
# memory_service.py — _build_incremental_extraction_state
extraction_messages = [
    {"role": message.role, "content": message.content, "id": message.id}
    for message in incremental
    if message.role != "assistant"
]
```

#### 前置修复（P1）：archive 不可变

```sql
-- session/sql_archive_store.py
ON CONFLICT (account_id, session_id, archive_id) DO NOTHING;
```

#### Provenance 改动（P2）

**1. `_Span` 携带 message_ids：**

```python
@dataclass
class _Span:
    start: int
    end: int
    reason: str = ""
    categories: list[str] = field(default_factory=list)
    message_ids: list[str] = field(default_factory=list)
```

`_identify_spans` 解析后填充：

```python
span.message_ids = [
    m["id"] for m in messages[span.start : span.end + 1] if "id" in m
]
```

**2. CandidateMemory 新增字段：**

```python
@dataclass
class CandidateMemory:
    # ... 现有字段 ...
    source_message_ids: list[str] = field(default_factory=list)
    provenance_ids: list[str] = field(default_factory=list)
```

**3. `_structure_spans` 中注入：**

```python
# extraction/tools.py
extraction_ctx = ExtractionContext(
    archive_id=archive_id,       # 从 _background_extract_write 传入
    message_ids=prepared.span.message_ids,
)
for candidate in candidates:
    candidate.source_message_ids = prepared.span.message_ids
    self._provenance_mgr.inject_provenance(candidate, schema, extraction_ctx)
```

**4. ArchiveBuilder 写入 metadata：**

```python
# commit/archive_builder.py
metadata["source_message_ids"] = candidate.source_message_ids
prov_metadata = self._provenance_mgr.build_metadata(candidate, schema, plan)
metadata.update(prov_metadata)
```

**5. Merge 策略合并溯源：**

```python
# commit/merge_policies.py
merged_fields["source_message_ids"] = (
    existing.metadata.get("source_message_ids", [])
    + candidate.source_message_ids
)
merged_fields["provenance_ids"] = self._provenance_mgr.merge_provenance(
    existing, candidate, schema
)
```

**6. IndexRecord 传递到向量索引：**

```python
# index/index_record_builder.py
for key in ("source_message_ids", "provenance_ids"):
    if key in node.metadata:
        record.metadata[key] = node.metadata[key]
```

### 3.6 多来源 Merge 示例

```
Turn 1: "我叫 Alice，是后端工程师"
  → extract_profile(occupation)
  → provenance_ids: ["prov:archive:20260513_100000_a1b2:msgs:msg_x1"]

Turn 2: "我现在转去学前端了"
  → extract_profile(occupation) → merge
  → provenance_ids: [
      "prov:archive:20260513_100000_a1b2:msgs:msg_x1",       ← 旧来源
      "prov:archive:20260513_103000_d4e5f:msgs:msg_y3",      ← 新来源
    ]
```

### 3.7 查询

#### 正向查询：记忆 → 来源消息

```python
def resolve_provenance(self, node: ContextNode) -> list[SourceDetail]:
    ids = node.metadata.get("provenance_ids", [])
    results = []
    for pid in ids:
        ref = SourceRef.from_provenance_id(pid)
        if ref.source_type == "archive":
            # 按 session_id 查所有 archive，在 messages JSONB 中匹配 message ID
            archives = self._archive_store.list_by_session(
                ref.source_id, account_id
            )
            if ref.detail and ref.detail.startswith("msgs:"):
                target_ids = ref.detail[5:].split(",")
                for archive in archives:
                    matched = [
                        m for m in archive.messages
                        if m.get("id") in target_ids
                    ]
                    if matched:
                        results.append(SourceDetail(ref, matched))
                        break
        elif ref.source_type == "memory":
            source_node = self._fs.read_node(ref.source_id)
            results.append(SourceDetail(ref, source_node))
    return results
```

#### 反向查询：session → 产生的记忆

```sql
SELECT uri, metadata->>'provenance_ids' as prov_ids
FROM context_nodes
WHERE account_id = %s
  AND metadata->>'provenance_ids' LIKE '%prov:archive:%';
```

或更精确地按 message_id 查（需要 GIN 索引支持）。

---

## 4. 扩展到其他组件

### 4.1 Dreaming（规划中）

```python
dream_output.provenance_ids = [
    f"prov:memory:{input_node.uri}:v{input_node.version}"
    for input_node in dream_inputs
]
```

### 4.2 Graph DB（规划中）

```python
edge.provenance_ids = [
    f"prov:memory:{source_node.uri}:v{source_node.version}",
    f"prov:archive:{archive_id}:msgs:{','.join(msg_ids)}",
]
```

### 4.3 Vector Index

IndexRecord.metadata 自然传递 provenance_ids，compose API 返回结果直接携带来源信息。

---

## 5. 阶段规划

### P0：前置修复 — 保留 message ID（~15 行，4 文件）

| 文件 | 改动 |
|------|------|
| `server/memory_service.py:1928` | extraction_messages 保留 `id` 字段 |
| `extraction/tools.py:50` | `_Span` 新增 `message_ids` 字段 |
| `extraction/tools.py` `_identify_spans` | 解析后填充 `message_ids` |
| `core/models.py:145` | CandidateMemory 新增 `source_message_ids` |

独立于 Provenance 系统，纯 bugfix。修复 span 索引与 archive 索引不对齐的根源问题。

### P1：前置修复 — archive 不可变（1 行）

| 文件 | 改动 |
|------|------|
| `session/sql_archive_store.py:134` | `DO UPDATE` → `DO NOTHING` |

### P2：Schema-Driven Provenance（~60 行，8 文件）

| 文件 | 改动 |
|------|------|
| `extraction/schemas/models.py` | 新增 `ProvenanceConfig`、`ProvenanceSource` |
| `extraction/schemas/registry.py` | 解析 YAML `provenance:` 块 |
| `extraction/schemas/definitions/*.yaml` | 各 schema 加 `provenance:` 声明 |
| `extraction/tools.py` | 构建 ExtractionContext，调用 ProvenanceManager |
| `commit/archive_builder.py` | 写入 metadata |
| `commit/merge_policies.py` | merge 时合并 provenance_ids |
| `index/index_record_builder.py` | 传递到向量索引 |
| 新增 `provenance/manager.py` | ProvenanceManager + SourceRef |

### P3（可选）：独立 Provenance 表

如果 metadata JSONB 查询性能不足：

```sql
CREATE TABLE provenance_edges (
    id             UUID PRIMARY KEY,
    target_uri     TEXT NOT NULL,
    provenance_id  TEXT NOT NULL,
    source_type    TEXT NOT NULL,
    source_ref     JSONB,
    created_at     TIMESTAMP
);
```

从 metadata JSONB 到专用表的平滑迁移——schema 声明和 ProvenanceManager 不需要改，只改存储层。

---

## 6. 设计总结

```
┌─────────────────────────────────────────────────────────┐
│ YAML Schema                                             │
│   provenance:                                           │
│     enabled: true                                       │
│     sources: [{source_type, auto_inject, granularity}]  │
└────────────────────────┬────────────────────────────────┘
                         │ 声明
                         ▼
┌─────────────────────────────────────────────────────────┐
│ ProvenanceManager (通用组件)                             │
│   inject_provenance()  — 根据声明自动注入               │
│   build_metadata()     — 写入 metadata                  │
│   merge_provenance()   — merge 时合并溯源列表           │
│   resolve()            — 解析 provenance_id → 来源详情  │
└────────────────────────┬────────────────────────────────┘
                         │ 使用
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
   Extraction        Dreaming        Graph DB
   (P2)              (规划中)         (规划中)
          │              │              │
          ▼              ▼              ▼
   context_nodes    dream_outputs    graph_edges
   metadata JSONB   metadata        edge attributes
          │
          ▼
   IndexRecord.metadata
   (向量检索带溯源)
```

核心原则：
1. **前置修复优先** — P0/P1 修复 pipeline 断裂，独立于 Provenance 设计
2. **Schema 声明，框架执行** — 溯源需求在 YAML 中声明，代码处理通用逻辑
3. **Message ID 而非 span 索引** — 稳定标识，不受数组过滤/重排影响
4. **archive_id 而非 session_id** — archive 是不可变锚点，session 不是
5. **多来源列表** — merge 追加而非覆盖，保留完整溯源链
6. **渐进式架构** — P2 用 metadata JSONB，P3 可平滑迁移到独立表
