---
type: concept
status: active
created: 2026-05-19
updated: 2026-05-19
tags: [ogmemory, react, abstraction, template-method, design-pattern, architecture]
related:
  - "[[oG-Memory Extraction and Storage Analysis]]"
  - "[[Agentic Memory ReAct Research]]"
---

# oG-Memory ReactLoop 抽象基类设计

ReactLoop 是 oG-Memory 中 ReAct (Reasoning-Acting) 模式的抽象基类实现，为 extraction 和 retrieval 等场景提供通用的工具调用循环模板。

> 代码位置: `core/react_loop.py` | 提交: `47fa9e13` (泛化), `fa36014f` (重命名)

---

## 设计模式

采用 **Template Method** 模式。基类 `run()` 定义循环骨架，子类实现 4 个抽象方法 + 2 个可选 override。

```
┌─────────────────────────────────────────────────────────────┐
│                      ReactLoop.run()                        │
├─────────────────────────────────────────────────────────────┤
│  1. _build_messages() → 初始化消息                          │
│  2. while iteration < max_iterations:                       │
│     ├─ timeout 检查                                         │
│     ├─ _call_llm() → (tool_calls, content)                 │
│     ├─ 三路分支:                                            │
│     │  ├─ tool_calls → _execute_tool() → continue          │
│     │  ├─ content → _parse_output() → should_continue?     │
│     │  └─ neither → disable_tools → continue               │
│     └─ 追踪迭代 latency/tool_calls                          │
│  3. return ReactLoopResult                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 核心类

### 数据类

| 类 | 职责 |
|---|------|
| `ReactTaskContext` | 任务上下文基类，子类扩展携带场景参数 |
| `ReactIterationTrace` | 单次迭代追踪：iteration, latency, tool_calls_count, tool_call_names |
| `ReactTrace` | 完整运行追踪：iterations[], total_latency, total_read_uris |
| `ReactLoopResult` | 返回结果：output, tools_used, iterations, read_uris, completed, finish_reason |

### 抽象方法（必须实现）

```python
@abstractmethod
def _get_tools(self) -> list[dict]:
    """返回可用工具定义 (OpenAI function calling 格式)"""

@abstractmethod
def _build_messages(self, ctx, task_ctx) -> list[dict]:
    """构建初始消息（系统提示 + 上下文）"""

@abstractmethod
def _execute_tool(self, name: str, input: dict, ctx) -> dict | str:
    """执行单个工具调用"""

@abstractmethod
def _parse_output(self, content: str) -> list[Any] | None:
    """解析 LLM 输出为结构化结果"""
```

### 可选 Override

```python
def _should_continue_after_output(parsed, messages, ctx, iter_trace) -> bool:
    """解析成功后是否继续循环。默认 False（停止）"""

def _create_iter_trace(...) -> ReactIterationTrace:
    """Factory Method：创建迭代追踪。子类可返回自己的 Trace 子类"""
```

---

## 子类继承扩展

### ExtractionReActLoop (`extraction/extraction_react_loop.py`)

| 继承类 | 扩展字段 |
|--------|----------|
| `ExtractionIterationTrace(ReactIterationTrace)` | `safety_check_triggered`, `refetch_uris` |
| `ExtractionTrace(ReactTrace)` | `final_candidate_count` |
| `ExtractionTaskContext(ReactTaskContext)` | `conversation_text`, `prefetch_result` |

Override 方法:
- `_get_tools()` → 返回 read/list/relations/access_stats + 动态 extraction 工具
- `_create_iter_trace()` → 返回 `ExtractionIterationTrace`
- `_build_messages()` → 系统提示 + prefetch context + conversation
- `_execute_tool()` → if-elif 分发各工具
- `_parse_output()` → JSON 解析 → CandidateMemory
- `_should_continue_after_output()` → 安全检查：未读文件 refetch
- `_track_tool_usage()` → 统计 read URI 到 `self._read_uris`

---

## 设计要点

### Factory Method `_create_iter_trace`

允许基类 `run()` 创建正确的追踪类型，避免 `_to_react_result` 中手动类型转换。

```python
# 基类默认
def _create_iter_trace(...) -> ReactIterationTrace:
    return ReactIterationTrace(...)

# Extraction 子类 override
def _create_iter_trace(...) -> ExtractionIterationTrace:
    return ExtractionIterationTrace(...)
```

### `_read_uris` 提升到基类

通用统计逻辑，所有 ReAct 循环都需要追踪已读 URI。子类通过 `_track_tool_usage` override 扩展统计规则。

### 循环控制

- `max_iterations`: 最大迭代次数（默认 5）
- `timeout_seconds`: 总超时（默认 30s）
- `_disable_tools_for_iteration`: 未知工具触发，下轮禁用工具
- Last iteration 消息提示 LLM 返回最终结果

### Warning 日志

基类记录:
- `_parse_output` 返回 None → "LLM content but parse failed"
- 既无 tool_calls 也无 content → "neither tools nor content"
- Last iteration 无输出 → "returning empty output"

---

## 测试覆盖

`tests/unit/core/test_react_loop.py` 使用 `MockReactLoop` 测试基类逻辑:

- Tool call → output 转换
- Max iterations exhausted
- Empty response disables tools
- Content parse failure handling
- `_should_continue_after_output` override
- Timeout graceful exit
- LLM exception handling
- Last iteration termination message

---

## 重构历程

1. **原状**: `extraction/react_loop.py` 中 `ExtractionReActLoop` 包含所有逻辑
2. **Commit 1** (`fa36014f`): 重命名 `react_loop.py` → `extraction_react_loop.py`，类加 Extraction 前缀
3. **Commit 2** (`47fa9e13`):
   - 提取 `ReactLoop` 抽象基类到 `core/react_loop.py`
   - 追踪类使用继承而非重复定义
   - `_read_uris` / `_build_tool_descriptions` 提升到基类
   - Factory Method `_create_iter_trace` 避免手动类型转换
   - 补回原有注释

---

## 延后讨论

ToolDef 注册模式（轻量无 handler 版本）曾提案但未实施。当前定义-执行-追踪仍通过手写字符串耦合，改了工具名需改三处。

---

*Session: 2026-05-19 | Commits: fa36014f, 47fa9e13 (amended)*