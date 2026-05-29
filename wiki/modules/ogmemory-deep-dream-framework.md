# oG-Memory Deep Dream Framework 设计文档

> 版本: v2.2-draft | 日期: 2026-05-29 | 状态: 设计阶段

## 架构总览

Deep Dream 是 oG-Memory 的记忆整合与优化机制（对应 OpenClaw 的 Deep Phase），通过 ReAct Loop 让 LLM 自主组合 acquire → process → output 工具，从大量具体记忆中提炼高层次洞察、聚合冗余、建立关联、保留溯源。

**配置 = 工具列表**: 不再按阶段组合策略，而是配置可用工具。LLM 按工具前缀语义（acquire_*, process_*, output_*）理解调用顺序。

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     Deep Dream ReAct Loop                                │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│   Operational tools（确定性，真正执行）:                                   │
│   ┌─── acquire_recent(limit=100) ──── 返回 100 条 URI ────┐              │
│   └─── read(uri=...) ──────── 返回记忆内容 ─────────────────┤              │
│                                                               │              │
│   Dream intent tools（LLM 填表，工具校验+记录）:                          │
│   ┌─── process_promotion ── LLM 填入: 话题、摘要、重要性 ──┤              │
│   └─── output_create ─── LLM 填入: dream_type, provenance ─┘              │
│                                                                           │
│   LLM 的典型路径:                                                         │
│   1. acquire_recent → 看到 100 条记忆                                      │
│   2. read → 深入阅读几条                                                    │
│   3. process_promotion → 提取反复出现的话题，填写晋升 schema               │
│   4. output_create → 填写创建 schema                                       │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 依赖模块

### 溯源机制（ProvenanceResolver）

Deep Dream 输出需要完整溯源链——知道每条 dream 记忆从哪些原始记忆提炼而来。现有 `ProvenanceResolver` 已预留 `dream` source_type，无需改动。

Provenance ID 格式：
```
prov:1:{source_type}:{urlencoded(source_id)}:{urlencoded(detail)}

示例:
- prov:1:dream:promotion_20260520_a1b2c3:              (dream 输出本身)
- prov:1:memory:ctx%3A%2F%2Facme%2Fmemories%2Fpref%2Fcoding:  (来源记忆 URI)
- prov:1:archive:20260513_100000_a1b2:msg_a3f8               (继承的 archive 来源)
```

Dream 输出的 `provenance_ids` 包含两层：
1. **直接来源**: 被处理的记忆 URI（通过 `ProvenanceResolver.build_id("memory", uri)` 生成）
2. **继承溯源**: 来源记忆自身的 `provenance_ids`（追溯更早来源）

### @operational_tool 框架（已实现）

`core/operational_tool_spec.py` 提供的装饰器，自动从函数签名 + docstring 生成 OpenAI function-calling schema。关键特性：

- **不包函数**: 只在 `fn._operational_tool_spec` 上挂 spec 对象，原函数身份不变
- **runtime 参数过滤**: `_RUNTIME_TYPES`（RequestContext, ContextFS）类型的参数不进 LLM schema，执行时由 `call_with_injection()` 注入

执行流程：
```
ReactLoop._execute_tool()
→ dict lookup: spec = _operational_dispatch[name]
→ call_with_injection(spec, llm_input, ctx=ctx, context_fs=fs)
→ runtime 参数按 name → type 匹配注入
→ spec.handler(**kwargs) 调用原函数
```

### Extraction intent tools（已实现）

Extraction 的 `extract_*` 工具是 intent marker——LLM 调用它们表达"要提取什么"，工具返回 `{"status": "recorded"}`，loop 结束后 `_to_candidate()` 从工具调用参数里解析出 CandidateMemory。

模式：**LLM 填表，工具校验+记录，post-loop 解析**。

Dream intent tools 复用同一模式，但 schema 和输出类型不同：

| | Extraction intent | Dream intent |
|---|---|---|
| schema 来源 | YAML → SchemaRegistry | YAML → DreamSchemaRegistry（或复用 SchemaRegistry） |
| schema 定义字段 | category, routing_key, owner_scope, abstract... | dream_type, topic, importance, provenance_uris, action |
| 输出类型 | CandidateMemory | DreamOutput → CandidateMemory |
| 校验逻辑 | parse_tool_call + schema.validate | parse_tool_call + dream_schema.validate |

共享机制可抽取为通用的 `parse_intent_tool_call()`，但 extraction 和 dream 的 schema 定义、输出类型各自维护。

### Extraction tools vs Operational tools vs Dream intent tools

oG-Memory 现有三类工具：

| 特性 | Operational tools | Extraction intent tools | Dream intent tools |
|------|-------------------|------------------------|-------------------|
| 定义方式 | `@operational_tool` 装饰器 | YAML → SchemaRegistry | YAML → SchemaRegistry |
| LLM 调用含义 | 真正执行函数逻辑 | 填表表达意图 | 填表表达意图 |
| 工具返回值 | 真实执行结果 | `{"status": "recorded"}` | `{"status": "recorded"}` |
| 解析时机 | loop 内即时 | post-loop 批量解析 | post-loop 批量解析 |
| dispatch | 基类 dict lookup + `call_with_injection` | 子类 `_execute_custom_tool()` | 子类 `_execute_custom_tool()` |
| 输出类型 | dict（直接返回 LLM） | CandidateMemory | DreamOutput → CandidateMemory |

Operational tools 是"调函数"——LLM 调用 `acquire_recent` 真正触发获取逻辑，返回真实数据，LLM 根据返回结果决定下一步。

Intent tools（extraction + dream）是"填表"——LLM 调用 `process_promotion` 填入话题和重要性，工具只校验+记录，loop 结束后统一解析。区别在于 schema 定义和输出类型。

### ReactLoop 基类（已重构）

基类已重构为管理 operational tool dispatch，Deep Dream 只需继承并传入工具列表：
- 构造器接受 `operational_tools` 参数，构建 `_operational_dispatch` dict
- `_get_tools()` 组合 operational specs + subclass `_get_custom_tools()`
- `_execute_tool()` dict lookup + `call_with_injection`，fallback 到 `_execute_custom_tool`
- 3 个 abstract: `_runtime_injection_values()`, `_get_custom_tools()`, `_execute_custom_tool()`

---

## 工具清单

### Operational tools（确定性，真正执行）

复用 extraction 的 read/list/get_relations/get_access_stats，加上 Deep Dream 专属的 acquire_* 工具：

| 工具名 | 功能 | LLM 参数 | 来源 |
|--------|------|----------|------|
| `read` | 读取记忆节点内容 | `uri` | extraction（复用） |
| `list` | 列出子节点 | `uri` | extraction（复用） |
| `get_relations` | 获取关联节点 | `uri` | extraction（复用） |
| `get_access_stats` | 获取访问统计 | `uri` | extraction（复用） |
| `acquire_recent` | 获取最近 N 条记忆 URI | `limit`, `category_filter` | Deep Dream |
| `acquire_light_dream_result` | 从 Light+REM 模块获取候选 | `source` | Deep Dream |

### Dream intent tools（LLM 填表，校验+记录）

按语义前缀分两组。LLM 填入 schema 定义的字段，工具只返回 `{"status": "recorded"}`，post-loop 解析为 DreamOutput。

#### process_* — 处理记忆

| 工具名 | 功能 | schema 字段 | dream_type |
|--------|------|-------------|------------|
| `process_promotion` | 提取反复出现的话题，生成晋升输出 | `topic`, `abstract`, `content`, `importance`, `provenance_uris` | `promotion` |
| `process_reflection` | 反思机制：对记忆生成洞察 | `insight`, `abstract`, `content`, `importance`, `provenance_uris` | `reflection` |
| `process_cluster_summarize` | 聚类后生成类别摘要 | `cluster_label`, `abstract`, `content`, `importance`, `provenance_uris` | `cluster_summary` |

#### output_* — 写入结果

| 工具名 | 功能 | schema 字段 |
|--------|------|-------------|
| `output_create` | 创建新 dream 记忆 | `dream_type`, `content`, `abstract`, `provenance_uris`, `importance` |
| `output_merge` | 合并到已有记忆 | `target_uri`, `content`, `provenance_uris` |
| `output_archive` | 归档记忆 | `uri`, `reason` |

### 关于 process_promotion

process_promotion 不是确定性逻辑（阈值过滤+转换），而是 LLM 填表：
- LLM 先通过 acquire + read 看到大量记忆
- 然后调用 `process_promotion`，填入反复出现的话题、摘要、重要性
- 工具校验+记录，post-loop 解析为 DreamOutput

MVP 版本 prompt 指导 LLM "提取多次出现的话题"，后续可迭代为更复杂的反思、聚类等。

### 工具配置

```python
# operational tools: 确定性函数
DEEP_DREAM_OPERATIONAL_TOOLS = [
    acquire_recent,
    acquire_light_dream_result,
    # 复用 extraction 的
    read, list, get_relations, get_access_stats,
]

# dream intent tools: LLM 填表
DEEP_DREAM_INTENT_TOOLS = [
    process_promotion,
    output_create,
    output_merge,
    output_archive,
]
```

### DeepDreamReActLoop 实现

```python
# deepdream/deep_dream_react_loop.py

class DeepDreamReActLoop(ReactLoop):
    def __init__(self, llm, fs, registry, operational_tools=None, ...):
        super().__init__(
            llm, max_iterations, timeout_seconds,
            operational_tools=operational_tools or DEEP_DREAM_OPERATIONAL_TOOLS,
            pipeline_name="deepdream",
        )
        self._fs = fs
        self._registry = registry
        self._intent_tools = build_intent_tools(registry)  # 类似 build_extraction_tools

    def _runtime_injection_values(self, ctx):
        return {"ctx": ctx, "context_fs": self._fs}

    def _get_custom_tools(self):
        return self._intent_tools  # dream intent tools (process_*, output_*)

    def _execute_custom_tool(self, name, input, ctx):
        if name.startswith("process_") or name.startswith("output_"):
            return {"status": "recorded"}  # intent marker
        return {"error": f"Unknown tool: {name}"}
```

混用 extraction + deep dream 工具：
```python
tools = EXTRACTION_OPERATIONAL_TOOLS + DEEP_DREAM_OPERATIONAL_TOOLS
loop = SomeHybridLoop(llm, fs, operational_tools=tools, ...)
```

---

## 输出格式

### Category: dream

新增 `dream` category，与 `ProvenanceResolver` 预留的 `dream` source_type 对应。

**URI 格式**:
```
ctx://{account}/users/{user}/memories/dream/{dream_id}/
```

**dream_id 格式**:
```
{dream_type}_{timestamp}_{content_hash}

示例:
- promotion_20260520_a1b2c3
- reflection_20260520_c4d5e6
- cluster_summary_20260521_e7f8g9
```

### DreamOutput 数据结构

```python
@dataclass
class DreamOutput:
    """Deep Dream 处理输出
    
    由 dream intent tools 的 post-loop 解析生成，
    可转换为 CandidateMemory 或 ContextNode。
    """
    
    # === 内容字段 ===
    content: str           # 生成的反思/摘要内容（完整）
    abstract: str          # ≤200 字符摘要（用于索引）
    overview: str          # 结构化概述（可选，用于快速浏览）
    
    # === 分类字段 ===
    dream_type: str        # promotion | reflection | cluster_summary | compression | ...
    importance: float      # 重要性权重（dream 输出通常 > 1.0）
    
    # === 溯源字段 ===
    provenance_ids: list[str]   # 溯源链：来源记忆 URI + 继承的 provenance_ids
    
    # === 写入动作 ===
    action: str            # CREATE | MERGE | ARCHIVE | DELETE
    
    # === 额外元数据 ===
    extra_metadata: dict   # 扩展字段
```

### metadata 存储结构

ContextNode.metadata 字段：

```json
{
    "provenance_ids": [
        "prov:1:memory:ctx://acme/memories/pref/coding_style:",
        "prov:1:memory:ctx://acme/memories/entity/rust_project:",
        "prov:1:archive:20260513_100000_a1b2:msg_a3f8"
    ],
    "dream_type": "promotion",
    "importance": 1.5,
    "created_at": "2026-05-20T15:30:00Z",
    "processor": "process_promotion",
    "acquire_strategy": "acquire_recent"
}
```

---

## MVP 实现

### 最小工具配置

```python
DEEP_DREAM_MVP_OPERATIONAL = [acquire_recent, read, list]
DEEP_DREAM_MVP_INTENT = [process_promotion, output_create]
```

LLM 的典型 ReAct 路径：
```
Iteration 1: acquire_recent(limit=100) → 返回 100 条 URI + 简要信息
Iteration 2: read(uri="ctx://acme/.../pref/coding_style") → 读取几条记忆内容
Iteration 3: process_promotion(topic="coding_style", abstract="...", provenance_uris=[...])
              → {"status": "recorded"}（intent marker，post-loop 解析）
Iteration 4: output_create(dream_type="promotion", content="...", provenance_uris=[...])
              → {"status": "recorded"}（intent marker）
Iteration 5: (content) → 输出最终 JSON，loop 结束后统一解析所有 intent
```

### process_promotion 的 schema 定义

YAML 定义（类似 extraction 的 YAML schema）：

```yaml
# deepdream/schemas/promotion.yaml
category: dream
dream_type: promotion
tool_name: process_promotion
description: "从记忆中提取反复出现的话题，生成晋升输出"

fields:
  topic:
    type: string
    required: true
    description: "反复出现的话题名称"
  abstract:
    type: string
    required: true
    description: "≤200 字符摘要"
  content:
    type: string
    required: true
    description: "关于该话题的完整洞察内容"
  importance:
    type: number
    required: true
    description: "重要性权重（1.0-3.0）"
  provenance_uris:
    type: array
    required: true
    description: "来源记忆 URI 列表"
```

### process_promotion 的 prompt 指导

System prompt 中包含指导，告诉 LLM 如何从记忆中提取话题：

```
## process_promotion 使用指南

当你阅读了大量记忆后，寻找反复出现的话题。一个"话题"是：
- 在多条记忆中都被提及的领域、偏好或模式
- 例如：如果 5 条记忆都提到 Rust 相关的内容，"rust_development" 就是一个话题

调用 process_promotion 时：
- topic: 话题名称（短且具描述性，如 "rust_development"）
- abstract: 对该话题的简短总结
- content: 为什么这个话题值得被晋升为 dream 记忆
- importance: 根据出现频率和深度评估（1.0-3.0）
- provenance_uris: 你读过的与该话题相关的记忆 URI 列表
```

后续迭代方向：
- 从简单话题提取 → 更复杂的反思（生成问题→搜索→回答）
- 从单条晋升 → 聚类摘要（多条相关记忆合并为一条摘要）
- 从固定 prompt → 可配置的 prompt 模板

### Post-loop 解析

类似 extraction 的 `_to_extraction_react_result()`，Deep Dream loop 结束后从 tools_used 和 output JSON 中解析 dream intent：

```python
def _to_dream_react_result(self, loop_result, ctx):
    # 从 output JSON 解析
    dream_outputs = [item for item in loop_result.output if isinstance(item, DreamOutput)]
    
    # 从 tools_used 解析 intent markers
    for record in loop_result.tools_used:
        if record.get("result", {}).get("status") == "recorded":
            tool_name = record["tool_name"]
            tool_input = record["params"]
            output = self._parse_intent_call(tool_name, tool_input)
            if output:
                dream_outputs.append(output)
    
    # 转换为 CandidateMemory
    candidates = [self._to_candidate(d) for d in dream_outputs]
    
    return DeepDreamReActResult(candidates=candidates, ...)
```

---

## HTTP 端点

### POST /api/v1/deepdream

触发 Deep Dream 处理。端点内部创建 `DeepDreamReActLoop` 并调用。

**Request**:
```json
{
    "config": {
        "operational_tools": ["acquire_recent", "read"],
        "intent_tools": ["process_promotion", "output_create"]
    },
    "context": {
        "user_id": "alice",
        "agent_id": "assistant",
        "session_id": "session_001"
    }
}
```

**Response**:
```json
{
    "status": "success",
    "report": {
        "iterations": 5,
        "tools_used": ["acquire_recent", "read", "process_promotion", "output_create"],
        "created_uris": [
            "ctx://acme/users/alice/memories/dream/promotion_20260520_a1b2c3/",
            "ctx://acme/users/alice/memories/dream/promotion_20260520_c4d5e6/"
        ],
        "duration_ms": 15234
    }
}
```

---

## 需修改的现有代码

### 1. core/validation.py

```python
valid_categories = [
    "profile", "preference", "entity", "event",
    "case", "pattern", "skill", "tool", "dream"  # 新增
]
```

### 2. core/uri_resolver.py

支持 dream category 的 URI 解析。格式: `ctx://{account}/users/{user}/memories/dream/{dream_id}/`

### 3. commit/merge_policies.py

新增 `DreamPolicy`：dream 类型总是 CREATE，不合并。URI 由 dream_id 生成确保唯一性。

### 4. core/react_loop.py（已重构）

基类已管理 operational tool dispatch。Deep Dream 只需继承并传入 `DEEP_DREAM_OPERATIONAL_TOOLS`。

### 5. core/operational_tool_spec.py（已实现）

`@operational_tool` 装饰器框架已实现并验证，Deep Dream 直接复用。

### 6. extraction/tool_builder.py（需扩展）

`build_extraction_tools()` 和 `parse_tool_call()` 目前只服务于 extraction schemas。需要抽取通用机制支持 dream schemas：

- `build_intent_tools(registry)` — 从 SchemaRegistry 生成 intent tool 定义（extraction 和 dream 共用）
- `parse_intent_tool_call(name, input, registry)` — 通用 intent 解析（返回类型由 schema 决定）

### 7. deepdream/schemas/（新增）

YAML schema 定义，格式与 extraction schemas 一致，注册到 SchemaRegistry 或独立的 DreamSchemaRegistry。

---

## 目录结构

```
oG-Memory/
├── deepdream/
│   ├── __init__.py              # 模块入口
│   ├── deep_dream_react_loop.py # DeepDreamReActLoop(ReactLoop)
│   ├── models.py                # DreamOutput, DeepDreamReActResult
│   ├── operational_tools.py     # @operational_tool（acquire_recent 等）
│   ├── schemas/                 # YAML schema 定义
│   │   ├── promotion.yaml       # process_promotion schema
│   │   ├── reflection.yaml      # process_reflection schema (未来)
│   │   ├── create.yaml          # output_create schema
│   │   ├── merge.yaml           # output_merge schema (未来)
│   │   └── archive.yaml         # output_archive schema (未来)
│   └── config.py                # 工具列表配置（MVP / 丰富）
│
├── server/
│   └── app.py                   # 新增 POST /api/v1/deepdream 端点
│
├── core/
│   ├── validation.py            # 修改: valid_categories 添加 "dream"
│   ├── uri_resolver.py          # 修改: 支持 dream URI
│   ├── provenance_resolver.py   # 无修改（已支持 dream）
│   ├── operational_tool_spec.py # @operational_tool 框架（已实现）
│   ├── react_loop.py            # 基类（已重构）
│   ├── tool_builder.py          # 扩展: 抽取通用 intent tool 机制
│
├── extraction/
│   ├── operational_tools.py     # extraction 的 @operational_tool（已实现）
│   ├── extraction_react_loop.py # ExtractionReActLoop（已重构）
│   ├── tool_builder.py          # 现有，需抽取通用部分
│   ├── schemas/                 # extraction YAML schemas（不动）
│
├── commit/
│   └── merge_policies.py        # 新增: DreamPolicy
```

---

## 参考

- [[OpenClaw Dreaming Mechanism]] - OpenClaw Light/REM/Deep 三阶段设计
- [[ogmemory-architecture]] - oG-Memory 整体架构
- [[ogmemory-operational-tool-framework]] - @operational_tool 装饰器框架设计
- [[agentic-memory-evaluation-survey]] - Agent 记忆调研报告

---

## 变更日志

| 版本 | 日期 | 变更 |
|------|------|------|
| v2.2-draft | 2026-05-29 | 引入三类工具体系：operational / extraction intent / dream intent；process_promotion 从确定性函数改为 dream intent tool（LLM 填表：提取反复出现的话题）；tool_builder 需抽取通用 intent 机制；新增 deepdream/schemas/ YAML 定义 |
| v2.1-draft | 2026-05-29 | 文档重构：按模块依次介绍 |
| v2.0-draft | 2026-05-29 | 去掉流水线，直接用 ReAct Loop；策略改为 @operational_tool 工具函数 |
| v1.5-draft | 2026-05-29 | 工具定义改为 @operational_tool 框架 |
| v1.0-draft | 2026-05-20 | 初版设计 |