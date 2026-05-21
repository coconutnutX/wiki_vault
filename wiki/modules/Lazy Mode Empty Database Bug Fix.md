---
name: lazy-mode-empty-db-fix
description: Lazy模式空数据库场景无法提取的bug修复记录，含统一解析重构
tags:
  - bug-fix
  - extraction
  - react-loop
  - lazy-mode
  - refactor
created: 2026-05-20
updated: 2026-05-20
---

# Lazy Mode Empty Database Bug Fix

## 问题

Lazy 模式在空数据库场景下返回空 candidates，无法正常提取。

## 根因分析

### 1. Prompt 未指导空数据库场景

[extraction_react_loop.py](../oG-Memory/extraction/extraction_react_loop.py) 的 system prompt 只说 "read before deciding"，没有指导 LLM 在找不到已有记忆时如何处理。

### 2. tool_choice 参数错误

[openai_llm.py](../oG-Memory/providers/llm/openai_llm.py) 在 `tools=None` 时传递 `tool_choice="none"`，导致 OpenAI API 报错：
```
'tool_choice' is only allowed when 'tools' are specified
```

### 3. tool_calls 格式不匹配

- OpenAILLM 返回：`{"tool": name, "input": {...}}`
- react_loop.py 期望：`{"name": ..., "input": ...}`

### 4. extract_* 工具调用未转换为 candidates

LLM 通过 tool calling 调用 `extract_*`，但 `_to_react_result()` 只从 `loop_result.output` 提取 candidates，未处理 `tools_used`。

## 修复（Commit dc2a5e9b）

### 文件修改清单

| 文件 | 修改 |
|------|------|
| [extraction/extraction_react_loop.py](../oG-Memory/extraction/extraction_react_loop.py) | 添加 instruction 4："If no existing memories found, proceed to create new directly" |
| [providers/llm/openai_llm.py](../oG-Memory/providers/llm/openai_llm.py) | `tools=None` 时设置 `tool_choice=None` |
| [core/react_loop.py](../oG-Memory/core/react_loop.py) | 支持 `tool` 和 `name` 两种格式；过滤空工具名 |
| [extraction/extraction_react_loop.py](../oG-Memory/extraction/extraction_react_loop.py) | `_to_react_result()` 从 `tools_used` 提取 candidates |

## 统一解析重构（Commit cf025bfe）

### 问题发现

Bug 修复后发现数据结构不一致：
- `tools_used` 存储 `{"tool_name": name, "params": input, "result": {"tool": name, "input": input, "status": "recorded"}}`
- `result` 中又存了一份 `tool` + `input`，冗余且与 Eager 模式格式不匹配

### 重构方案

**1. `_execute_tool` 简化返回结构**：
```python
# 之前（冗余）
return {"tool": name, "input": input, "status": "recorded"}

# 之后（简洁）
return {"status": "recorded"}
```

**2. 添加 `_to_candidate` 统一解析方法**：
```python
def _to_candidate(self, tool_name: str, tool_input: dict) -> CandidateMemory | None:
    """Convert extract_* tool call to CandidateMemory.
    Unified parsing used by both _parse_output and _to_react_result.
    """
    if not tool_name.startswith("extract_"):
        return None
    if "raw" in tool_input and len(tool_input) == 1:
        return None  # skip unparsed
    parsed = parse_tool_call(tool_name, tool_input, self._registry)
    if parsed:
        return parsed[2]
    return None
```

**3. `_parse_output` 使用 `_to_candidate`**：
```python
candidate = self._to_candidate(tool_name, tool_input)
if candidate:
    candidates.append(candidate)
```

**4. `_to_react_result` 使用 `params` + `_to_candidate`**：
```python
for tool_record in loop_result.tools_used:
    tool_params = tool_record.get("params", {})  # 用 params，不用 result.input
    if tool_result.get("status") == "recorded":
        candidate = self._to_candidate(tool_name, tool_params)
        if candidate:
            candidates.append(candidate)
```

### 数据结构对比

| 位置 | 之前 | 之后 |
|------|------|------|
| `_execute_tool` 返回 | `{"tool": name, "input": input, "status": "recorded"}` | `{"status": "recorded"}` |
| `_to_react_result` 数据源 | `result.input` | `params` |
| 解析调用 | 内联 `parse_tool_call` | `_to_candidate` |

## 结果

Promptfoo 测试：`0% → 100% passed`（6 tests）

Eager 和 Lazy 模式都能正确提取：
- 同名实体（两个张伟：程序员 vs 设计师）
- 时间表达（下周三 → 2026-05-27）
- 用户偏好（Python/FastAPI）

## 相关

- [[Promptfoo Extraction Testing Framework]]
- [[oG-Memory ReactLoop Abstract Base Class]]
- [[schema-deadlock-multiprocess]]

## 教训

- Lazy 模式的 prompt 需明确指导"无数据时直接创建"
- 工具调用结果需要被正确收集转换为输出
- 多进程并发初始化需考虑跨进程锁
- **统一解析方法避免 Eager/Lazy 数据结构不一致**