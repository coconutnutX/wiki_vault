# oG-Memory Deep Dream Framework 设计文档

> 版本: v2.3-draft | 日期: 2026-05-29 | 状态: 设计阶段

## 架构总览

Deep Dream 是 oG-Memory 的记忆整合与优化机制（对应 OpenClaw 的 Deep Phase），通过 ReAct Loop 让 LLM 自主组合 acquire → process → output 工具，从大量具体记忆中提炼高层次洞察、聚合冗余、建立关联、保留溯源。

Deep Dream 需要处理的记忆量可能很大，且处理路径取决于中间结果——例如 acquire 100 条后发现太多，先 process 压缩再 acquire 补充；提取一个话题后搜索相似话题再次提取，反复多次。这种迭代决策流水线做不到，ReAct Loop 让 LLM 自主决定下一步。

**配置 = 工具列表**: 按可用工具配置，LLM 按工具前缀语义（acquire_*, process_*, output_*）理解调用顺序。

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     Deep Dream ReAct Loop                                │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│   Operational tools（确定性函数，返回真实结果）:                           │
│   ┌─── acquire_recent(limit=100) ──→ 返回 100 条 URI ──────────┐         │
│   └─── read(uri=...) ──→ 返回记忆内容 ─────────────────────────┘         │
│                                                                           │
│   Prompt-driven tools（YAML prompt + schema，返回校验反馈）:             │
│   ┌─── process_promotion(topic=..., ...) ──→                            │
│   │   {"status": "valid", "duplicate_uri": "...", "suggestion": "merge"} │
│   └─── output_create(dream_type=..., ...) ──→                           │
│       {"status": "created", "uri": "ctx://.../dream/promotion_..."}      │
│                                                                           │
│   LLM 的迭代路径:                                                        │
│   1. acquire_recent → 100 条记忆                                         │
│   2. read → 看到 coding_style 反复出现                                    │
│   3. acquire_search(query="coding_style") → 更多相关记忆                  │
│   4. process_promotion → "valid，已有类似 dream 记忆，建议 merge"         │
│   5. acquire_search(query="rust development") → 另一个相关话题            │
│   6. process_promotion → "valid，无重复"                                  │
│   7. output_create → "created at URI ..."                                │
│   8. output_merge → "merged"                                             │
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

模式：**LLM 填表，工具校验+记录，post-loop 解析**。适合一次性流程（对话→提取→结束），不适合需要迭代反馈的场景。

### ReactLoop 基类（已重构）

基类已重构为管理 operational tool dispatch，Deep Dream 只需继承并传入工具列表：
- 构造器接受 `operational_tools` 参数，构建 `_operational_dispatch` dict
- `_get_tools()` 组合 operational specs + subclass `_get_custom_tools()`
- `_execute_tool()` dict lookup + `call_with_injection`，fallback 到 `_execute_custom_tool`
- 3 个 abstract: `_runtime_injection_values()`, `_get_custom_tools()`, `_execute_custom_tool()`

---

## Prompt-driven tools 机制

### YAML 定义（schema + prompt + validator）

Prompt-driven tool 由 YAML 文件定义，包含三个部分：

1. **schema**: 工具参数定义（告诉 LLM 要填哪些字段），格式与 extraction YAML 一致
2. **prompt_guidance**: prompt 指导（告诉 LLM 如何思考），用户可配置
3. **post_call_validator**: 确定性校验函数名（检查去重、校验 URI），不在函数里调 LLM

```yaml
# deepdream/schemas/promotion.yaml
tool_name: process_promotion
description: "从记忆中提取反复出现的话题，生成晋升输出"
category: dream

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

prompt_guidance: |
  ## 如何提取话题

  当你阅读了大量记忆后，寻找反复出现的话题。一个"话题"是：
  - 在多条记忆中都被提及的领域、偏好或模式
  - 例如：如果 5 条记忆都提到 Rust 相关的内容，"rust_development" 就是一个话题

  调用 process_promotion 时：
  - topic: 话题名称（短且具描述性，如 "rust_development"）
  - abstract: 对该话题的简短总结
  - content: 为什么这个话题值得被晋升为 dream 记忆
  - importance: 根据出现频率和深度评估（1.0-3.0）
  - provenance_uris: 你读过的与该话题相关的记忆 URI 列表

  提取后如果发现还有相关话题，可以继续 acquire_search 搜索更多记忆，
  然后再调用 process_promotion 提取新话题。这个过程可以反复进行。

post_call_validator: check_dream_duplicate
```

### 执行流程

LLM 调用 prompt-driven tool 时：

```
LLM 调用 process_promotion(topic="coding_style", ...)
  ↓
_execute_custom_tool() 检测到 prompt-driven tool name
  ↓
1. schema 校验：检查必填字段、类型
  ↓
2. post_call_validator 执行确定性逻辑（不调 LLM）:
   check_dream_duplicate → 在 dream category 下搜索相同 topic
   → 发现已有 dream 记录 ctx://.../dream/promotion_20260515_x1y2
  ↓
3. 返回校验反馈给 LLM:
   {"status": "duplicate_found", 
    "duplicate_uri": "ctx://.../dream/promotion_20260515_x1y2",
    "suggestion": "已有相同话题的 dream 记忆，建议用 output_merge 合并"}
  ↓
LLM 看到反馈，决定下一步:
   → acquire_search 搜索更多 → 再 process_promotion
   → 或 output_merge 合并到已有 dream
```

**关键设计**：prompt_guidance 配置在 YAML 里（不硬编码在代码中），用户可以根据自己的偏好调整——比如改变"话题"的定义、调整重要性评估标准、或增加特定领域的提取指导。

### post_call_validator 示例

```python
# deepdream/validators.py

def check_dream_duplicate(tool_input, ctx, context_fs, uri_resolver):
    """确定性校验：检查是否已有相同 topic 的 dream 记忆"""
    topic = tool_input.get("topic", "")
    # 在 dream category 下搜索相同 topic
    dream_dir = uri_resolver.resolve("dream", {"dream_type": "promotion"}, ctx)
    children = context_fs.list_children(dream_dir, ctx)
    for child_uri in children:
        node = context_fs.read_node(child_uri, ctx)
        if node.metadata.get("topic") == topic:
            return {
                "status": "duplicate_found",
                "duplicate_uri": child_uri,
                "suggestion": "已有相同话题的 dream 记忆，建议用 output_merge 合并",
            }
    return {"status": "valid", "suggestion": "可以继续用 output_create 创建"}
```

validator 是确定性代码（不调 LLM），接受 `tool_input + runtime dependencies`（ctx, context_fs, uri_resolver），返回校验反馈 dict。LLM 看到反馈后自主决定下一步。

### 三类工具的本质区别

oG-Memory 有三类工具，核心区别在于"逻辑在哪执行"和"LLM 是否需要看到返回值"：

|                | Operational tools                      | Extraction intent tools             | Prompt-driven tools                |
| -------------- | -------------------------------------- | ----------------------------------- | ---------------------------------- |
| **逻辑在哪**       | 确定性函数代码                                | prompt + LLM 填表                     | prompt 指导 LLM 思考 + 代码校验            |
| **LLM 看到返回值**  | 是（真实结果）                                | 不需要（一次性流程）                          | 是（校验结果 + 去重提示）                     |
| **定义方式**       | `@operational_tool` 装饰器                | YAML → SchemaRegistry               | YAML schema + YAML prompt_guidance |
| **工具返回值**      | 真实执行结果                                 | `{"status": "recorded"}`            | 校验反馈（valid/duplicate/suggestion）   |
| **解析时机**       | loop 内即时                               | post-loop 批量解析                      | loop 内即时（LLM 据此迭代）                 |
| **prompt 可配置** | 不可（docstring 固定）                       | 可（YAML 模板）                          | 可（YAML prompt_guidance）            |
| **适用场景**       | 查询、读取、搜索                               | 从对话提取信息                             | 迭代式记忆处理                            |
| **dispatch**   | 基类 dict lookup + `call_with_injection` | 子类 `_execute_custom_tool()`         | 子类 `_execute_custom_tool()`        |
| **典型工具**       | read, acquire_recent                   | extract_profile, extract_preference | process_promotion, output_create   |

**为什么 extraction 用 intent marker 而 Deep Dream 不能用**：
Extraction 是一次性流程——读完对话→提取→结束，LLM 不需要看到提取结果来决定下一步。Deep Dream 是迭代流程——LLM 需要看到 process_promotion 的校验反馈（"已有重复话题，建议 merge"）来决定是继续搜还是换个方向。如果返回 `{"status": "recorded"}`，LLM 无法迭代决策。

**为什么 prompt-driven 不是 operational tool**：
`process_promotion` 的核心逻辑（提取话题、评估重要性）是 LLM 思考完成的，不在函数代码里——`@operational_tool` handler 无法实现"让 LLM 思考"。但 prompt-driven 需要确定性校验（去重检查、URI 校验），这部分不在函数里调 LLM，是纯代码逻辑。

---

## 工具清单

### Operational tools（确定性，返回真实结果）

复用 extraction 的 read/list/get_relations/get_access_stats，加上 Deep Dream 专属的 acquire_* 工具：

| 工具名                          | 功能                 | LLM 参数                     | 来源             |
| ---------------------------- | ------------------ | -------------------------- | -------------- |
| `read`                       | 读取记忆节点内容           | `uri`                      | extraction（复用） |
| `list`                       | 列出子节点              | `uri`                      | extraction（复用） |
| `get_relations`              | 获取关联节点             | `uri`                      | extraction（复用） |
| `get_access_stats`           | 获取访问统计             | `uri`                      | extraction（复用） |
| `acquire_recent`             | 获取最近 N 条记忆 URI     | `limit`, `category_filter` | Deep Dream     |
| `acquire_search`             | 搜索相关记忆             | `query`, `top_k`           | Deep Dream     |
| `acquire_light_dream_result` | 从 Light+REM 模块获取候选 | `source`                   | Deep Dream     |

### Prompt-driven tools（YAML prompt + schema + 校验反馈）

#### process_* — 处理记忆

| 工具名                         | 功能           | YAML schema 字段                                                          | validator               | dream_type        |
| --------------------------- | ------------ | ----------------------------------------------------------------------- | ----------------------- | ----------------- |
| `process_promotion`         | 提取反复出现的话题    | `topic`, `abstract`, `content`, `importance`, `provenance_uris`         | `check_dream_duplicate` | `promotion`       |
| `process_reflection`        | 反思机制：对记忆生成洞察 | `insight`, `abstract`, `content`, `importance`, `provenance_uris`       | `check_dream_duplicate` | `reflection`      |
| `process_cluster_summarize` | 聚类后生成类别摘要    | `cluster_label`, `abstract`, `content`, `importance`, `provenance_uris` | `check_dream_duplicate` | `cluster_summary` |

#### output_* — 写入结果

| 工具名              | 功能           | YAML schema 字段                                                       | validator              |
| ---------------- | ------------ | -------------------------------------------------------------------- | ---------------------- |
| `output_create`  | 创建新 dream 记忆 | `dream_type`, `content`, `abstract`, `provenance_uris`, `importance` | `validate_and_create`  |
| `output_merge`   | 合并到已有记忆      | `target_uri`, `content`, `provenance_uris`                           | `validate_and_merge`   |
| `output_archive` | 归档记忆         | `uri`, `reason`                                                      | `validate_and_archive` |

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
    
    由 prompt-driven tools 的返回值和 post-loop 解析生成，
    可转换为 CandidateMemory 或 ContextNode。
    """
    
    # === 内容字段 ===
    content: str           # 生成的反思/摘要内容（完整）
    abstract: str          # ≤200 字符摘要（用于索引）
    overview: str          # 结构化概述（可选，用于快速浏览）
    
    # === 分类字段 ===
    dream_type: str        # promotion | reflection | cluster_summary | ...
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
    "topic": "coding_style",
    "importance": 2.5,
    "created_at": "2026-05-20T15:30:00Z",
    "processor": "process_promotion",
    "acquire_strategy": "acquire_recent"
}
```

---

## MVP 实现

### 最小工具配置

```python
DEEP_DREAM_MVP_OPERATIONAL = [acquire_recent, acquire_search, read]
DEEP_DREAM_MVP_PROMPT_DRIVEN = [process_promotion, output_create]
```

LLM 的典型迭代路径：
```
Iteration 1: acquire_recent(limit=100) → 返回 100 条 URI + 简要信息
Iteration 2: read(uri="ctx://acme/.../pref/coding_style") → 读取几条记忆内容
Iteration 3: acquire_search(query="coding_style") → 搜索更多相关记忆
Iteration 4: process_promotion(topic="coding_style", ...)
              → {"status": "duplicate_found", "duplicate_uri": "ctx://.../dream/promotion_20260515_x1y2",
                 "suggestion": "建议用 output_merge 合并"}
Iteration 5: acquire_search(query="rust development") → 另一个相关话题
Iteration 6: process_promotion(topic="rust", ...)
              → {"status": "valid", "suggestion": "可以继续用 output_create 创建"}
Iteration 7: output_create(dream_type="promotion", ...)
              → {"status": "created", "uri": "ctx://.../dream/promotion_20260520_a1b2c3/"}
Iteration 8: output_merge(target_uri="ctx://.../dream/promotion_20260515_x1y2", ...)
              → {"status": "merged"}
```

### process_promotion 的 YAML 定义

见上方 "Prompt-driven tools 机制" 中的 promotion.yaml 示例。

MVP 版本 prompt_guidance 指导 LLM "提取多次出现的话题"，后续可迭代：
- 从简单话题提取 → 更复杂的反思（生成问题→搜索→回答）
- 从单条晋升 → 聚类摘要（多条相关记忆合并为一条摘要）
- 用户通过修改 YAML prompt_guidance 自定义提取策略

### DeepDreamReActLoop 实现

```python
# deepdream/deep_dream_react_loop.py

class DeepDreamReActLoop(ReactLoop):
    def __init__(self, llm, fs, registry, uri_resolver,
                 operational_tools=None, prompt_driven_tools=None, ...):
        super().__init__(
            llm, max_iterations, timeout_seconds,
            operational_tools=operational_tools or DEEP_DREAM_MVP_OPERATIONAL,
            pipeline_name="deepdream",
        )
        self._fs = fs
        self._registry = registry
        self._uri_resolver = uri_resolver
        # 从 YAML 构建 prompt-driven tool schema + prompt_guidance
        self._prompt_driven_specs = build_prompt_driven_specs(registry)
        # validator dispatch: name → validator function
        self._validator_dispatch = {
            "check_dream_duplicate": check_dream_duplicate,
            "validate_and_create": validate_and_create,
            ...
        }

    def _runtime_injection_values(self, ctx):
        return {"ctx": ctx, "context_fs": self._fs}

    def _get_custom_tools(self):
        # prompt-driven tool schemas（来自 YAML）
        return [spec.tool_dict() for spec in self._prompt_driven_specs]

    def _execute_custom_tool(self, name, input, ctx):
        spec = self._prompt_driven_dispatch.get(name)
        if spec is not None:
            # 1. schema 校验
            errors = spec.validate(input)
            if errors:
                return {"status": "validation_error", "errors": errors}
            # 2. post_call_validator 执行确定性校验
            validator = self._validator_dispatch.get(spec.post_call_validator)
            if validator:
                return validator(input, ctx=ctx, context_fs=self._fs,
                                 uri_resolver=self._uri_resolver)
            return {"status": "valid"}
        return {"error": f"Unknown tool: {name}"}

    def _build_messages(self, ctx, task_ctx):
        # 组装 system prompt: 基础 prompt + 各 prompt-driven tool 的 prompt_guidance
        system_prompt = self._prompt_manager.render("deepdream", "system_prompt")
        for spec in self._prompt_driven_specs:
            system_prompt += f"\n\n{spec.prompt_guidance}"
        ...
```

---

## HTTP 端点

### POST /api/v1/deepdream

触发 Deep Dream 处理。端点内部创建 `DeepDreamReActLoop` 并调用。

**Request**:
```json
{
    "config": {
        "operational_tools": ["acquire_recent", "acquire_search", "read"],
        "prompt_driven_tools": ["process_promotion", "output_create"]
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
        "iterations": 8,
        "tools_used": ["acquire_recent", "read", "acquire_search", 
                        "process_promotion", "output_create", "output_merge"],
        "created_uris": [
            "ctx://acme/users/alice/memories/dream/promotion_20260520_a1b2c3/"
        ],
        "merged_uris": [
            "ctx://acme/users/alice/memories/dream/promotion_20260515_x1y2/"
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

基类已管理 operational tool dispatch。Deep Dream 只需继承并传入 `DEEP_DREAM_MVP_OPERATIONAL`。

### 5. core/operational_tool_spec.py（已实现）

`@operational_tool` 装饰器框架已实现并验证，Deep Dream 的 operational tools 直接复用。

### 6. core/prompt_driven_tool_spec.py（新增）

Prompt-driven tool 的 spec 定义和执行机制，类似 extraction 的 `build_extraction_tools()` 但返回校验反馈而非 `{"status": "recorded"}`：

```python
@dataclass
class PromptDrivenToolSpec:
    name: str
    description: str
    input_schema: dict
    prompt_guidance: str          # YAML prompt_guidance → 注入 system prompt
    post_call_validator: str      # validator 函数名 → dispatch lookup
    category: str                 # dream
    dream_type: str               # promotion / reflection / ...

    def tool_dict(self) -> dict:
        # 同 extraction: {name, description, input_schema}
        ...

    def validate(self, input: dict) -> list[str]:
        # schema 校验：必填字段、类型检查
        ...
```

### 7. extraction/tool_builder.py（可能需扩展）

`build_extraction_tools()` 和 `parse_tool_call()` 目前只服务于 extraction。如果 prompt-driven tool 的 schema 构建机制与 extraction 有共同逻辑，可抽取为 `core/tool_builder.py` 的通用函数。但 prompt-driven tool 的执行流程（返回校验反馈而非 intent marker）与 extraction 不同，可能不需要强行统一。

### 8. deepdream/schemas/（新增）

YAML schema + prompt_guidance 定义目录，格式见上方 promotion.yaml 示例。

### 9. deepdream/validators.py（新增）

post_call_validator 确定性函数，如 `check_dream_duplicate`, `validate_and_create` 等。

---

## 目录结构

```
oG-Memory/
├── deepdream/
│   ├── __init__.py              # 模块入口
│   ├── deep_dream_react_loop.py # DeepDreamReActLoop(ReactLoop)
│   ├── models.py                # DreamOutput, DeepDreamReActResult
│   ├── operational_tools.py     # @operational_tool（acquire_recent 等）
│   ├── validators.py            # post_call_validator 确定性函数
│   ├── schemas/                 # YAML schema + prompt_guidance
│   │   ├── promotion.yaml       # process_promotion
│   │   ├── reflection.yaml      # process_reflection (未来)
│   │   ├── create.yaml          # output_create
│   │   ├── merge.yaml           # output_merge (未来)
│   │   └── archive.yaml         # output_archive (未来)
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
│   ├── prompt_driven_tool_spec.py # 新增: prompt-driven tool spec + 执行机制
│   ├── react_loop.py            # 基类（已重构）
│
├── extraction/
│   ├── operational_tools.py     # extraction 的 @operational_tool（已实现）
│   ├── extraction_react_loop.py # ExtractionReActLoop（已重构）
│   ├── tool_builder.py          # 现有
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
| v2.3-draft | 2026-05-29 | 引入第三类工具 prompt-driven tools（YAML schema + prompt_guidance + post_call_validator），解决迭代场景下 intent marker 断链问题；process_promotion 不再是确定性函数或 intent marker，改为 prompt-driven；新增三类工具本质区别对照表；新增 core/prompt_driven_tool_spec.py 和 deepdream/validators.py；acquire_search 加入工具清单 |
| v2.2-draft | 2026-05-29 | 三类工具体系初版（operational / extraction intent / dream intent） |
| v2.1-draft | 2026-05-29 | 文档重构：按模块依次介绍 |
| v2.0-draft | 2026-05-29 | 去掉流水线，直接用 ReAct Loop |
| v1.0-draft | 2026-05-20 | 初版设计 |