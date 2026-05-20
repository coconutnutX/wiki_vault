---
name: lazy-mode-empty-db-fix
description: Lazy模式空数据库场景无法提取的bug修复记录
tags:
  - bug-fix
  - extraction
  - react-loop
  - lazy-mode
created: 2026-05-20
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

## 修复

### 文件修改清单

| 文件 | 修改 |
|------|------|
| [extraction/extraction_react_loop.py](../oG-Memory/extraction/extraction_react_loop.py) | 添加 instruction 4："If no existing memories found, proceed to create new directly" |
| [providers/llm/openai_llm.py](../oG-Memory/providers/llm/openai_llm.py) | `tools=None` 时设置 `tool_choice=None` |
| [core/react_loop.py](../oG-Memory/core/react_loop.py) | 支持 `tool` 和 `name` 两种格式；过滤空工具名 |
| [extraction/extraction_react_loop.py](../oG-Memory/extraction/extraction_react_loop.py) | `_to_react_result()` 从 `tools_used` 提取 candidates |

### 核心代码修改

**1. System Prompt 补充**：
```python
## Instructions
4. **If no existing memories found (read/list returns empty or error), 
   proceed to create new memories directly using extract_* tools.**
```

**2. Candidates 提取**：
```python
def _to_react_result(self, loop_result, ctx):
    candidates = [...]
    
    # 从 tools_used 提取 extract_* 工具调用
    for tool_record in loop_result.tools_used:
        if tool_record.get("tool_name", "").startswith("extract_"):
            tool_input = tool_record.get("result", {}).get("input", {})
            parsed = parse_tool_call(tool_name, tool_input, self._registry)
            if parsed:
                candidates.append(parsed[2])
```

## 结果

Promptfoo 测试从 `0% passed` 提升到 `100% passed`。

Eager 和 Lazy 模式都能正确提取：
- 同名实体（两个张伟）
- 时间表达（下周三 → 2026-05-27）
- 用户偏好（Python/FastAPI）

## 相关

- [[Promptfoo Extraction Testing Framework]]
- [[schema-deadlock-multiprocess]]

## 教训

- Lazy 模式的 prompt 需明确指导"无数据时直接创建"
- 工具调用结果需要被正确收集转换为输出
- 多进程并发初始化需考虑跨进程锁