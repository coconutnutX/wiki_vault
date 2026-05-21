---
type: module
title: "oG-Memory Operation Tool Optimization"
status: active
tags: [ogmemory, tools, operation-tools, registry, decorator]
created: 2026-05-21
updated: 2026-05-21
---

# oG-Memory 操作工具优化设计

> 日期: 2026-05-21 | 背景: Extraction tools 保留现有设计，优化操作工具（write/list/search）解决 switch-case、name 硬编码、复用困难问题

---

## 一、现状分析

### 1.1 Extraction Tools（保留现有设计）

**已确认保留**：
- YAML 配置驱动（SchemaRegistry）
- Pydantic 校验（tool_schemas.py）
- ClassVar 硬编码 tool_name（特殊设计：结构化结果作为 tool input 节省 token）

**原因**：
1. 可配置性：通过 YAML 动态启用/禁用 schema
2. Token 优化：结构化结果作为 tool input，校验后直接生成 CandidateMemory
3. 设计初衷明确：extraction 是"输出工具"，Agent 产生信息

### 1.2 Operation Tools（需要优化）

**现状**：
```python
# service/api.py - API 形态（非 function calling 工具）
ReadAPI.search_memory(query, ctx) → SearchMemoryResult
ReadAPI.read_memory(uri, ctx) → RetrievedBlock

# service/memory_fs.py - 文件系统形态
MemoryFS.list(path, ctx) → List[Dict]
MemoryFS.stat(path, ctx) → Dict
MemoryFS.read_abstract(path, ctx) → str

# commit/context_writer.py - 写入形态
ContextWriter.write_candidate(candidate, ctx) → WritePlan
```

**问题**：
- 目前是 API，不是 Agent 可调用的 function calling 工具
- 未来需要工具化时，需要设计工具注册/调用机制
- 不应沿用 extraction 的 ClassVar 硬编码模式

---

## 二、优化目标

### 2.1 解决三个问题

| 问题 | 现状 | 目标 |
|------|------|------|
| **switch-case 逻辑** | for 循环匹配 tool_name（tool_schemas.py:368） | Registry 自动注册，直接 dict 查找 |
| **name 容易出错** | ClassVar 硬编码，可能与实际不匹配 | 从函数名自动推导，编译时检查 |
| **不直观不好复用** | 需维护 TOOL_MODELS 列表、get_tool_definitions() | 自动注册，无需手动维护列表 |

### 2.2 工具分层架构

参考 [[ogmemory-tool-category-analysis]] 的分层建议：

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Operation Tools 分层                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Layer 1: 只读工具（安全，可开放给 Agent）                           │
│   ─────────────────────────────────                                 │
│   search_memory(query) → 搜索记忆                                    │
│   read_memory(uri) → 读取记忆                                        │
│   list_memories(path) → 列出记忆                                     │
│                                                                      │
│   Layer 2: 写入工具（受限，特定场景）                                 │
│   ─────────────────────────────────                                 │
│   write_memory(candidate) → 写入新记忆                               │
│   archive_memory(uri) → 归档记忆                                     │
│                                                                      │
│   Layer 3: 高危工具（谨慎开放）                                       │
│   ─────────────────────────────────                                 │
│   update_memory(uri, content) → 更新记忆                             │
│   delete_memory(uri) → 删除记忆                                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 三、优化方案

### 3.1 方案 A：Registry 模式（推荐）

**核心思路**：
- 自动注册：装饰器注册工具，无需手动维护列表
- 自动推导：tool_name 从函数名生成，避免硬编码
- 类型安全：函数签名自动生成 schema，编译时检查

```python
# tools/registry.py

from typing import Callable, Dict, Any
from dataclasses import dataclass

@dataclass
class ToolMeta:
    """工具元信息"""
    name: str
    description: str
    layer: int  # 1=只读, 2=写入, 3=高危
    handler: Callable
    input_schema: Dict[str, Any]

class ToolRegistry:
    """操作工具注册中心"""
    
    _tools: Dict[str, ToolMeta] = {}
    
    @classmethod
    def register(cls, layer: int, description: str = ""):
        """装饰器：自动注册工具
        
        Args:
            layer: 工具层级 (1=只读, 2=写入, 3=高危)
            description: 工具描述（可选，默认从 docstring 提取）
        """
        def decorator(fn: Callable) -> Callable:
            # 自动推导 name
            tool_name = fn.__name__
            
            # 从 docstring 提取 description
            doc = description or (fn.__doc__ or "").strip()
            
            # 从函数签名生成 input_schema
            input_schema = _build_schema_from_signature(fn)
            
            cls._tools[tool_name] = ToolMeta(
                name=tool_name,
                description=doc,
                layer=layer,
                handler=fn,
                input_schema=input_schema,
            )
            return fn
        return decorator
    
    @classmethod
    def get_tool(cls, name: str) -> ToolMeta | None:
        """直接 dict 查找，无 switch-case"""
        return cls._tools.get(name)
    
    @classmethod
    def list_tools(cls, layer: int | None = None) -> list[Dict]:
        """列出工具定义（用于 LLM function calling）"""
        tools = []
        for meta in cls._tools.values():
            if layer is None or meta.layer <= layer:
                tools.append({
                    "name": meta.name,
                    "description": meta.description,
                    "input_schema": meta.input_schema,
                })
        return tools


def _build_schema_from_signature(fn: Callable) -> Dict[str, Any]:
    """从函数签名自动生成 JSON Schema"""
    import inspect
    from typing import get_type_hints
    
    hints = get_type_hints(fn)
    sig = inspect.signature(fn)
    
    properties = {}
    required = []
    
    for name, param in sig.parameters.items():
        if name in ("ctx", "self"):
            continue  # 跳过 context 参数
        
        type_hint = hints.get(name, str)
        prop_def = _type_to_json_schema(type_hint)
        
        if param.default is inspect.Parameter.empty:
            required.append(name)
        
        properties[name] = prop_def
    
    return {
        "type": "object",
        "properties": properties,
        "required": required,
    }


def _type_to_json_schema(type_hint) -> Dict[str, Any]:
    """Python type → JSON Schema"""
    if type_hint == str:
        return {"type": "string"}
    elif type_hint == int:
        return {"type": "integer"}
    elif type_hint == float:
        return {"type": "number"}
    elif type_hint == bool:
        return {"type": "boolean"}
    elif type_hint == list:
        return {"type": "array"}
    else:
        return {"type": "string"}  # 默认
```

**使用示例**：

```python
# tools/read_tools.py

from tools.registry import ToolRegistry

@ToolRegistry.register(layer=1)
def search_memory(query: str, top_k: int = 10) -> SearchResult:
    """Search memories by semantic similarity.
    
    Args:
        query: Search query text
        top_k: Number of results to return (default 10)
    
    Returns:
        SearchResult with hits and scores
    """
    # 实现逻辑...

@ToolRegistry.register(layer=1)
def read_memory(uri: str) -> RetrievedBlock:
    """Read a specific memory by URI.
    
    Args:
        uri: Memory URI (ctx://account/users/user/memories/...)
    
    Returns:
        RetrievedBlock with full content
    """
    # 实现逻辑...

@ToolRegistry.register(layer=2)
def write_memory(category: str, routing_key: str, abstract: str, content: str) -> WriteResult:
    """Write a new memory node.
    
    Args:
        category: Memory category (event, case, pattern, skill)
        routing_key: Unique identifier within category
        abstract: Brief summary (≤200 chars)
        content: Full content
    
    Returns:
        WriteResult with target_uri and action
    """
    # 实现逻辑...
```

**调用方式**：

```python
# Agent 调用工具（无 switch-case）

def handle_tool_call(tool_name: str, tool_input: dict, ctx: RequestContext):
    # 直接 dict 查找
    tool = ToolRegistry.get_tool(tool_name)
    if tool is None:
        raise ValueError(f"Unknown tool: {tool_name}")
    
    # 调用 handler
    return tool.handler(**tool_input, ctx=ctx)

# 获取工具定义（LLM function calling）
tools = ToolRegistry.list_tools(layer=2)  # 只返回 Layer 1-2 工具
```

---

### 3.2 方案对比

| 方案 | switch-case | name 硬编码 | 复用性 | 改动量 |
|------|-------------|------------|--------|--------|
| **现有 extraction** | for 循环匹配 | ClassVar 硬编码 | 需维护列表 | 保留不变 |
| **方案 A: Registry** | dict 直接查找 | 函数名自动推导 | 自动注册 | 新增 registry.py |

---

## 四、设计优势

### 4.1 无 switch-case

```python
# 现有（extraction tool_schemas.py:368）
for m in TOOL_MODELS:
    if m.tool_name == tool_name:  # ← switch-case 逻辑
        model = m
        break

# Registry 模式
tool = ToolRegistry._tools.get(tool_name)  # ← dict 直接查找
```

### 4.2 name 自动推导

```python
# 现有（硬编码可能不匹配）
class ExtractProfileInput:
    tool_name: ClassVar[str] = "extract_profile"  # ← 硬编码

# Registry 模式
@ToolRegistry.register(layer=1)
def search_memory(...):  # ← name 从函数名自动推导
    ...
```

### 4.3 自动注册

```python
# 现有（手动维护列表）
TOOL_MODELS = [
    ExtractProfileInput,
    ExtractPreferenceInput,
    ...
]

# Registry 模式
@ToolRegistry.register(layer=1)
def search_memory(...): pass

@ToolRegistry.register(layer=1)
def read_memory(...): pass

# 无需维护列表，装饰器自动注册
```

---

## 五、与 Extraction Tools 的区别

### 5.1 Extraction Tools（保留现有）

**特殊设计原因**：
1. YAML 配置驱动 → 可动态启用/禁用
2. 结构化结果作为 tool input → 节省 token
3. 工具作用是校验 → 不是执行操作

### 5.2 Operation Tools（使用 Registry）

**设计差异**：
1. 执行操作 → 不是校验，需要实际执行逻辑
2. 固定工具集 → 不需要 YAML 配置动态性
3. 函数签名 → 直接定义输入 schema，无需 Pydantic Model

---

## 六、实施建议

### 6.1 新模块使用 Registry

```python
# deepdream/tools.py - 新模块直接使用 Registry

from tools.registry import ToolRegistry

@ToolRegistry.register(layer=1)
def memory_acquire_recent(limit: int = 100) -> list[ContextNode]:
    """获取最近 N 条记忆。"""
    ...

@ToolRegistry.register(layer=2)
def deep_dream_acquire(acquire: str, process: str) -> DeepDreamReport:
    """执行 Deep Dream 流程。"""
    ...
```

### 6.2 现有 API 工具化（可选）

```python
# tools/read_tools.py - 将现有 ReadAPI 工具化

from service.api import ReadAPI
from tools.registry import ToolRegistry

@ToolRegistry.register(layer=1)
def search_memory(query: str, top_k: int = 10) -> SearchMemoryResult:
    """Search memories by semantic similarity."""
    return read_api.search_memory(query, ctx, top_k=top_k)

@ToolRegistry.register(layer=1)
def read_memory(uri: str) -> RetrievedBlock:
    """Read a specific memory by URI."""
    return read_api.read_memory(uri, ctx)
```

### 6.3 分层权限控制

```python
# Agent 调用时按层级过滤

# 只读 Agent：只使用 Layer 1
tools = ToolRegistry.list_tools(layer=1)

# Deep Dream：使用 Layer 1-2
tools = ToolRegistry.list_tools(layer=2)

# 管理后台：使用全部 Layer 1-3
tools = ToolRegistry.list_tools(layer=3)
```

---

## 七、结论

### 7.1 Extraction Tools 保留现有设计

原因：
1. YAML 配置驱动（特殊需求）
2. Token 优化（结构化结果作为 tool input）
3. 设计初衷明确（校验工具）

### 7.2 Operation Tools 使用 Registry 模式

优势：
1. **无 switch-case** → dict 直接查找
2. **name 自动推导** → 从函数名生成
3. **自动注册** → 装饰器注册，无需维护列表
4. **分层控制** → layer 参数控制权限

### 7.3 新模块优先使用 Registry

Deep Dream 等新模块直接使用 Registry，无需历史包袱。

---

## 参考

- [[ogmemory-tool-category-analysis]] - Extraction vs 操作工具本质区别分析
- [[ogmemory-tool-definition-refactor-analysis]] - 工具定义改造分析
- [[ogmemory-deep-dream-framework]] - Deep Dream 设计文档