# oG-Memory Deep Dream Framework

> 版本: v3.0-implemented | 日期: 2026-06-01 | 状态: 已实现（dev_0530 分支）

## 架构总览

Deep Dream 是 oG-Memory 的记忆整合与优化机制（对应 OpenClaw 的 Deep Phase），通过 ReAct Loop 让 LLM 自主组合 acquire → read → process_promotion 工具，从大量具体记忆中提炼高层次洞察。

**触发路径**: `POST /api/v1/deepdream` → `MemoryService.deepdream()` → `DeepDreamReActLoop.run()`

**产出与存储**: LLM 通过 tool call 调用 `process_promotion` → `DreamOutput` → loop 结束后转换为 `CandidateMemory` → `WriteAPI` → commit pipeline → `AGFS` 存储（`add_only` 模式，每条 dream 是独立节点）。

Deep Dream 处理的记忆量可能很大，且处理路径取决于中间结果——例如 acquire 100 条后发现太多，先 process 压缩再 acquire 补充；提取一个话题后搜索相似话题再次提取。这种迭代决策适合用 ReAct Loop 让 LLM 自主决定下一步。

**配置 = 工具列表**: 按可用工具配置，LLM 按工具前缀语义（acquire_*, read, process_*）理解调用顺序。更换 acquire 策略只需替换工具函数，更换 process 策略只需替换 YAML 文件，不需要改动 ReAct Loop 本身。

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     Deep Dream ReAct Loop                                │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Operational tools（确定性函数，返回真实结果）                             │
│  ├── acquire_recent ──→ [{"uri": "...", "abstract": "..."}, ...]         │
│  ├── acquire_search ──→ 搜索结果列表                                     │
│  └── read            ──→ 记忆节点完整内容                                 │
│                                                                           │
│  Prompt-driven tool（YAML schema + prompt + validator，LLM 填表 → 反馈） │
│  └── process_promotion(topic, abstract, content, confidence,             │
│                        provenance_uris)                                   │
│      → validator: check_dream_duplicate                                  │
│      → {"status": "valid"} 或                                            │
│      → {"status": "duplicate_found", "duplicate_uri": "...",             │
│         "suggestion": "已有相同话题的 dream 记忆"}                        │
│                                                                           │
│  LLM 的迭代决策流程:                                                      │
│                                                                           │
│  ┌ acquire_recent(100) ──→ 看到记忆列表                                  │
│  │ read(uri=X) ─────────→ 阅读 coding_style 的内容                       │
│  │ acquire_search("coding") ─→ 搜索更多相关记忆                          │
│  │ process_promotion("coding_style", ...)                                │
│  │   ──→ "duplicate_found, 已有 dream 记忆"                              │
│  │ acquire_search("rust") ──→ 搜索另一个话题                             │
│  │ process_promotion("rust", ...)                                        │
│  │   ──→ "valid, 无重复"                                                 │
│  └ (loop 结束 → DreamOutput → CandidateMemory → WriteAPI → AGFS)       │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 两层 Schema 设计

Deep Dream 的 YAML 定义分为两层，各服务于不同的执行路径：

| 层           | 目录                     | 职责                          | Registry              | 使用者                        |
|-------------|------------------------|-------------------------------|----------------------|-----------------------------|
| **存储层**    | `deepdream/schemas/`   | 定义 `dream` memory type 的存储格式 | `SchemaRegistry`     | `URIResolver`, `DreamPolicy` |
| **LLM 工具层** | `deepdream/tool_specs/` | 定义 prompt-driven tool 的参数和指导 | `PromptDrivenSchemaRegistry` | `DeepDreamReActLoop`         |

### dream.yaml（存储层）

定义 `dream` memory type，用于 URI 解析和写入策略：

```yaml
memory_type: dream
directory: "ctx://{{ account_id }}/users/{{ user_id }}/memories/dream"
filename_template: "{{ routing_key }}"
operation_mode: add_only   # 每条 dream 是独特记录，不合并
owner_scope: user
fields:
  routing_key: {type: string, required: true}
  abstract: {type: string, required: true}
  content: {type: string, required: true}
  confidence: {type: number, required: true, description: "置信度（0.0-1.0）"}
  provenance_ids: {type: array, required: true}
```

注意：不再有 `overview` 字段（dream 记忆的 `overview = abstract`），`importance` 已改为 `confidence`（0.0-1.0）。

### promotion.yaml（LLM 工具层）

定义 LLM 可调用的 `process_promotion` 工具：

```yaml
tool_name: process_promotion
description: "从记忆中提取反复出现的话题，生成晋升输出"
category: dream
dream_type: promotion

fields:
  topic: {type: string, required: true, description: "反复出现的话题名称"}
  abstract: {type: string, required: true, description: "≤200 字符摘要"}
  content: {type: string, required: true, description: "完整洞察内容"}
  confidence: {type: number, required: true, description: "置信度（0.0-1.0）"}
  provenance_uris: {type: array, required: true, description: "来源记忆 URI 列表"}

prompt_guidance: |
  ## 如何提取话题
  当你阅读了大量记忆后，寻找反复出现的话题...

post_call_validator: check_dream_duplicate
```

---

## 三类工具的本质区别

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

**为什么 extraction 用 intent marker 而 Deep Dream 不能用**：
Extraction 是一次性流程——读完对话→提取→结束。Deep Dream 是迭代流程——LLM 需要看到 process_promotion 的校验反馈来决定下一步。如果返回 `{"status": "recorded"}`，LLM 无法迭代决策。

---

## DreamOutput 数据结构

```python
@dataclass
class DreamOutput:
    content: str               # 生成的反思/摘要内容（完整）
    abstract: str              # ≤200 字符摘要（用于索引）
    dream_type: str            # promotion | reflection | cluster_summary | ...
    confidence: float          # 置信度（0.0-1.0）
    provenance_ids: list[str]  # 溯源链：来源记忆 URI + 继承的 provenance_ids
    action: str = "create"     # 当前实现只支持 create
    topic: str | None = None   # 话题名称（promotion 专属）
    extra_metadata: dict = field(default_factory=dict)

    def to_candidate_memory(self) -> CandidateMemory:
        routing_key = f"{self.dream_type}_{self.topic}" if self.topic else self.dream_type
        return CandidateMemory(
            category="dream", owner_scope="user", routing_key=routing_key,
            abstract=self.abstract, overview=self.abstract,  # overview = abstract
            content=self.content, confidence=self.confidence,
            provenance_ids=self.provenance_ids, tool_stats=self.extra_metadata,
        )
```

**关键设计决策**：
- **`overview = abstract`**: dream 记忆不需要独立的 overview 层，`CandidateMemory.overview` 直接设为 `abstract`
- **`confidence` 替代 `importance`**: 与 `CandidateMemory` 字段名统一，范围改为 0.0-1.0
- **无 `overview` 字段**: `DreamOutput` 不再有 `overview` 属性，避免与 `abstract` 冗余

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
    "confidence": 0.85,
    "created_at": "2026-05-20T15:30:00Z",
    "processor": "process_promotion",
    "acquire_strategy": "acquire_recent"
}
```

---

## DeepDreamReActLoop 实现

```python
class DeepDreamReActLoop(ReactLoop):
    def __init__(self, llm, fs, prompt_driven_registry, uri_resolver,
                 prompt_manager, vector_index, embedder,
                 operational_tools=None, max_iterations=10, timeout_seconds=120.0):
        # 注意: 没有 dream_registry 参数（已移除——URIResolver 通过 extra_registries 使用它）
        ...

    def _runtime_injection_values(self, ctx):
        return {"ctx": ctx, "context_fs": self._fs,
                "vector_index": self._vector_index, "embedder": self._embedder}

    def _get_custom_tools(self):
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
            result = validator(input, ctx=ctx, context_fs=self._fs,
                              uri_resolver=self._uri_resolver)
            # 3. 无论 status 是 valid 还是 duplicate_found，都收集 DreamOutput
            dream_output = self._tool_input_to_dream_output(input, spec)
            self._dream_outputs.append(dream_output)
            return result
        return {"error": f"Unknown tool: {name}"}

    def _parse_output(self, content) -> None:
        # Deep Dream 通过 tool calls 产出，不走 content 解析
        return None

    def _on_invalid_response(self, content, iteration, max_iterations) -> bool:
        if self._dream_outputs:
            # 有收集到的 DreamOutput → 视为完成信号 → 结束 loop
            return True
        # 无 DreamOutput → 继续迭代，保持 tools enabled
        return False
```

**`_on_invalid_response` 的作用**（Bug 2 修复）：
ReactLoop 基类在 `_parse_output()` 返回 None 时，会进入 invalid response 处理路径——默认行为是禁用 tools、只发 system prompt 让 LLM 重新回答。这对 Deep Dream 是灾难性的：LLM 只能通过 tool calls 产出，禁用 tools = LLM 被困住反复迭代无法产出。

覆盖 `_on_invalid_response` 的逻辑：
- 有 DreamOutput → LLM 内容输出是"我完成了"的信号 → 结束 loop
- 无 DreamOutput → LLM 可能是闲聊/思考 → 继续迭代，保持 tools enabled

---

## Bug 修复记录

### Bug 1: check_dream_duplicate 跨 registry 查找失败（SEVERE）

**问题**: `validators.py` 用 `uri_resolver._registry.get("dream")` 查找 dream schema，这只检查 primary registry。但在生产环境中，dream schema 在 `extra_registries` 里，primary 是 extraction——导致 duplicate 检测被静默禁用。

**修复**: 改用 `uri_resolver.get_directory_uri("dream")`，它调用 `_find_schema()` 搜索所有 registries（primary + extra）。当 schema 不存在时 catch ValueError，返回 `{"status": "valid"}`。

```python
# 修复前（只在 primary registry 查找，永远找不到）
dream_dir = uri_resolver._registry.get("dream")

# 修复后（跨 registry 查找）
try:
    dream_dir = uri_resolver.get_directory_uri("dream", ctx=ctx)
except ValueError:
    return {"status": "valid", "suggestion": "Dream schema not found. Proceed."}
```

### Bug 2: _parse_output 返回 None 导致 tools 被禁用（MEDIUM）

**问题**: `DeepDreamReActLoop._parse_output()` 返回 None（Deep Dream 通过 tool calls 产出，不走 content 解析）。ReactLoop 基类将 None 视为 invalid response → 禁用 tools → LLM 无法继续调用工具 → 被困在无意义的迭代中。

**修复**: 覆盖 `_on_invalid_response()`，在有 DreamOutput 时结束 loop，无 DreamOutput 时继续迭代并保持 tools enabled。

### Bug 3: overview 字段冗余导致 LLM 无法填写

**问题**: `dream.yaml` 要求 `overview` 字段，但 `promotion.yaml`（LLM 工具层）不提供 `overview` 输入——LLM 无法填写它不认识的字段。

**修复**: 从 `dream.yaml` 移除 `overview`，在 `DreamOutput.to_candidate_memory()` 中设 `overview = abstract`。

### Bug 4: importance vs confidence 字段名不一致

**问题**: YAML 定义用 `importance`，但 `CandidateMemory` 用 `confidence`——LLM 输入的字段名与最终存储的字段名不匹配。

**修复**: YAML 和 `DreamOutput` 统一改为 `confidence`（0.0-1.0 范围）。

---

## URI 解析与跨 Registry

Deep Dream 的 schema 存储在独立 registry 中（`SchemaRegistry("deepdream/schemas")`），与 extraction 的 primary registry 并列。`URIResolver` 通过 `extra_registries` 参数支持跨 registry 查找：

```python
# MemoryService.deepdream() 中的 wiring
uri_resolver = URIResolver(extraction_registry, extra_registries=[dream_registry])

# URIResolver._find_schema() 搜索路径：
# 1. primary registry (extraction schemas)
# 2. extra_registries[0] (dream schemas)
# → "dream" 类型在 extra_registries 中找到
```

---

## 目录结构（已实现）

```
oG-Memory/deepdream/
├── __init__.py              # 导出 DeepDreamReActLoop, DreamOutput
├── deep_dream_react_loop.py # DeepDreamReActLoop(ReactLoop)
├── models.py                # DreamOutput, DeepDreamTaskContext, DeepDreamReActResult
├── operational_tools.py     # @operational_tool（acquire_recent, acquire_search）
├── validators.py            # check_dream_duplicate
├── schemas/                 # 存储层 YAML（SchemaRegistry 格式）
│   ├── dream.yaml           # dream memory type schema
│   └── __init__.py
├── tool_specs/              # LLM 工具层 YAML（PromptDrivenSchemaRegistry 格式）
│   ├── promotion.yaml       # process_promotion tool spec
│   └── __init__.py
└── prompts/                 # prompt 模板
│   ├── deepdream.yaml       # system_prompt, identity_block, output_instruction
│   └── __init__.py
```

---

## HTTP 端点

### POST /api/v1/deepdream

**Request**:
```json
{
    "config": {
        "operational_tools": ["acquire_recent", "acquire_search", "read"],
        "prompt_driven_tools": ["process_promotion"]
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
        "iterations": 6,
        "tools_used": ["acquire_recent", "read", "acquire_search", "process_promotion"],
        "created_uris": [
            "ctx://acme/users/alice/memories/dream/promotion_coding_style/"
        ],
        "duration_ms": 15234
    }
}
```

---

## Commit 历史（dev_0530）

17 个实现 commit + 2 个修复/重构 commit：

- **实现**: feat: add PromptDrivenToolSpec → models → YAML schemas → operational tools → validators → DeepDreamReActLoop + prompt → URIResolver/validation/policies → HTTP endpoint + service wiring
- **`d1829b75`**: fix: resolve two Deep Dream bugs (cross-registry lookup + _parse_output behavior)
- **`4ec24d25`**: refactor: split deepdream YAML dirs, unify field names, remove dead code

---

## 已识别但未修复的问题

1. **SchemaField vs field dict**: extraction 用 SchemaField dataclass，deepdream 用 plain YAML dict，两者都进 `build_input_schema_from_fields()`。潜在统一重构。
2. **operational_tools 缺少单元测试**: acquire_recent, acquire_search 无测试。
3. **DreamPolicy 硬编码**: 按 category name 选择行为，而非 schema.operation_mode。

---

## 参考

- [[OpenClaw Dreaming Mechanism]] - OpenClaw Light/REM/Deep 三阶段设计
- [[dreaming-consolidation-validation-survey]] - Dreaming/Consolidation 验证方法论调研
- [[oG-Memory ReactLoop Abstract Base Class]] - ReAct 模式抽象基类设计

---

## 变更日志

| 版本 | 日期 | 变更 |
|------|------|------|
| v3.0-implemented | 2026-06-01 | 反映实际实现：两层 schema 设计、Bug 修复、字段统一、目录重构 |
| v2.3-draft | 2026-05-29 | 引入 prompt-driven tools；新增验证方式章节 |
| v2.2-draft | 2026-05-29 | 三类工具体系初版 |
| v2.1-draft | 2026-05-29 | 文档重构 |
| v2.0-draft | 2026-05-29 | 去掉流水线，直接用 ReAct Loop |
| v1.0-draft | 2026-05-20 | 初版设计 |