
  

## Context

  

Deep Dream 是 oG-Memory 的记忆整合与优化机制，通过 ReAct Loop 让 LLM 自主组合 acquire → process → output 工具，从大量具体记忆中提炼高层次洞察。@operational_tool 框架和 ReactLoop 基类已完成重构，Deep Dream 是基于这些基础设施的新模块。

  

设计文档: `/mnt/c/Data/wiki-vault/wiki/modules/ogmemory-deep-dream-framework.md` (v2.3-draft)

  

关键决策:

- **deepdream/schemas/ 独立目录** — Deep Dream 有自己的 SchemaRegistry 和 PromptDrivenSchemaRegistry

- **MVP 最小集** — acquire_recent, acquire_search, read, process_promotion（不再需要 output_create）

- **三类工具**: Operational (确定性函数), Extraction intent (已实现), Prompt-driven (YAML schema + prompt_guidance + post_call_validator)

- **写入路径**: process_promotion 输出结构化内容，validator 返回校验反馈，loop 结束后 DreamOutput → CandidateMemory → WriteAPI → commit pipeline 写入。不需要单独的 output_create 工具。

  

---

  

## Commit Plan (8 commits, ordered by dependency)

  

### Commit 1: `feat: add PromptDrivenToolSpec framework`

  

新增 prompt-driven tool 的 spec 类和 YAML 加载机制，类似 OperationalToolSpec 的地位但服务于不同的执行模型（校验反馈而非即时执行）。

  

**新增文件**:

- `core/prompt_driven_tool_spec.py` — PromptDrivenToolSpec dataclass + PromptDrivenSchemaRegistry + build_prompt_driven_tools()

- `tests/unit/core/test_prompt_driven_tool_spec.py` — 单元测试

  

**关键设计**:

```python

@dataclass

class PromptDrivenToolSpec:

    name: str                    # e.g. "process_promotion"

    description: str             # Tool description for LLM

    input_schema: dict           # JSON Schema (LLM 参数)

    prompt_guidance: str         # YAML prompt_guidance → 注入 system prompt

    post_call_validator: str     # validator 函数名 → dispatch lookup

    category: str                # "dream"

    dream_type: str              # "promotion" / etc.

  

    def tool_dict(self) -> dict   # 同 OperationalToolSpec 格式

    def validate(input) → list[str]  # schema 校验：必填字段、类型

  

class PromptDrivenSchemaRegistry:

    # 从 YAML 目录加载 PromptDrivenToolSpec

    # 格式: tool_name, description, category, fields, prompt_guidance, post_call_validator

    __init__(schemas_dir) → self._specs: dict[str, PromptDrivenToolSpec]

    get(tool_name) → PromptDrivenToolSpec | None

    list_all() → list[PromptDrivenToolSpec]

  

def build_prompt_driven_tools(registry) → list[dict]  # OpenAI tool definitions

```

  

---

  

### Commit 2: `feat: add Deep Dream models`

  

Deep Dream 的数据结构，用于 loop 内收集结果和最终输出。

  

**新增文件**:

- `deepdream/__init__.py` — 导出 DeepDreamReActLoop, DreamOutput

- `deepdream/models.py` — DreamOutput, DeepDreamTaskContext, DeepDreamReActResult

  

**关键类**:

```python

@dataclass

class DreamOutput:

    content, abstract, overview, dream_type, importance, provenance_ids, action

    topic: str | None = None

    extra_metadata: dict = field(default_factory=dict)

    def to_candidate_memory() → CandidateMemory  # routing_key = "{dream_type}_{topic}"

  

@dataclass

class DeepDreamTaskContext(ReactTaskContext):

    dream_config: dict = field(default_factory=dict)

  

@dataclass

class DeepDreamReActResult:

    dream_outputs, candidates, tools_used, iterations, read_uris, trace, completed

```

  

---

  

### Commit 3: `feat: add Deep Dream YAML schemas`

  

Dream category 的 memory type schema + prompt-driven tool 的 YAML 定义。

  

**新增文件**:

- `deepdream/schemas/__init__.py`

- `deepdream/schemas/definitions/dream.yaml` — memory type schema (SchemaRegistry 格式)

- `deepdream/schemas/definitions/promotion.yaml` — prompt-driven tool (新格式)

  

**dream.yaml** (memory type schema):

```yaml

memory_type: dream

directory: "ctx://{{ account_id }}/users/{{ user_id }}/memories/dream"

filename_template: "{{ routing_key }}"

operation_mode: add_only  # 每条 dream 是独特记录，不合并

owner_scope: user

fields: routing_key, abstract, overview, content, importance, provenance_ids

```

  

**promotion.yaml** (prompt-driven tool):

```yaml

tool_name: process_promotion

category: dream

dream_type: promotion

fields: topic, abstract, content, importance, provenance_uris

prompt_guidance: | (指导 LLM 如何提取话题，包含 provenance_uris 填写说明)

post_call_validator: check_dream_duplicate

```

  

注意: 不再需要 `create.yaml`（output_create）。process_promotion 输出的结构化内容在 loop 结束后走 commit pipeline（DreamOutput → CandidateMemory → WriteAPI → write_node）。

  

---

  

### Commit 4: `feat: add Deep Dream operational tools`

  

Deep Dream 专属的 acquire_* @operational_tool 函数 + 工具列表配置。

  

**新增文件**:

- `deepdream/operational_tools.py` — acquire_recent, acquire_search, DEEP_DREAM_MVP_OPERATIONAL

  

**修改文件**:

- `core/operational_tool_spec.py` — `_RUNTIME_TYPES` 添加 VectorIndex, Embedder

  

**关键设计**:

```python

@operational_tool

def acquire_recent(limit: int = 100, category_filter: list[str] | None = None,

                   ctx: RequestContext, context_fs: ContextFS) -> dict:

    """获取最近 N 条记忆 URI。"""

  

@operational_tool

def acquire_search(query: str, top_k: int = 10,

                   ctx: RequestContext, context_fs: ContextFS,

                   vector_index: VectorIndex, embedder: Embedder) -> dict:

    """搜索相关记忆。"""

  

DEEP_DREAM_MVP_OPERATIONAL = [acquire_recent, acquire_search, read]

# read 从 extraction.operational_tools 导入复用

```

  

**_RUNTIME_TYPES 扩展**:

```python

_RUNTIME_TYPES: set[type] = {RequestContext, ContextFS, VectorIndex, Embedder}

```

向后兼容：现有 operational tools 不使用 VectorIndex/Embedder，不受影响。

  

---

  

### Commit 5: `feat: add Deep Dream validators`

  

post_call_validator 确定性函数，返回校验反馈让 LLM 迭代决策。

  

**新增文件**:

- `deepdream/validators.py` — check_dream_duplicate

  

**关键设计** (validators 不是 @operational_tool，是普通函数):

```python

def check_dream_duplicate(tool_input, ctx, context_fs, uri_resolver) → dict:

    # 在 dream 目录下搜索相同 topic

    # 返回 {"status": "duplicate_found", "duplicate_uri": "...", "suggestion": "..."}

    # 或 {"status": "valid", "suggestion": "可以继续提取这个话题"}

```

  

注意: 不需要 `validate_and_create`（之前的 output_create validator）。写入由 loop 结束后的 commit pipeline 完成。

  

---

  

### Commit 6: `feat: add DeepDreamReActLoop and prompt template`

  

核心 ReAct Loop 类 + prompt YAML 模板。

  

**新增文件**:

- `deepdream/deep_dream_react_loop.py` — DeepDreamReActLoop(ReactLoop)

- `deepdream/prompts/__init__.py`

- `deepdream/prompts/manager.py` — 导出 PromptManager（复用 extraction 的，传入 deepdream template_dir）

- `deepdream/prompts/templates/deepdream.yaml` — system_prompt, identity_block, output_instruction

  

**关键实现**:

```python

class DeepDreamReActLoop(ReactLoop):

    __init__(llm, fs, dream_registry, prompt_driven_registry,

             uri_resolver, prompt_manager, vector_index, embedder,

             operational_tools=None, max_iterations=10, timeout_seconds=120.0)

  

    _runtime_injection_values(ctx) → {"ctx": ctx, "context_fs": fs,

                                       "vector_index": vector_index, "embedder": embedder}

  

    _get_custom_tools() → [spec.tool_dict() for spec in prompt_driven_specs]

  

    _execute_custom_tool(name, input, ctx):

        # 1. schema 校验 (spec.validate)

        # 2. post_call_validator 执行 (validator_dispatch lookup)

        # 3. 收集 DreamOutput (无论 status 是 valid 还是 duplicate_found,

        #    都收集——duplicate 的 DreamOutput 会在 post-loop 处理时与已有节点合并)

        #    注意: 不实际写入 ContextFS，写入由 loop 后的 commit pipeline 完成

  

    _build_messages(ctx, task_ctx):

        # system_prompt + identity + prompt_guidance(各spec) + output_instruction

        # PromptManager 复用（不同 template_dir）

  

    _parse_output(content) → None  # Deep Dream 通过 tool calls 产出，不走 content 解析

  

    run(ctx, task_ctx) → DeepDreamReActResult

```

  

---

  

### Commit 7: `feat: integrate dream category into URIResolver, validation, and policies`

  

让现有基础设施支持 dream category，使 commit pipeline 能处理 DreamOutput candidates。

  

**修改文件**:

- `core/uri_resolver.py` — 添加 `extra_registries` 参数，支持跨 registry URI 解析

- `core/validation.py` — `valid_categories` 添加 `"dream"`

- `commit/merge_policies.py` — 新增 DreamPolicy (always CREATE)

- `commit/policy_router.py` — 注册 DreamPolicy 到 dream category

  

**URIResolver 扩展**:

```python

class URIResolver:

    __init__(registry, extra_registries=None)  # 向后兼容

  

    resolve(memory_type, fields, ctx):

        # 先查 primary registry，再查 extra_registries

        # dream 类型从 extra_registries 中找到 schema

```

  

**DreamPolicy**:

```python

class DreamPolicy:

    # add_only, always CREATE, dream_id 含 content hash 保证唯一

    def plan(candidate, ctx) → WritePlan(action="create", target_uri=...)

```

  

---

  

### Commit 8: `feat: add Deep Dream HTTP endpoint and service wiring`

  

HTTP 端点和 MemoryService 集成。

  

**修改文件**:

- `server/app.py` — 新增 `POST /api/v1/deepdream` 路由

- `server/memory_service.py` — 新增 `deepdream()` 方法 + lazy resource getters

  

**端点**:

```python

@app.route("/api/v1/deepdream", methods=["POST"])

def handle_deepdream():

    ctx = _build_authenticated_context(params)

    result = _get_service().deepdream(params)

    return jsonify(result)

```

  

**Service 方法**:

```python

def deepdream(self, params) → dict:

    # lazy-init: dream_registry, prompt_driven_registry

    # 创建 URIResolver(extraction_registry, extra_registries=[dream_registry])

    # 创建 DeepDreamReActLoop(...)

    # result = loop.run(ctx, task_ctx)

    # 通过 WriteAPI 写入 candidates

    # 返回 {status, iterations, created_uris, tools_used}

```

  

---

  

## 依赖图

  

```

C1 (PromptDrivenToolSpec) ──→ C3 (schemas) ──→ C6 (ReAct Loop)

C2 (models) ──────────────────────────────────→ C6

C4 (operational tools) ──────────────────────→ C6

C5 (validators) ─────────────────────────────→ C6

C3 (dream.yaml) ───────────→ C7 (integration)

C6 (ReAct Loop) ──────────→ C8 (HTTP endpoint)

C7 (integration) ──────────→ C8

```

  

C3, C4, C5, C7 可部分并行。C6 是最大集成点。C8 是最终接线。

  

---

  

## 验证

  

1. `python -m pytest tests/unit/core/test_prompt_driven_tool_spec.py` — 新增 spec 框架测试通过

2. `python -m pytest tests/unit/core/test_operational_tool.py` — 现有测试通过（_RUNTIME_TYPES 扩展不影响）

3. `python -m pytest tests/unit/extraction/` — extraction 测试通过（行为不变）

4. 手动验证: 每个 prompt-driven YAML 的 `PromptDrivenToolSpec.tool_dict()` 格式正确

5. `python -m pytest tests/unit/deepdream/` — Deep Dream 模块测试通过

6. 验证 URIResolver 跨 registry 解析: `resolve("dream", ..., ctx)` 返回正确 URI

7. 验证 DreamPolicy: `plan(candidate)` 返回 WritePlan(action="create")

8. 验证 HTTP 端点: `POST /api/v1/deepdream` 返回正确响应格式