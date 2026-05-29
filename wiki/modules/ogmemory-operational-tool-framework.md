---
type: module
title: "oG-Memory @operational_tool Framework Design"
status: approved
tags: [ogmemory, tools, operational-tool, decorator, react-loop, architecture]
created: 2026-05-27
updated: 2026-05-27
related:
  - "[[ogmemory-tool-category-analysis]]"
  - "[[ogmemory-operation-tool-optimization]]"
  - "[[ogmemory-tool-definition-refactor-analysis]]"
  - "[[ogmemory-deep-dream-framework]]"
---

# oG-Memory @operational_tool 框架设计

> 日期: 2026-05-27 | 状态: 已批准 | 背景: 重新审视 [[ogmemory-deep-dream-framework]] 中方案 B（@tool 装饰器）的可行性，结合现有代码确认 operational tools 和 extraction tools 是两类不同东西，需要不同的定义方式

---

## 一、问题分析

### 1.1 现有工具体系中的两类东西

| 维度 | Extraction tools | Operational tools |
|------|-----------------|-------------------|
| schema 来源 | YAML（动态可配置） | 硬编码 dict（固定） |
| LLM 交互方式 | LLM "填表" — 填字段值 | LLM "调函数" — 传参数 |
| 注册方式 | SchemaRegistry | module-level dict |
| 执行方式 | `parse_tool_call()` → CandidateMemory | `_execute_tool()` if/elif 链 |
| 适用场景 | 需要动态增删 memory type | 固定功能的读/写/查询 |

### 1.2 当前 operational tools 的问题

4 个工具（read, list, get_relations, get_access_stats）在 `extraction/extraction_react_loop.py` 中定义为硬编码 dict，handler 写在 `_execute_tool()` 的 if/elif 链里：

- **不可复用** — 其他 ReactLoop 无法复用这些工具定义和 handler
- **不可组合** — 不同 loop 需要不同的工具集，但没有配置机制
- **维护分散** — schema 定义和 handler 实现分离在不同位置

### 1.3 对 Deep Dream 方案 B 的重新审视

[[ogmemory-deep-dream-framework]] 的方案 B（LangChain @tool 装饰器）直觉正确——operational tools 是函数，不是"填表"。但原设计有两个硬伤：

1. **运行时依赖不能进 LLM schema** — `ctx: RequestContext`、`context_fs: ContextFS` 是运行时注入的对象，LLM 无法提供
2. **引入 LangChain 依赖** — 项目不使用 LangChain，不应为此引入重依赖

修正思路：自写轻量 `@operational_tool`，从函数签名自动生成 schema，但过滤掉 runtime-only 参数类型。

---

## 二、关键设计决策

1. **装饰器不包函数** — 只在 `fn._operational_tool_spec` 上挂一个 spec 对象，原函数身份不变，测试可直接调用
2. **runtime 参数过滤** — 签名里类型属于 `_RUNTIME_TYPES`（RequestContext, ContextFS 等）的参数不进 LLM schema，执行时由 `call_with_injection()` 注入
3. **不是全局注册表** — 工具定义放在各自的模块（`extraction/operational_tools.py`, `deep_dream/operational_tools.py`），每个 loop 通过构造器 `operational_tools` 参数选择要用哪些，默认值是模块级 list
4. **dispatch 是 dict 查找** — `_execute_tool()` 用 `name → spec` dict 替代 if/elif 链

---

## 三、框架核心：`core/operational_tool.py`

### 3.1 OperationalToolSpec

```python
_RUNTIME_TYPES: set[type] = {RequestContext, ContextFS}  # 可扩展

@dataclass
class OperationalToolSpec:
    name: str                  # 工具名（默认取函数名，可 @operational_tool(name=...) 覆盖）
    description: str           # docstring 首段
    input_schema: dict         # OpenAI JSON Schema（只含 LLM 可见参数）
    handler: Callable          # 原函数
    llm_params: list[tuple[str, type]]     # (name, type) LLM 可见
    runtime_params: list[tuple[str, type]] # (name, type) 运行时注入

    def tool_dict(self) -> dict:
        return {"name": self.name, "description": self.description, "input_schema": self.input_schema}
```

### 3.2 @operational_tool 装饰器

```python
def operational_tool(fn=None, *, name=None):
    """从函数签名+docstring 自动生成 spec，挂到 fn._operational_tool_spec

    处理流程：
    1. inspect.signature + get_type_hints
    2. 按 _RUNTIME_TYPES 分离 llm_params / runtime_params
    3. _extract_tool_description(doc) 取首段
    4. _extract_param_docs(doc) 取 :param name: desc
    5. 构建 input_schema（只含 llm_params）
    6. fn._operational_tool_spec = OperationalToolSpec(...)
    7. return fn（不包）
    """
```

支持 `@operational_tool` 和 `@operational_tool(name="custom_name")` 两种用法。

### 3.3 call_with_injection

```python
def call_with_injection(spec, llm_input, **runtime_values):
    """把 LLM 参数和 runtime 注入合并成 kwargs，调 spec.handler

    匹配策略：按参数名匹配 → 按 type 匹配 → 报错
    """
    kwargs = dict(llm_input)
    for param_name, param_type in spec.runtime_params:
        # 按 name 匹配（快路径）
        if param_name in runtime_values:
            kwargs[param_name] = runtime_values[param_name]
            continue
        # 按 type 匹配（回退）
        for key, value in runtime_values.items():
            if isinstance(value, param_type):
                kwargs[param_name] = value
                break
    return spec.handler(**kwargs)
```

---

## 四、工具定义：`extraction/operational_tools.py`

迁移现有 4 个硬编码 dict 为 `@operational_tool` 函数：

```python
@operational_tool
def read(uri: str, ctx: RequestContext, context_fs: ContextFS) -> dict:
    """Read a memory node by URI.
    :param uri: URI of the memory node to read.
    """
    if not uri:
        return {"error": "Missing required parameter: uri"}
    node = context_fs.read_node(uri, ctx)
    return {"uri": uri, "abstract": node.abstract, ...}

@operational_tool
def list(uri: str, ctx: RequestContext, context_fs: ContextFS) -> dict:
    """List child nodes under a directory URI.
    :param uri: URI of the directory to list.
    """
    ...

@operational_tool(name="get_relations")
def get_relations(uri: str, ctx: RequestContext, context_fs: ContextFS) -> dict:
    """Get related nodes for a URI. Returns one-hop neighbors.
    :param uri: URI of the node to get relations for.
    """
    ...

@operational_tool(name="get_access_stats")
def get_access_stats(uri: str, ctx: RequestContext, context_fs: ContextFS) -> dict:
    """Get access stats for a node: last access time, 30d hit count.
    :param uri: URI of the node to get access stats for.
    """
    ...

EXTRACTION_OPERATIONAL_TOOLS = [read, list, get_relations, get_access_stats]
```

注意：`list` 保持为函数名（与现有 tool name "list" 一致），模块级覆盖 builtin 在同一 scope 内安全。

---

## 五、ReactLoop 对接

### 5.1 ExtractionReActLoop 修改

**删除**：
- `_READ_TOOL`, `_LIST_TOOL`, `_RELATIONS_TOOL`, `_ACCESS_STATS_TOOL` module-level dicts
- `_execute_get_relations()` 和 `_execute_get_stats()` 方法

**新增构造器参数**：
```python
def __init__(self, ..., operational_tools=None):
    tool_fns = operational_tools or EXTRACTION_OPERATIONAL_TOOLS
    self._operational_tool_specs = [fn._operational_tool_spec for fn in tool_fns]
    self._operational_dispatch = {spec.name: spec for spec in self._operational_tool_specs}
    self._all_tools = [spec.tool_dict() for spec in self._operational_tool_specs] + self._extraction_tools
```

**替换 `_execute_tool()`**：
```python
def _execute_tool(self, name, input, ctx):
    try:
        spec = self._operational_dispatch.get(name)
        if spec is not None:
            return call_with_injection(spec, input, ctx=ctx, context_fs=self._fs)
        if name.startswith("extract_"):
            return {"status": "recorded"}
        return {"error": f"Unknown tool: {name}"}
    except Exception as e:
        return {"error": str(e)}
```

### 5.2 不改动的部分

- `core/react_loop.py` — 基类不变
- `extraction/tool_builder.py` — extraction 工具继续走 SchemaRegistry
- `extraction/schemas/` — YAML 定义不动
- `core/interfaces.py` — 不动
- `core/models.py` — 不动

---

## 六、未来扩展：Deep Dream ReactLoop

```python
# deep_dream/operational_tools.py
@operational_tool
def memory_acquire_recent(limit: int = 100, category_filter: list[str] | None = None,
                           ctx: RequestContext, context_fs: ContextFS) -> list[str]:
    """获取最近 N 条记忆 URI 作为 Deep Dream 输入。
    :param limit: 最大获取数量。
    :param category_filter: 可选的 category 过滤列表。
    """
    ...

DEEP_DREAM_OPERATIONAL_TOOLS = [memory_acquire_recent, ...]

# deep_dream/deep_dream_react_loop.py
class DeepDreamReActLoop(ReactLoop):
    def __init__(self, llm, fs, operational_tools=None, ...):
        tool_fns = operational_tools or DEEP_DREAM_OPERATIONAL_TOOLS
        self._operational_tool_specs = [fn._operational_tool_spec for fn in tool_fns]
        self._operational_dispatch = {spec.name: spec for spec in self._operational_tool_specs}
        ...

    def _execute_tool(self, name, input, ctx):
        spec = self._operational_dispatch.get(name)
        if spec:
            return call_with_injection(spec, input, ctx=ctx, context_fs=self._fs)
        return {"error": f"Unknown tool: {name}"}
```

**混用不同工具集**：
```python
tools = EXTRACTION_OPERATIONAL_TOOLS + DEEP_DREAM_OPERATIONAL_TOOLS
loop = SomeHybridLoop(llm, fs, operational_tools=tools, registry=registry)
```

---

## 七、验证

1. `python -m pytest tests/unit/core/` — 新增 operational_tool 测试通过
2. `python -m pytest tests/unit/extraction/` — 现有 extraction 测试通过（行为不变）
3. 手动验证：每个 `@operational_tool` 函数的 `spec.tool_dict()` 与旧硬编码 dict 结构一致
4. `python -m pytest tests/integration/extraction/` — integration 测试通过

---

## 八、变更日志

| 日期 | 变更 |
|------|------|
| 2026-05-27 | 初版：基于 [[ogmemory-deep-dream-framework]] 方案 B 重新审视，确认 operational tools 和 extraction tools 应分开定义，设计 @operational_tool 框架 |