---
type: design
status: draft
created: 2026-05-14
updated: 2026-05-14
tags: [ogmemory, bookkeeper, rfc, team-discussion]
related:
  - "[[Agentic Memory ReAct Research]]"
  - "[[oG-Memory Extraction and Storage Analysis]]"
---

# BookKeeper RFC v3

> **文档目的**：面向团队开发者的设计讨论稿。基于 v2 讨论结论重写。
>
> **与 v2 的核心区别**：BookKeeper 不再是第一步就要落地的独立 agent。现阶段的工作是：(1) 将 extraction ReAct 循环泛化，使其可复用于 deep dream；(2) 在此基础上逐步演化为统一的 BookKeeper。本文描述这条演进路径。

---

## 1. 背景

### 我们有什么

oG-Memory 目前的记忆管理能力：

| 能力 | 实现 | 局限 |
|------|------|------|
| **抽取** | `Extractor` 两阶段（span identification → span structuring） | eager mode 单次抽取，遇到冲突只能靠 schema 规则合并 |
| **检索** | `RetrievalPipeline` 四阶段（planner → seed → expand → rank） | 纯规则 pipeline，不理解用户上下文，复杂查询力不从心 |
| **后台维护** | 无 | 记忆只增不减：重复堆积、过期不清理、关系不主动发现 |

共同问题：这些操作都是**单次、无反馈**的。没有"看一看结果，决定下一步怎么做"的能力。

### 我们需要什么

一个 **LLM-in-the-loop 执行引擎**，能够：

1. **多步检索**：pipeline 结果不足时，基于已有结果追问、沿关系追踪、聚合多源信息
2. **智能抽取**：遇到与已有记忆冲突时，读取已有内容再做合并决策；对低价值信息主动跳过
3. **后台整理**（Dreaming）：定期扫描记忆库，发现重复、过期、可聚合的模式，执行清理

这三件事的核心模式相同：LLM 思考 → 调用工具观察 → 再思考 → 再调用 → 最终产出。即 ReAct 循环。

### 领域趋势

2026 年 Agentic Memory 领域的研究表明，记忆管理正从"被动存储"转向"主动推理驱动"。代表性工作：

- **MIA**（[arXiv:2604.04503](https://arxiv.org/abs/2604.04503)）：Planner-Executor 架构，先规划搜索策略再执行
- **LongSeeker**（[arXiv:2605.05191](https://arxiv.org/abs/2605.05191)）：五种记忆操作原语（Skip / Compress / Rollback / Snippet / Delete），证明有限原语集可表达完备的记忆管理能力
- **MemReader**（[arXiv:2604.07877](https://arxiv.org/abs/2604.07877)）：显式的信息价值评估——区分"值得存储"、"等待更多上下文"、"直接丢弃"
- **HiGMem**（[arXiv:2604.18349](https://arxiv.org/abs/2604.18349)）：用摘要层做锚点定位，再精准读取完整内容，减少无效读取

---

## 2. 需求

oG-Memory 中的记忆操作：

| 模块                    | 状态     | 做什么                                   |
| --------------------- | ------ | ------------------------------------- |
| `compose`             | ✅ 已有   | pipeline 四阶段检索                        |
| `extract` (eager)     | ✅ 已有   | 单次 LLM 调用抽取记忆                         |
| `extract`(react loop) | ✅ 已有   | 多轮抽取：读已有节点 → 决定合并策略 → 抽取              |
| Light Dream           | 🔲 计划中 | 扫描记忆库，发现重复/过期/可聚合的候选                  |
| Deep Dream            | 🔲 计划中 | 基于候选做整理：读、搜索、可能 re-extract、合并/归档/建立关系 |

**关键观察**：

- **Light Dream** 和 `search`、`extract`(eager) 一样，是比较直接的步骤。它可以作为一个工具或 HTTP 服务接口，不需要多轮推理。Dreaming 编排层直接调用即可。
- 由于记忆存储的不同，**Deep Dream** 相较于 `openclaw` 的实现更复杂。它和 `extract`(react loop) 需要相同的核心 react 能力：LLM 思考 → 调用工具 → 观察结果 → 再思考。它们的差异在于工具集和 prompt 不同，但循环逻辑一致。

### 目标

1. **Phase 1 (短期目标)**：泛化 `ExtractionReActLoop`，使其能同时服务抽取和 deep dream 两个场景
2. **Phase 2 (长期)**：演化为一个统一的 BookKeeper react agent——所有需要多轮记忆操作的场景（抽取、检索增强、dreaming）都走同一个入口

### 演进路径

```
Phase 0 (当前)                    Phase 1 (短期目标)              Phase 2 (长期)
┌──────────────────┐             ┌──────────────────┐         ┌──────────────────┐
│ ExtractionReAct  │             │ Generalized      │         │ BookKeeper       │
│ Loop             │             │ ReActLoop        │         │                  │
│                  │  ──泛化──>  │                   │──统一──>│ 统一的 ReAct     │
│ 固定工具集        │             │ 可配置工具集       │         │ agent 入口       │
│ 固定 prompt      │             │ 可配置 prompt     │         │                  │
│ 固定输出类型      │             │ 可配置输出解析     │         │ + 检索增强        │
│                  │            │                  │         │ + 更多场景        │
└──────────────────┘            └──────────────────┘         └──────────────────┘
                                         ↑
                                    Deep Dream
                                    基于此实现
```

## 3. Phase 1 设计

### 3.1 模块设计概述

在 `core/` 下新增 `ReactLoop` 抽象基类，实现固定的 ReAct 循环模板（LLM 思考 → 调用工具 → 观察结果 → 重复或产出）。子类通过 4 个抽象方法提供场景特定的行为：

| 抽象方法 | 职责 |
|---------|------|
| `_get_tools()` | 返回可用工具定义 |
| `_build_messages(ctx, task_ctx)` | 构建初始消息（system prompt + 场景上下文） |
| `_execute_tool(name, input, ctx)` | 执行单个工具调用 |
| `_parse_output(content)` | 将 LLM 最终输出解析为结构化结果 |

加 1 个可选覆盖：

| 可选覆盖 | 职责 |
|---------|------|
| `_should_continue_after_output(parsed, messages, ctx)` | 解析出结果后是否继续循环。默认 False |

`run()` 签名通过 `ReactTaskContext` 基类实现场景参数的统一传递，子类继承 `ReactTaskContext` 携带各自特定的参数。

改动范围：
- `core/react_loop.py` — 新增
- `extraction/react_loop.py` — 改造（ExtractionReActLoop 继承 ReactLoop，外部接口不变）
- `dreaming/deep_dream_loop.py` — 新增（DeepDreamReActLoop，Phase 1 后期）
- `core/interfaces.py`、`core/models.py`、`commit/`、`fs/` — 不改动

### 3.2 实现计划

#### T-1: 新建 core/react_loop.py — 数据结构 + ReactLoop 抽象基类

**描述：**
创建 `core/react_loop.py`，包含通用数据结构和 `ReactLoop` 抽象基类。数据结构从现有 `extraction/react_loop.py` 泛化而来，循环逻辑从现有 `ExtractionReActLoop.run()` 提取。

**涉及文件：**
- 新建：`core/react_loop.py`
- 新建：`tests/unit/core/test_react_loop.py`

**预期产出 — 数据结构：**

```python
@dataclass
class ReactTaskContext:
    """ReAct 循环的任务上下文基类。子类化以携带场景特定参数。"""
    pass

@dataclass
class ReactIterationTrace:
    """单次迭代的追踪信息。"""
    iteration: int
    latency_seconds: float
    tool_calls_count: int
    tool_call_names: list[str]
    content_length: int

@dataclass
class ReactTrace:
    """完整运行的追踪信息。"""
    total_iterations: int
    total_latency_seconds: float
    iterations: list[ReactIterationTrace]
    total_read_uris: int

@dataclass
class ReactLoopResult:
    """ReactLoop.run() 的返回值。"""
    output: list[Any]              # 解析后的操作列表
    tools_used: list[dict]         # 工具调用记录
    iterations: int                # 实际迭代次数
    read_uris: set[str]            # 本次读取过的 URI
    trace: ReactTrace | None
    completed: bool                # 是否正常完成
    finish_reason: str             # "done" | "timeout" | "max_iterations"
```

**预期产出 — ReactLoop 基类：**

```python
class ReactLoop(ABC):
    """通用 ReAct 循环基类。"""

    def __init__(self, llm: LLM, max_iterations: int = 5, timeout_seconds: float = 30.0):
        """llm 和循环参数在构造时确定。场景依赖由子类构造函数处理。"""
        ...

    # 模板方法
    def run(self, ctx: RequestContext, task_ctx: ReactTaskContext) -> ReactLoopResult:
        """执行 ReAct 循环。"""
        ...

    # 子类必须实现
    @abstractmethod
    def _get_tools(self) -> list[dict]: ...
    @abstractmethod
    def _build_messages(self, ctx, task_ctx) -> list[dict]: ...
    @abstractmethod
    def _execute_tool(self, name, input, ctx) -> dict | str: ...
    @abstractmethod
    def _parse_output(self, content) -> list[Any] | None: ...

    # 子类可选覆盖
    def _should_continue_after_output(self, parsed, messages, ctx) -> bool:
        return False
```

**与现有代码的关系**：
- 数据结构 ← 泛化自 `extraction/react_loop.py` 中的 `ReActIterationTrace`(L77)、`ReActTrace`(L89)、`ReActResult`(L99)
- 循环逻辑 ← 提取自 `ExtractionReActLoop.run()`(L167-318)
- `_call_llm()` ← 来自 react_loop.py:396-437
- `_execute_and_append_tool_calls()` ← 来自 react_loop.py:439-503

**循环流程**：
1. `_build_messages()` 构建初始消息
2. `while iteration < max_iterations`：
   - 超时检查 → break
   - 调用 LLM（带 `_get_tools()` 返回的工具）
   - 三路分支：
     - LLM 返回工具调用 → `_execute_tool()` 执行，格式化为消息，继续循环
     - LLM 返回内容 → `_parse_output()` 解析 → `_should_continue_after_output()` 判断是否继续
     - 两者都没有 → 禁用工具，继续循环
3. 返回 `ReactLoopResult`

**验收标准：**
- [ ] 文件创建，所有数据结构和 ReactLoop 可 import
- [ ] 新增单元测试验证基类的循环逻辑（用 mock 子类）
- [ ] 不影响现有任何代码（纯新增文件）

---

#### T-2: 改造 ExtractionReActLoop 继承 ReactLoop

**描述：**
将 `ExtractionReActLoop` 改为继承 `ReactLoop`，把现有方法映射到基类的抽象方法。外部接口（`run()` 签名）完全不变。

**涉及文件：**
- 修改：`extraction/react_loop.py`

**预期产出：**

```python
@dataclass
class ExtractionTaskContext(ReactTaskContext):
    """抽取场景的任务参数。"""
    conversation_text: str
    prefetch_result: PrefetchResult | None = None

class ExtractionReActLoop(ReactLoop):
    def __init__(self, llm, fs, registry, uri_resolver, prefetcher=None,
                 max_iterations=3, timeout_seconds=30.0):
        super().__init__(llm, max_iterations, timeout_seconds)
        self._fs = fs
        self._registry = registry
        self._uri_resolver = uri_resolver
        self._prefetcher = prefetcher

    # 外部接口不变
    def run(self, conversation_text, ctx, prefetch_result=None) -> ReActResult:
        task_ctx = ExtractionTaskContext(conversation_text, prefetch_result)
        result = super().run(ctx, task_ctx)
        return self._to_react_result(result)  # 向后兼容转换

    # 抽象方法实现（从现有方法平移）
    def _get_tools(self) -> list[dict]: ...           # ← 现有 react_loop.py:157-160
    def _build_messages(self, ctx, task_ctx): ...      # ← 现有 react_loop.py:320-394
    def _execute_tool(self, name, input, ctx): ...     # ← 现有 react_loop.py:505-565
    def _parse_output(self, content): ...              # ← 现有 react_loop.py:567-634

    # 可选覆盖
    def _should_continue_after_output(self, parsed, messages, ctx) -> bool:
        ...  # ← 现有 refetch 逻辑 react_loop.py:636-738

    # 保留的抽取专属方法（不暴露给基类）
    def _check_unread_existing_files(...): ...          # ← 现有 react_loop.py:636-679
    def _add_refetch_results(...): ...                  # ← 现有 react_loop.py:681-738
    def _validate_operations_uris(...): ...             # ← 现有 react_loop.py:740-767
    def _candidate_to_fields(...): ...                  # ← 现有 react_loop.py:769-797
    def _to_react_result(...): ...                      # ← 新增：ReactLoopResult → ReActResult 转换
```

**关键约束**：
- `ExtractionReActLoop` 的调用方（`Extractor._structure_span_lazy`）不需要任何改动
- `run()` 返回值类型仍为 `ReActResult`（通过 `_to_react_result` 转换）
- 现有数据结构（`ReActResult`、`ReActIterationTrace`、`ReActTrace`）保留在 `extraction/react_loop.py`，不删除（向后兼容）

**验收标准：**
- [ ] ExtractionReActLoop的所有测试仍然通过（功能不变）

---

#### T-3: DeepDreamReActLoop 实现 [设计中]

> **状态说明**：本任务的实现依赖工具层的准备，同时需要与 Light Dream 模块设计对齐（如候选输入格式、扫描策略的衔接）。下方列出各工具的实现状态，待工具层就绪后再细化实现计划。

**涉及文件：**
- 新建：`dreaming/deep_dream_loop.py`
- 新建：`tests/unit/dreaming/test_deep_dream_loop.py`

**预期产出（框架不变）**：`DeepDreamReActLoop` 继承 `ReactLoop`，实现 `_get_tools`、`_build_messages`、`_execute_tool`、`_parse_output`。具体代码结构待工具层确定后补充。

**Deep Dream 工具集及实现状态**：

| 工具                                | 功能       | 状态      | 说明                                                                         |
| --------------------------------- | -------- | ------- | -------------------------------------------------------------------------- |
| `read(uri)`                       | 读取节点内容   | ✅ 已实现   | `ExtractionReActLoop` 中已有 `_READ_TOOL`，可直接复用                               |
| `list(uri)`                       | 列出子节点    | ✅ 已实现   | 已有 `_LIST_TOOL`，可直接复用                                                      |
| `get_relations(uri)`              | 获取关系边    | ⚠️ 部分实现 | `_RELATIONS_TOOL` 定义已有，`_execute_get_relations()` 在 `react_loop.py:803-822` 已实现，但 `ExtractionReActLoop` 缺少 `_relation_store` 属性，当前通过 `hasattr(self._fs, 'get_relations')` 检查永远失败。依赖 Relation PRD P0-R3：注入 `RelationStore`（遵循关注点分离） |
| `extract_*(fields)`               | 重新抽取     | ✅ 已实现   | `build_extraction_tools()` 已有，可直接复用                                        |
| `archive_node(uri)`               | 归档节点     | ✅ 已实现   | `ContextFS.archive_node()` 接口已存在                                           |
| `create_relation(src, tgt, type)` | 建立关系     | ✅ 已实现   | `RelationStore.upsert_edges()` 接口已存在                                       |
| `search(query, filters)`          | 搜索记忆     | ⚠️ 部分实现 | `ReadAPI.search_memory()` 封装了 `RetrievalPipeline`，但未封装为 tool 定义。需添加薄适配层    |
| `update_node(uri, fields)`        | 更新节点字段   | ⚠️ 部分实现 | `ContextWriter.write_candidate()` 仅接受 `CandidateMemory`，不支持任意字段更新。需适配或新增方法 |
| `merge_nodes(uris, strategy)`     | 合并节点     | 🔲 规划中  | 无现有实现。需：读 N 个节点 → LLM 合并 → 写入结果。是否作为原子操作待讨论                                |
| `find_duplicates(category)`       | 查找重复节点   | 🔲 规划中  | 需基于 `VectorIndex` + 相似度阈值实现                                                |
| `find_related(uri, max_hops)`     | 沿关系图发现关联 | 🔲 规划中  | 需基于 `RelationStore.get_one_hop()` 递归遍历                                     |
| `compute_importance(uri)`         | 计算节点重要性  | 🔲 规划中  | 需基于元数据（access_count × recency × relation_count）计算                          |

**Deep Dream 机制设计思考**：

Deep Dream 可部分参考 OpenClaw 的实现思路，但由于 oG-Memory 的存储结构更复杂（ContextFS 层级存储 vs OpenClaw 扁平槽位），需要独立设计。初步需要解决的问题：

1. **抽取不全**：eager mode 单次抽取可能遗漏信息。Deep Dream 需要识别"看起来不完整"的节点，触发 re-extract 补充
2. **分类不准**：event 被误分类为 entity、同类 entity 未归并；如果抽出了多个 `entity1_entity2` 类型节点，以哪个为准需要规则或 LLM 判断
3. **热度/重要性评估**：参考 OpenClaw Dreaming 的 hot 评分机制（基于访问频率、时效性、关系数），但如何应用到 oG-Memory 的层级存储结构还需设计

---

## 4. Phase 2 展望

当 Phase 1 的泛化 `ReactLoop` + `DeepDreamReActLoop` 稳定运行后，可以演化为统一的 BookKeeper agent。本节描述目标形态和关键设计决策，细节留到实施时再定。

### 4.1 目标形态

```
Phase 1                              Phase 2
┌─────────────────────┐             ┌─────────────────────────────┐
│ ExtractionReActLoop │             │ BookKeeper                  │
│ ┌─────────────────┐ │             │ ┌─────────────────────────┐ │
│ │   ReactLoop     │ │   ──统一──> │ │  ReactLoop              │ │
│ └─────────────────┘ │             │ └─────────────────────────┘ │
├─────────────────────┤             │                             │
│ DeepDreamReActLoop  │             │ 统一入口 + 场景注册：       │
│ ┌─────────────────┐ │   ──统一──> │  BookKeeper.run(scene, ...) │
│ │   ReactLoop     │ │             │                             │
│ └─────────────────┘ │             │  - extraction               │
└─────────────────────┘             │  - deep_dream               │
                                    │  - retrieval (新增)          │
                                    └─────────────────────────────┘
```

### 4.2 核心变化

**统一入口**：子类独立 `run()` → `BookKeeper.run(scene, ...)`。场景通过注册机制加载，每个场景提供自己的工具集、prompt、输出解析。

**检索增强**（新增场景）：当 `RetrievalPipeline` 结果不足时，fallback 到 ReAct 循环——基于已有结果追问、沿关系追踪、聚合多源信息。这是 Phase 2 新增的第三个场景。

**模块重组**：`ReactLoop` 从 `core/react_loop.py` 移到独立的 `bookkeeper/` 模块，成为一等公民。

### 4.3 Open to discussion

**统一 action vocabulary**：Phase 1 中各场景的 `_parse_output` 返回不同类型（extraction → `CandidateMemory`，deep dream → 操作列表）。Phase 2 需要标准化的操作原语。领域研究表明（LongSeeker，arXiv:2605.05191），有限的原语集（Skip / Compress / Merge / Archive / Extract）即可表达完备的记忆管理能力。初步候选：

| 原语                | 语义             |
| ----------------- | -------------- |
| `extract`         | 抽取新记忆或重新抽取已有节点 |
| `merge`           | 合并多个节点         |
| `archive`         | 归档/降级节点        |
| `skip`            | 跳过当前操作（信息价值不足） |
| `create_relation` | 建立关系边          |

**检索增强的触发条件**：如何判断 pipeline 结果"不足"？是由调用方决定还是 BookKeeper 内部判断？

**安全边界**：Deep Dream 可以归档/合并节点，retrieval 场景是只读的。不同场景的安全策略如何统一管理？多租户场景如何进行权限管理？
