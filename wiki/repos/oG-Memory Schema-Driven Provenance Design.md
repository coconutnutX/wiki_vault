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

设计一个统一的 Provenance ID 体系，能跨组件使用：

```
prov:{source_type}:{source_id}:{detail}
```

具体示例：

| 来源 | Provenance ID |
|------|---------------|
| Session Archive 中的消息范围 | `prov:session:session-abc123:span:0-2` |
| Session Archive 整体 | `prov:session:session-abc123:all` |
| 另一个 context_node | `prov:memory:ctx://acme/users/alice/memories/profile/name:v3` |
| Dream 输出 | `prov:dream:dream-20260513-001:phase:reinforce` |
| Graph 构建 | `prov:graph:build-20260513:edge:alice-knows-bob` |

特点：
- **全局唯一**：`prov:` 前缀 + source_type + source_id 足以保证唯一性
- **可解析**：通过 `:` 分隔可以反解出 source_type 和 source_id
- **版本感知**：对于 context_node 可以带版本号（`:v3`），指向具体版本
- **多来源支持**：一个产出物可以有多个 provenance ID（列表）

### 3.2 SourceRef 数据模型

```python
@dataclass
class SourceRef:
    """统一溯源引用。"""
    source_type: str        # "session" | "memory" | "dream" | "graph" | "vector"
    source_id: str          # session_id / memory_uri / dream_id / ...
    detail: str | None      # "span:0-2" / "v3" / "phase:reinforce" / None
    timestamp: str          # ISO8601，创建时间

    def to_provenance_id(self) -> str:
        parts = [f"prov:{self.source_type}:{self.source_id}"]
        if self.detail:
            parts.append(self.detail)
        return ":".join(parts)

    @staticmethod
    def from_provenance_id(pid: str) -> "SourceRef":
        # prov:session:session-abc:span:0-2
        #   → source_type="session", source_id="session-abc", detail="span:0-2"
        _, source_type, source_id, *detail_parts = pid.split(":")
        return SourceRef(
            source_type=source_type,
            source_id=source_id,
            detail=":".join(detail_parts) if detail_parts else None,
            timestamp="",  # 需要额外查询
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
  enabled: true                        # 是否启用溯源
  sources:                             # 溯源来源声明
    - source_type: "session"           # 来源类型
      auto_inject: true                # 自动注入，不让 LLM 填
      granularity: "span"             # 粒度：span (消息范围) | session (整个 session)
      multi_source: true              # 支持多来源（merge 后追加）
  metadata_key: "provenance_ids"      # 存入 metadata 的 key 名

fields:
  # ... 现有字段不变 ...
```

不同 schema 可以有不同的溯源策略：

```yaml
# event.yaml — 事件类型，需要精确到消息
provenance:
  enabled: true
  sources:
    - source_type: "session"
      auto_inject: true
      granularity: "span"
      multi_source: false             # event 不需要多来源
  metadata_key: "provenance_ids"

# skill.yaml — 技能累积，溯源到之前的记忆
provenance:
  enabled: true
  sources:
    - source_type: "session"
      auto_inject: true
      granularity: "span"
    - source_type: "memory"           # 可能从其他记忆推导出来
      auto_inject: false              # 需要代码逻辑注入
      granularity: "uri"
  metadata_key: "provenance_ids"
```

#### SchemaField 扩展

```python
@dataclass(frozen=True)
class SchemaField:
    name: str
    field_type: FieldType
    required: bool = False
    description: str = ""
    default: Any = None
    enum: List[str] | None = None
    # 新增
    auto_inject: bool = False
    inject_from: str | None = None    # "extraction_context" | "dream_context" | ...
```

#### MemoryTypeSchema 扩展

```python
@dataclass(frozen=True)
class MemoryTypeSchema:
    # ... 现有字段 ...
    provenance: ProvenanceConfig | None = None  # 新增

@dataclass(frozen=True)
class ProvenanceConfig:
    enabled: bool = False
    sources: list[ProvenanceSource] = field(default_factory=list)
    metadata_key: str = "provenance_ids"

@dataclass(frozen=True)
class ProvenanceSource:
    source_type: str       # "session" | "memory" | "dream" | "graph"
    auto_inject: bool = True
    granularity: str = "span"  # "span" | "session" | "uri" | "version"
    multi_source: bool = False
```

### 3.4 框架层：ProvenanceManager

一个通用组件，负责溯源的注入、存储、查询：

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

            if source_decl.source_type == "session":
                ref = SourceRef(
                    source_type="session",
                    source_id=context.session_id,
                    detail=f"span:{context.span_start}-{context.span_end}"
                           if source_decl.granularity == "span" else None,
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
        """Merge 时合并溯源列表。"""
        key = schema.provenance.metadata_key

        # 已有的溯源 IDs
        existing_ids = existing_node.metadata.get(key, [])

        # 新的溯源 IDs
        new_ids = getattr(candidate, "provenance_ids", [])

        # 去重合并
        merged = list(dict.fromkeys(existing_ids + new_ids))
        return merged
```

### 3.5 全链路改动点

#### 1. Extraction 层：传递 span 上下文

```python
# extraction/tools.py — _structure_spans
def _structure_spans(self, spans, messages, ...):
    for prepared in prepared_spans:
        # 构建 extraction context（新增）
        extraction_ctx = ExtractionContext(
            session_id=getattr(ctx, "session_id", ""),
            span_start=prepared.span.start,
            span_end=prepared.span.end,
        )
        candidates = self._structure_span_eager(prepared, ...)

        # 注入溯源（新增）
        for candidate in candidates:
            self._provenance_mgr.inject_provenance(
                candidate, schema, extraction_ctx
            )
```

#### 2. CandidateMemory：新增 provenance_ids

```python
@dataclass
class CandidateMemory:
    # ... 现有字段 ...
    provenance_ids: list[str] = field(default_factory=list)
```

#### 3. ArchiveBuilder：写入 metadata

```python
# commit/archive_builder.py — build()
# 已有模式：if candidate.when: metadata["when"] = candidate.when
# 新增：
prov_metadata = self._provenance_mgr.build_metadata(candidate, schema, plan)
metadata.update(prov_metadata)
```

#### 4. Merge 策略：合并溯源列表

```python
# commit/merge_policies.py — ProfilePolicy.plan()
def plan(self, candidate, ctx):
    # ... 现有逻辑 ...
    if exists:
        merged_provenance = self._provenance_mgr.merge_provenance(
            existing, candidate, schema
        )
        merged_fields["provenance_ids"] = merged_provenance
```

#### 5. IndexRecord：传递到向量索引

```python
# index/index_record_builder.py — build_index_records()
key = schema.provenance.metadata_key if schema.provenance else "provenance_ids"
if key in node.metadata:
    record.metadata["provenance_ids"] = node.metadata[key]
```

### 3.6 多来源 Merge 示例

```
Turn 1: "我叫 Alice，是后端工程师"
  → extract_profile(occupation)
  → provenance_ids: ["prov:session:s-abc:span:0-0"]

Turn 2: "我现在转去学前端了"
  → extract_profile(occupation) → merge
  → provenance_ids: [
      "prov:session:s-abc:span:0-0",     ← 保留旧来源
      "prov:session:s-abc:span:5-5",     ← 追加新来源
    ]
```

metadata 中存储：
```json
{
  "provenance_ids": [
    "prov:session:s-abc:span:0-0",
    "prov:session:s-abc:span:5-5"
  ],
  "when": null,
  "who": null
}
```

### 3.7 查询与使用

#### 正向查询：记忆 → 来源

```python
def resolve_provenance(self, node: ContextNode) -> list[SourceDetail]:
    """给定一个 context_node，返回所有来源的详细信息。"""
    ids = node.metadata.get("provenance_ids", [])
    results = []
    for pid in ids:
        ref = SourceRef.from_provenance_id(pid)
        if ref.source_type == "session":
            archive = self._archive_store.get_by_session(ref.source_id)
            if ref.detail and ref.detail.startswith("span:"):
                start, end = parse_span(ref.detail)
                messages = archive.messages[start:end+1]
                results.append(SourceDetail(ref, messages))
            else:
                results.append(SourceDetail(ref, archive))
        elif ref.source_type == "memory":
            source_node = self._fs.read_node(ref.source_id)
            results.append(SourceDetail(ref, source_node))
    return results
```

#### 反向查询：来源 → 记忆

对于"这个 session 产生了哪些记忆"这类查询，可以用 SQL JSONB 查询：

```sql
SELECT uri, metadata->>'provenance_ids' as prov_ids
FROM context_nodes
WHERE metadata->>'provenance_ids' LIKE '%prov:session:s-abc:%';
```

如果性能不够，后续可以加 GIN 索引：

```sql
CREATE INDEX idx_prov_ids ON context_nodes
USING GIN (metadata jsonb_path_ops);
```

或者抽取为独立的 provenance 表（方案 B 的思路，作为阶段二升级）。

---

## 4. 扩展到其他组件

### 4.1 Dreaming（规划中）

Dreaming 的输出（强化后的记忆）需要溯源到参与 dream 的 context_nodes：

```yaml
# dream_schema.yaml (概念示例)
provenance:
  enabled: true
  sources:
    - source_type: "memory"
      auto_inject: true
      granularity: "uri"
      multi_source: true            # 多个输入记忆
  metadata_key: "provenance_ids"
```

```python
# dream 输出时
dream_output.provenance_ids = [
    f"prov:memory:{input_node.uri}:v{input_node.version}"
    for input_node in dream_inputs
]
```

### 4.2 Graph DB（规划中）

图的边需要溯源到产生该关系的抽取操作：

```python
edge.provenance_ids = [
    f"prov:memory:{source_node.uri}:v{source_node.version}",
    f"prov:session:{session_id}:span:{span_start}-{span_end}",
]
```

### 4.3 Vector Index

IndexRecord 已经携带 metadata，溯源信息自然传递：

```python
record.metadata["provenance_ids"] = node.metadata.get("provenance_ids", [])
```

检索时 compose API 返回的结果中直接包含来源信息。

---

## 5. 阶段规划

### Phase 1：Extraction 溯源（当前可实现）

改动范围：
1. `extraction/schemas/models.py` — 新增 `ProvenanceConfig`、`ProvenanceSource`
2. `extraction/schemas/registry.py` — 解析 YAML `provenance:` 块
3. `extraction/schemas/definitions/*.yaml` — 各 schema 加 `provenance:` 声明
4. `extraction/tools.py` — 传递 span 上下文，调用 ProvenanceManager
5. `core/models.py` — CandidateMemory 新增 `provenance_ids`
6. `commit/archive_builder.py` — 写入 metadata
7. `commit/merge_policies.py` — merge 时合并 provenance_ids
8. `index/index_record_builder.py` — 传递到向量索引

核心新增组件：`ProvenanceManager`（约 60 行）

### Phase 2：查询 API

新增 `/api/v1/provenance/{uri}` 端点：
- 正向查询：给定记忆 URI，返回来源详情
- 反向查询：给定 session_id，返回所有从中产生的记忆

### Phase 3：Dreaming + Graph 溯源

Dreaming 和 Graph 构建时复用同一套 ProvenanceManager + SourceRef 体系。

### Phase 4（可选）：独立 Provenance 表

如果 metadata JSONB 查询性能不足，抽取为独立表：

```sql
CREATE TABLE provenance_edges (
    id             UUID PRIMARY KEY,
    target_uri     TEXT NOT NULL,       -- 被溯源的对象
    provenance_id  TEXT NOT NULL,       -- "prov:session:s-abc:span:0-2"
    source_type    TEXT NOT NULL,       -- "session" / "memory" / "dream"
    source_ref     JSONB,              -- SourceRef 完整信息
    created_at     TIMESTAMP
);

CREATE INDEX idx_prov_target ON provenance_edges(target_uri);
CREATE INDEX idx_prov_source ON provenance_edges(provenance_id);
CREATE INDEX idx_prov_type ON provenance_edges(source_type);
```

这是从 metadata JSONB 到专用表的**平滑迁移**——schema 声明和 ProvenanceManager 不需要改，只改存储层。

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
   (Phase 1)         (Phase 3)       (Phase 3)
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
1. **Schema 声明，框架执行** — 溯源需求在 YAML 中声明，代码处理通用逻辑
2. **Provenance ID 全局唯一** — `prov:{type}:{id}:{detail}` 跨组件可用
3. **多来源列表** — merge 追加而非覆盖，保留完整溯源链
4. **渐进式架构** — Phase 1 用 metadata JSONB，Phase 4 可平滑迁移到独立表
