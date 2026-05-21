---
type: module
title: "oG-Memory Tool Definition Refactor Analysis"
status: active
tags: [ogmemory, tool-definition, refactor, @tool-decorator]
created: 2026-05-21
updated: 2026-05-21
---

# oG-Memory 工具定义改造分析

> 日期: 2026-05-21 | 分析目标: 评估 tool_builder.py / tool_schemas.py 改造成 @tool 装饰器模式的可行性

---

## 一、现有架构分析

### 1.1 文件依赖关系

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Schema Registry                              │
│   extraction/schemas/definitions/*.yaml                             │
│   (YAML 定义 schema: fields, description, required)                  │
└─────────────────────────────────────────────────────────────────────┘
                              ↓ 加载
┌─────────────────────────────────────────────────────────────────────┐
│                       SchemaRegistry                                 │
│   extraction/schemas/registry.py                                     │
│   - 从 YAML 加载 schema                                              │
│   - list_enabled() 返回可用 schema                                   │
└─────────────────────────────────────────────────────────────────────┘
                              ↓ 动态生成
┌─────────────────────────────────────────────────────────────────────┐
│                       tool_builder.py                                │
│   - build_extraction_tools(registry) → 工具定义列表                  │
│   - build_tool_to_category(registry) → 映射 dict                     │
│   - parse_tool_call(name, input, registry) → CandidateMemory        │
└─────────────────────────────────────────────────────────────────────┘
                              ↓ 被 Extractor 使用
┌─────────────────────────────────────────────────────────────────────┐
│                       tools.py (Extractor)                           │
│   - _extraction_tools: 工具定义列表                                  │
│   - _tool_to_category: 映射 dict                                     │
│   - _parse_tool_call_fn: 解析函数                                    │
│   - _to_candidate(): 转换为 CandidateMemory                          │
└─────────────────────────────────────────────────────────────────────┘
                              ↓ 被 service/api.py 使用
┌─────────────────────────────────────────────────────────────────────┐
│                       service/api.py                                 │
│   - MemoryWriteAPI._create_extractors() → [Extractor]               │
│   - commit_session() 调用 extractor.extract()                        │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│         tool_schemas.py (Legacy, 硬编码 Pydantic Model)              │
│   - ExtractProfileInput, ExtractPreferenceInput, ...                │
│   - ClassVar tool_name, tool_description                            │
│   - get_tool_definitions() → 硬编码生成                              │
│   (仅当 SchemaRegistry 未提供时使用)                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 关键数据流

```python
# 1. YAML schema 定义
# extraction/schemas/definitions/profile.yaml
memory_type: profile
description: "Extract user profile attributes..."
fields:
  - name: routing_key
    type: string
    required: true
    description: "Attribute identifier"

# 2. SchemaRegistry 加载
registry = SchemaRegistry()  # 自动加载 YAML
schemas = registry.list_enabled()

# 3. tool_builder.py 动态生成工具定义
tools = build_extraction_tools(registry)
# 输出: [{"name": "extract_profile", "description": "...", "input_schema": {...}}]

mapping = build_tool_to_category(registry)
# 输出: {"extract_profile": ("profile", "user")}

# 4. Extractor 使用
class Extractor:
    def __init__(self, schema_registry):
        if schema_registry:
            self._extraction_tools = build_extraction_tools(schema_registry)
            self._tool_to_category = build_tool_to_category(schema_registry)
        else:
            # Legacy fallback
            self._extraction_tools = EXTRACTION_TOOLS  # from tool_schemas.py

# 5. LLM 调用工具
tool_calls = llm.complete_with_tools(prompt, tools=self._extraction_tools)

# 6. 解析工具调用
result = parse_tool_call(tool_name, tool_input, registry)
# 输出: (memory_type, owner_scope, CandidateMemory)
```

### 1.3 使用方分析

| 文件 | 使用方式 | 影响 |
|------|----------|------|
| `service/api.py` | 创建 `Extractor(schema_registry)` | 直接依赖 |
| `commit/candidate_pipeline.py` | 调用 `extractor.extract()` | 间接依赖 |
| `extraction/tools.py` | 内部使用 `_extraction_tools`, `_tool_to_category` | 核心依赖 |
| `extraction/tool_builder.py` | `build_extraction_tools()`, `parse_tool_call()` | 改造核心 |
| `extraction/tool_schemas.py` | Legacy fallback | 可废弃 |

---

## 二、@tool 装饰器模式分析

### 2.1 LangChain @tool 模式

```python
from langchain_core.tools import tool

@tool
def extract_profile(
    routing_key: str,
    abstract: str,
    overview: str,
    content: str,
    confidence: float,
) -> CandidateMemory:
    """Extract individual user profile attributes.
    
    Args:
        routing_key: Attribute identifier (e.g., 'name', 'location')
        abstract: Brief summary (≤200 chars)
        ...
    """
    return CandidateMemory(
        category="profile",
        routing_key=routing_key,
        abstract=abstract,
        ...
    )
```

**优势**：
- 工具名自动从函数名提取（`extract_profile`）
- Schema 自动从函数签名生成
- Description 自动从 docstring 提取
- 无需维护列表

**问题**：
- 硬编码：category, owner_scope 写在函数内部
- 不支持动态配置：新增 category 需改代码
- 与 YAML schema 驱动架构冲突

### 2.2 核心矛盾

| 特性 | 现有架构 (YAML + Registry) | @tool 模式 |
|------|---------------------------|------------|
| **Schema 定义** | YAML 文件（数据驱动） | Python 函数（代码驱动） |
| **动态性** | 可动态启用/禁用 schema | 需改代码 |
| **扩展性** | 新增 YAML 即可扩展 | 新增函数 + 注册 |
| **工具名** | 从 schema.tool_name | 从函数名 |
| **Validation** | `parse_tool_call()` 动态验证 | Pydantic 自动验证 |
| **配置管理** | 外部 YAML 文件 | 代码内部 |

---

## 三、改造方案

### 方案 A：完全迁移到 @tool（不推荐）

**改造内容**：
1. 删除 `SchemaRegistry`, `tool_builder.py`, `tool_schemas.py`
2. 为每个 category 创建 `@tool` 函数
3. 修改 `Extractor` 直接使用 `@tool` 函数列表

**工作量**：
- 10 个 YAML schema → 10 个 `@tool` 函数
- 修改 `Extractor._structure_span_eager()` 调用方式
- 修改 `parse_tool_call()` 逻辑
- 修改 `service/api.py` 创建方式

**问题**：
- 失去 YAML 配置的灵活性
- 硬编码 category 信息
- 与现有架构设计理念冲突

**影响范围**：

| 文件 | 改动量 |
|------|--------|
| `extraction/tools.py` | 重写工具调用逻辑 (~100 行) |
| `service/api.py` | 修改 Extractor 创建 (~20 行) |
| 新增 `extraction/tool_functions.py` | 新建 (~200 行) |
| 删除 `extraction/schemas/*` | 删除目录 |
| 删除 `extraction/tool_builder.py` | 删除文件 |
| 删除 `extraction/tool_schemas.py` | 删除文件 |

**结论**：不推荐，损失架构优势。

---

### 方案 B：混合模式（保留 YAML + 添加 @tool 辅助）

**改造思路**：
- YAML schema 继续作为数据源
- `@tool` 装饰器作为**辅助层**，自动生成函数

```python
# extraction/tool_functions.py

from langchain_core.tools import tool
from extraction.schemas.registry import SchemaRegistry

_registry = SchemaRegistry()

def _generate_tool_function(schema):
    """从 YAML schema 动态生成 @tool 函数"""
    
    @tool
    def generated_tool(**kwargs) -> CandidateMemory:
        """{schema.description}"""
        return _parse_to_candidate(schema, kwargs)
    
    generated_tool.name = schema.tool_name
    return generated_tool

# 自动生成所有工具
DREAM_TOOLS = [_generate_tool_function(s) for s in _registry.list_enabled()]
```

**优势**：
- 保留 YAML 配置灵活性
- 获得 @tool 的自动 schema 生成
- 无需手动维护工具列表

**工作量**：
- 新增 `extraction/tool_functions.py` (~50 行)
- 修改 `Extractor` 使用生成函数列表 (~30 行)

**问题**：
- 函数签名无法精确反映 YAML fields
- 动态生成的函数难以调试

**结论**：可行，但复杂度增加，收益有限。

---

### 方案 C：改良现有架构（推荐）

**改造思路**：
- 不引入 @tool 装饰器
- 改进现有 ClassVar 硬编码问题
- 保持 YAML + Registry 的架构优势

**具体改进**：

```python
# extraction/tool_builder.py 改进版

def build_extraction_tools(registry: SchemaRegistry) -> list[dict]:
    """生成工具定义（改进：自动从 schema 生成，无需 ClassVar）"""
    tools = []
    for schema in registry.list_enabled():
        # tool_name 从 schema.memory_type 自动推导，无需硬编码
        tool_name = f"extract_{schema.memory_type}"
        
        # description 直接从 schema.description
        # input_schema 从 schema.fields 自动生成
        tools.append({
            "name": tool_name,
            "description": schema.description,
            "input_schema": _build_input_schema(schema.fields),
        })
    return tools

# 问题解决：
# 1. tool_name 不再硬编码在 ClassVar，从 schema 推导
# 2. 无需维护 DREAM_TOOL_MODELS 列表，从 registry.list_enabled() 自动获取
# 3. 新增 category 只需添加 YAML，无需改代码
```

**工作量**：
- 优化 `tool_builder.py` (~20 行改动)
- 可选：删除 `tool_schemas.py`（legacy）

**优势**：
- 保留架构设计优势（YAML 数据驱动）
- 解决 ClassVar 硬编码问题
- 最小改动量

**结论**：推荐，最小改动 + 最大收益。

---

## 四、影响范围评估

### 4.1 方案对比

| 方案 | 改动文件数 | 改动行数 | 架构变化 | 风险 |
|------|-----------|----------|----------|------|
| A: 完全迁移 @tool | 6 | ~300 | 重大重构 | 高 |
| B: 混合模式 | 3 | ~80 | 新增辅助层 | 中 |
| C: 改良现有 | 1-2 | ~20 | 最小改进 | 低 |

### 4.2 方案 C 具体改动

**改动文件**：

| 文件 | 改动内容 |
|------|----------|
| `extraction/tool_builder.py` | 优化 `build_extraction_tools()`，确保 tool_name 从 schema 推导 |
| `extraction/tool_schemas.py` | 可删除（legacy fallback）或保留兼容 |

**不改动文件**：
- `extraction/schemas/registry.py` - 保持不变
- `extraction/schemas/definitions/*.yaml` - 保持不变
- `extraction/tools.py` - 保持不变（使用现有接口）
- `service/api.py` - 保持不变

### 4.3 测试覆盖

现有测试覆盖：
- `tests/unit/extraction/test_tool_builder.py`
- `tests/unit/service/test_memory_fs.py`
- `tests/integration/test_full_pipeline.py`

改动后需补充测试：
- `build_extraction_tools()` 新 schema 测试
- 新增 YAML schema 端到端测试

---

## 五、结论与建议

### 5.1 不推荐完全迁移 @tool

原因：
1. **架构冲突**：现有 YAML 数据驱动架构设计良好，@tool 是代码驱动
2. **损失灵活性**：YAML 允许动态启用/禁用、扩展 category
3. **改动量大**：需重写多个核心文件

### 5.2 推荐改良现有架构（方案 C）

改动点：

```python
# extraction/tool_builder.py 关键改动

def build_extraction_tools(registry: SchemaRegistry) -> list[dict]:
    tools = []
    for schema in registry.list_enabled():
        # tool_name 从 schema 自动推导（核心改进）
        tool_name = f"extract_{schema.memory_type}"
        
        # 无需 ClassVar 硬编码
        tools.append({
            "name": tool_name,
            "description": schema.description,
            "input_schema": _build_input_schema(schema),
        })
    return tools
```

收益：
- 解决 ClassVar 硬编码问题
- 保留 YAML 配置灵活性
- 最小改动量（~20 行）

### 5.3 Deep Dream 可直接使用 @tool

Deep Dream 与现有 extraction 架构**独立**，可直接使用 @tool 模式：

```python
# deepdream/tools.py

from langchain_core.tools import tool

@tool
def memory_acquire_recent(limit: int = 100) -> list[ContextNode]:
    """获取最近 N 条记忆。"""
    ...

@tool  
def deep_dream(acquire: str, process: str) -> DeepDreamReport:
    """执行完整 Deep Dream 流程。"""
    ...
```

原因：
- Deep Dream 是新模块，无历史包袱
- 工具数量固定（~5 个），不需要动态配置
- 符合 Agent 调用模式

---

## 六、执行建议

| 模块 | 建议 |
|------|------|
| `extraction/tool_builder.py` | 改良：确保 tool_name 从 schema 推导，消除 ClassVar 硬编码 |
| `extraction/tool_schemas.py` | 可删除（legacy），或保留 fallback 兼容 |
| `deepdream/tools.py` | 新模块直接使用 @tool 装饰器 |
| 测试 | 补充 tool_builder 改动后的单元测试 |

**优先级**：
1. Deep Dream 新模块使用 @tool（立即）
2. extraction/tool_builder.py 改良（后续）
3. 删除 tool_schemas.py legacy（最后）

---

## 参考

- [[ogmemory-deep-dream-framework]] - Deep Dream 设计文档
- [LangChain @tool decorator](https://python.langchain.com/docs/how_to/custom_tools/)
- [Anthropic Claude tool use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)