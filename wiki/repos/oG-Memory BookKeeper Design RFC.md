---
type: design
status: draft
created: 2026-05-14
updated: 2026-05-14
tags: [ogmemory, bookkeeper, dreaming, retrieval, design, rfc]
related:
  - "[[oG-Memory Extraction and Storage Analysis]]"
  - "[[OpenClaw Dreaming Mechanism]]"
  - "[[OpenClaw MEMORY.md Lifecycle]]"
  - "[[OpenClaw Memory System Overview]]"
---

# oG-Memory BookKeeper 设计 RFC

讨论将现有 `ExtractionReActLoop` 泛化为 **BookKeeper**——一个通用的记忆管理 ReAct agent，作为 oG-Memory 后台服务的内部组件，在检索、抽取/存储、dreaming 三大场景中按需运行。

> 状态：早期设计讨论，核心问题尚未定论。

---

## 一、现状分析

### 当前 ReAct 能力（extraction/react_loop.py）

`ExtractionReActLoop` 是一个**专用于抽取**的 ReAct 循环：

- 固定工具集：`read`, `list`, `get_relations`, `get_access_stats`, `extract_*`
- 固定目标：从对话中抽取 `CandidateMemory`
- 安全机制：写入前自动 refetch 未读的已有节点（防覆盖）
- 触发点：仅 `after_turn` → `_structure_span_lazy` 路径

**局限**：

| 方面 | 现状 | 缺失 |
|------|------|------|
| 检索 | 纯 pipeline（4 阶段串行），无 ReAct | 复杂查询的多轮探索、关系追踪 |
| 存储 | 仅有 eager mode 和 lazy mode | 存储时的冲突发现、记忆合并、关系更新 |
| 后台维护 | 无 | dreaming / 记忆整理 / 过期清理 |

### 对比 OpenClaw 的 Dreaming

OpenClaw 的 dreaming 分三个阶段（[[OpenClaw Dreaming Mechanism]]）：

```
Light Phase  →  REM Phase  →  Deep Phase
(短期召回统计)   (候选评分过滤)   (晋升写入 MEMORY.md)
```

但 OpenClaw 的存储是**文件系统**（MEMORY.md），dreaming 本质是"从短期笔记中挑选值得长期保留的内容"。oG-Memory 的存储是**结构化数据库**（context_nodes + 向量索引），这带来了本质区别：

| 维度 | OpenClaw | oG-Memory |
|------|----------|-----------|
| 存储 | 文本文件，追加写入 | 结构化节点，upsert/merge |
| 梳理手段 | 评分 + 阈值门控 | 可利用关系图、访问统计、时效性 |
| 输出 | 追加 section 到 MEMORY.md | 更新/合并/删除/新建节点 |
| 复杂度 | 相对简单（文本拼接） | 更复杂（需要理解节点间的语义关系） |

---

## 二、核心概念：BookKeeper

### 定义

**BookKeeper** = oG-Memory 内部的 LLM-in-the-loop 执行引擎。它是一个"图书管理员"——按照给定的指令和工具集，在记忆库中执行多轮搜索、分析、读写操作。

### BookKeeper 在 oG-Memory 服务中的定位

oG-Memory 是一个 **ToB 后台服务**，通过 REST API 对外提供记忆能力，设计上支持多租户、跨团队。BookKeeper 不是外部 API，而是服务**内部的执行单元**。

```
┌──────────────────────────────────────────────────────────────┐
│                     oG-Memory Service                        │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  REST API    │  │  REST API    │  │  REST API    │  ...  │
│  │  /compose    │  │  /after_turn │  │  /dreaming   │       │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │
│         │                 │                  │               │
│         ▼                 ▼                  ▼               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              BookKeeper（按需激活）                     │   │
│  │                                                        │   │
│  │  每次运行绑定一个 RequestContext                        │   │
│  │  → 天然继承租户隔离（account_id + owner_space）         │   │
│  │  → 权限边界由调用方控制（传入哪些 tools）               │   │
│  └──────────────────────────────────────────────────────┘   │
│                           │                                  │
│                           ▼                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │        存储层（SQLContextFS + 向量索引）                │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### 多租户下的 BookKeeper 实例化

BookKeeper **不是全局单例**，而是**每次运行绑定一个 `RequestContext`**。这意味着：

| 问题                  | 答案                                                                            |
| ------------------- | ----------------------------------------------------------------------------- |
| BookKeeper 是针对哪个租户？ | 每次运行时绑定。由调用方（API handler / dreaming scheduler）指定 `account_id` + `owner_space` |
| 能跨租户操作吗？            | 不能。与现有 API 共享同一个 `RequestContext` + RLS 隔离                                    |
| 团队级别的知识整理？          | 通过 `visible_owner_spaces` 实现。团队共享空间的 BookKeeper 可以看到团队所有成员在该空间下的记忆            |
| 系统级别的 BookKeeper？   | 当前不建议。系统级操作（如全局 schema 管理）不属于 BookKeeper 职责                                   |

### "图书管理员"的权限边界

BookKeeper 的权限由**工具集**决定——调用方传入什么工具，它就能做什么。

| 权限级别 | 可用操作                                                  | 适用场景          |
| ---- | ----------------------------------------------------- | ------------- |
| 只读   | search, read, list, get_relations, get_access_stats   | 复杂检索增强        |
| 受限写入 | 只读 + extract_*, update_node（仅 merge 策略）               | 抽取/存储         |
| 全权写入 | 只读 + 所有分析 + 所有写入工具（含 archive, merge, create_relation） | dreaming deep |
| 团队管理 | 全权写入 + 跨 owner_space 操作（团队共享空间内）                      | 团队知识整理        |

**BookKeeper 不能做的事**（无论权限级别）：
- 修改其他租户的记忆
- 删除节点（只能 archive）
- 绕过 PolicyRouter 的合并策略
- 直接操作数据库（必须通过 ContextFS 接口）

---

## 三、BookKeeper 的调用关系

### 谁调用 BookKeeper？

BookKeeper 是一个**被调用者**，有三个调用方：

```
                    ┌──────────────────┐
                    │   BookKeeper     │
                    │ (LLM + 工具循环)  │
                    └────────┬─────────┘
                             │ 调用工具
                    ┌────────┴─────────┐
                    │   search / read  │
                    │   extract_*      │
                    │   update_node    │
                    │   merge_nodes    │
                    │   ...            │
                    └──────────────────┘

调用方（谁调用 BookKeeper）：

  ① /compose API handler         → BookKeeper(retrieval mode)
     "pipeline 结果不够，请追加搜索"

  ② /after_turn API handler      → BookKeeper(extraction mode)
     "这个 span 需要多轮抽取"

  ③ DreamingOrchestrator         → BookKeeper(light/deep mode)
     "请扫描并整理这个租户的记忆"
```

### Dreaming 和 BookKeeper 的关系

确认：**Dreaming 是编排层，BookKeeper 是执行引擎**。Dreaming 的 Light Phase 和 Deep Phase 各自是 BookKeeper 的一次运行，配置不同的 prompt + tools。

```
DreamingOrchestrator（编排层）
  │
  ├── 调度逻辑：每天凌晨 3 点为每个活跃租户执行一轮
  │
  ├── Light Phase
  │   └── BookKeeper.run(
  │         ctx = RequestContext(account="acme", owner_space="user:alice"),
  │         tools = [search, read, get_access_stats, find_duplicates, ...],
  │         prompt = "扫描近 7 天的记忆，发现值得整理的模式...",
  │         max_iterations = 5,
  │       )
  │       → 输出：候选任务列表 → 持久化到 dreaming_candidates 表
  │
  └── Deep Phase（可以是同一次运行，也可以是另一次 cron）
      └── BookKeeper.run(
            ctx = RequestContext(account="acme", owner_space="user:alice"),
            tools = [read, search, update_node, merge_nodes, create_relation, ...],
            prompt = "根据候选任务整理记忆...",
            candidates = light_result.candidates,
            max_iterations = 10,
          )
          → 输出：实际写入操作 → 通过 ContextWriter 执行
```

关键点：`light_dream`、`deep_dream` 不是 BookKeeper 内部的模式名，而是 **DreamingOrchestrator 为 BookKeeper 准备的 prompt + tools 配置**。BookKeeper 自身不感知"light"和"deep"的区别——它只知道"按这个 prompt，用这些工具，执行这个任务"。

---

## 四、三个触发场景

### 场景 A：复杂检索

**触发条件**：pipeline 检索结果不足（低分、缺失）或查询需要多步推理。

**示例查询**：
- "上次讨论数据库迁移时的决策是什么？后来有没有变更？" → 先找事件，再沿关系找后续
- "这个用户对前端框架的偏好是什么？" → 聚合多条 preference
- "最近一个月讨论过的重要架构决策" → 时间过滤 + 重要性判断

**方案**：pipeline 先执行，结果不够好时 fallback 到 BookKeeper。

```
query → pipeline.run()
          ↓
       结果评估（score 不足 / hit 数过少）
          ↓ 触发
       BookKeeper.run(
         ctx = request_ctx,
         tools = [search, read, list, get_relations, find_related, aggregate],
         prompt = "在已有搜索结果基础上，进一步查找...",
         initial_context = pipeline_result,
         max_iterations = 3,
       )
          ↓
       追加结果 → 合并返回
```

### 场景 B：存储时（Extraction）

**触发条件**：当前 eager mode 无法处理的情况——低 confidence、潜在冲突。

**与 dreaming 的本质区别**：

| 维度 | 存储 BookKeeper | Dreaming |
|------|----------------|----------|
| 时机 | 实时，after_turn 中 | 后台，定时或空闲 |
| 输入 | 当前对话消息 | 存储中的已有记忆 |
| 目标 | 准确抽取新信息，处理冲突 | 发现有价值的模式、聚合、清理 |
| 时延要求 | 低（秒级） | 无限制（可分钟级） |
| 写入范围 | 仅当前对话相关内容 | 该租户的任意记忆 |

**结论**：存储场景不需要独立的 BookKeeper 模式。增强现有 `ExtractionReActLoop` 的能力——增加冲突检测工具，在 lazy mode 路径中使用即可。后续可以将 `ExtractionReActLoop` 内部实现委托给 BookKeeper（extraction tools 配置）。

### 场景 C：Dreaming（后台记忆整理）

**触发条件**：定时（cron），为每个活跃租户执行。

**Light mode**（快速扫描）：
- 发现被多次召回但缺少跨引用的记忆
- 检测同 category 下的潜在重复
- 发现可以聚合为 pattern 的 event 群
- 检测过期/冲突的 profile 字段
- 输出：候选任务 → 持久化

**Deep mode**（执行更新）：
- 聚合相关记忆（多条 event → 一条 pattern）
- 合并重复节点
- 构建关系边（"决策 X 影响了项目 Y"）
- 归档过期记忆
- 输出：通过 ContextWriter 执行的写入操作

---

## 五、工具设计

### 工具分层

BookKeeper 的工具分三层，调用方按需组合：

```
Layer 1 — 只读（所有场景共享）
  ├── search(query, filters, top_k)
  ├── read(uri)
  ├── list(uri, depth?)
  ├── get_relations(uri, direction?)
  └── get_access_stats(uri)

Layer 2 — 分析（retrieval + dreaming）
  ├── find_related(uri, max_hops) → 沿关系图探索
  ├── find_duplicates(category, time_range?) → 发现重复节点
  ├── detect_conflicts(uris[]) → 检测语义冲突
  ├── compute_importance(uri) → 综合评分
  └── aggregate(uris[], strategy) → 合并多个节点的摘要

Layer 3 — 写入（extraction + dreaming_deep）
  ├── extract_*(fields) → 现有抽取工具
  ├── update_node(uri, fields) → 更新节点（通过 PolicyRouter）
  ├── merge_nodes(uris[], strategy) → 合并节点
  ├── create_relation(source, target, type, weight)
  └── archive_node(uri) → 归档（软删除）
```

### 按场景组合

| 场景 | Layer 1 | Layer 2 | Layer 3 | max_iter | timeout |
|------|---------|---------|---------|----------|---------|
| 复杂检索 | all | find_related, aggregate | none | 3 | 15s |
| 抽取增强 | read, list, get_relations | detect_conflicts | extract_*, update_node | 3 | 30s |
| Dreaming Light | all | all | none | 5 | 60s |
| Dreaming Deep | all | all | all | 10 | 300s |

---

## 六、多租户架构

### Dreaming 的租户调度

```
DreamingScheduler（系统级）
  │
  │  每小时扫描活跃租户列表
  │
  ├── 为 tenant "acme" 启动 DreamingOrchestrator
  │     ctx = RequestContext(account="acme", ...)
  │     → BookKeeper 运行在 acme 的 RLS 隔离下
  │
  ├── 为 tenant "globex" 启动 DreamingOrchestrator
  │     ctx = RequestContext(account="globex", ...)
  │     → BookKeeper 运行在 globex 的 RLS 隔离下
  │
  └── ... 并发控制：最多 N 个租户同时 dreaming
```

### 团队知识整理

ToB 场景中，团队共享记忆需要特殊的 BookKeeper 权限：

```
# 个人记忆整理
BookKeeper.run(
  ctx = RequestContext(
    account="acme",
    owner_space="user:alice",
    visible_owner_spaces=["user:alice"],
  ),
  ...
)

# 团队知识整理（可看到团队共享空间 + 自己的记忆）
BookKeeper.run(
  ctx = RequestContext(
    account="acme",
    owner_space="team:backend",
    visible_owner_spaces=["team:backend", "user:alice"],
  ),
  tools=[...team_level_tools...],
  prompt="整理团队共享的架构知识...",
)
```

**待讨论**：团队级别的 Dreaming 是由团队管理员手动触发，还是自动调度？如果是自动的，频率如何？团队空间的记忆和个人空间的记忆整理策略应有所不同。

---

## 七、模块接口规范

### 7.1 模块依赖总图

```
                    ┌─────────────────────────────┐
                    │        调用方（上游）          │
                    │                             │
                    │  ① /compose handler          │
                    │  ② /after_turn handler       │
                    │  ③ LightDreaming / DeepDreaming │
                    └──────────┬──────────────────┘
                               │ 调用 BookKeeper.run()
                               ▼
                    ┌─────────────────────────────┐
                    │        BookKeeper            │
                    │                             │
                    │  输入：Task + Tools + Config  │
                    │  输出：BookKeeperResult       │
                    └──────────┬──────────────────┘
                               │ 调用工具
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
     │  只读层       │ │  分析层       │ │  写入层       │
     │  (Layer 1)   │ │  (Layer 2)   │ │  (Layer 3)   │
     ├──────────────┤ ├──────────────┤ ├──────────────┤
     │ ContextFS    │ │ RelationStore│ │ ContextWriter │
     │ VectorIndex  │ │ (同左)       │ │ PolicyRouter  │
     │ RelationStore│ │              │ │ SchemaRegistry│
     │ Embedder     │ │              │ │               │
     └──────────────┘ └──────────────┘ └──────────────┘
              │                │                │
              └────────────────┼────────────────┘
                               ▼
                    ┌─────────────────────────────┐
                    │    core/ 已有接口             │
                    │  ContextFS, LLM, VectorIndex │
                    │  RelationStore, Embedder     │
                    │  SchemaRegistry, URIResolver │
                    └─────────────────────────────┘
```

### 7.2 BookKeeper 自身的接口

#### 构造函数 — BookKeeper 需要什么

```python
class BookKeeper:
    def __init__(
        self,
        llm: LLM,                              # core.interfaces.LLM — 已有 Protocol
        fs: ContextFS,                          # core.interfaces.ContextFS — 已有 Protocol
        relation_store: RelationStore,          # core.interfaces.RelationStore — 已有 Protocol
        vector_index: VectorIndex,              # core.interfaces.VectorIndex — 已有 Protocol
        embedder: Embedder,                     # core.interfaces.Embedder — 已有 Protocol
        registry: SchemaRegistry,               # extraction.schemas.registry — 已有
        uri_resolver: URIResolver,              # core.uri_resolver — 已有
        context_writer: ContextWriter,          # commit.context_writer — 已有（Layer 3 写入需要）
    ):
        ...
```

**关键设计决策**：BookKeeper 在构造时接收所有可能的依赖，但**每次 `run()` 时由调用方通过 `tools` 参数决定实际使用哪些**。构造时全量注入，运行时按需激活。

这与现有 `ExtractionReActLoop` 的区别：后者把 `fs`、`registry` 等在构造时绑定后固定使用。BookKeeper 的依赖更多，但使用模式相同——构造一次，多次运行。

#### run() — 核心接口

```python
@dataclass
class BookKeeperTask:
    """BookKeeper 的一次运行任务。"""
    instruction: str                              # 自然语言指令（替代 system_prompt）
    tools: list[ToolDefinition]                   # 本次运行可用的工具集
    context: dict[str, Any] | None = None         # 附加上下文（对话文本 / pipeline 结果 / 候选列表）
    max_iterations: int = 5
    timeout_seconds: float = 60.0

@dataclass
class ToolCallTrace:
    """单次工具调用追踪。"""
    tool_name: str
    tool_input: dict
    tool_output: dict | str
    latency_ms: float

@dataclass
class BookKeeperResult:
    """BookKeeper 一次运行的输出。"""
    # 结构化输出 — LLM 最终产出的操作列表
    # 调用方根据场景解读这些操作
    actions: list[dict]            # [{"type": "extract_profile", "input": {...}}, ...]
                                   # 或 [{"type": "found_related", "uris": [...]}, ...]
                                   # 或 [{"type": "merge", "uris": [...], "reason": "..."}, ...]

    # 副产物
    read_uris: set[str]            # 本次运行读取过的 URI（供调用方做安全检查）
    trace: list[ToolCallTrace]     # 工具调用追踪
    iterations: int                # 实际迭代次数
    elapsed_seconds: float

    # 状态
    completed: bool = True         # False = 超时或被中断
    finish_reason: str = "done"    # "done" | "timeout" | "max_iterations" | "error"
    error: str | None = None

class BookKeeper:
    def run(
        self,
        task: BookKeeperTask,
        ctx: RequestContext,
    ) -> BookKeeperResult:
        """执行一次 ReAct 循环。

        Args:
            task: 运行任务（指令 + 工具 + 配置）
            ctx: 租户隔离上下文（account_id + owner_space）

        Returns:
            BookKeeperResult — 结构化结果 + 追踪信息
        """
        ...
```

**设计说明**：

1. `actions` 是通用操作列表，调用方按场景解读。不强制输出类型，因为不同场景的输出结构差异大：
   - 检索场景 → `[{"type": "search_result", "uris": [...], "reason": "..."}]`
   - 抽取场景 → `[{"type": "extract_profile", "input": {...}}]`（即现有 `CandidateMemory` 的来源）
   - Dreaming Light → `[{"type": "candidate_merge", "uris": [...], "evidence": "..."}]`
   - Dreaming Deep → `[{"type": "merge", "uris": [...], "reason": "..."}]`

2. `instruction` 替代了 `system_prompt`——BookKeeper 内部负责将 instruction 组装为完整的 system prompt（包含工具描述、上下文格式等）。调用方只需描述任务意图。

3. `RequestContext` 在每次 `run()` 时传入，同一个 BookKeeper 实例可以为不同租户运行。

### 7.3 上游接口 — 调用方怎么用 BookKeeper

#### ① LightDreaming 调用 BookKeeper

这是需要与同事对齐的关键接口。

```python
# dreaming/light_dreaming.py（同事负责）

from bookkeeper import BookKeeper, BookKeeperTask, BookKeeperResult

class LightDreaming:
    def __init__(self, keeper: BookKeeper, candidate_store: DreamingCandidateStore):
        self._keeper = keeper
        self._store = candidate_store

    def run(self, ctx: RequestContext) -> DreamingLightResult:
        """执行 Light Dreaming。"""

        # 1. 构建 BookKeeper 任务
        task = BookKeeperTask(
            instruction=self._build_instruction(ctx),
            tools=self._build_tools(),
            context={"time_range_days": 7},
            max_iterations=5,
            timeout_seconds=60.0,
        )

        # 2. 调用 BookKeeper
        result = self._keeper.run(task, ctx)

        # 3. 解读 actions，持久化候选
        candidates = self._parse_candidates(result.actions)
        self._store.save(ctx.account_id, candidates)

        return DreamingLightResult(
            candidates=candidates,
            trace=result.trace,
            iterations=result.iterations,
        )

    def _build_instruction(self, ctx: RequestContext) -> str:
        """Light Dreaming 的任务指令。

        ⚠️ LightDreaming 负责编写这个 prompt，BookKeeper 不关心内容。
        """
        return (
            "你是记忆管理员。扫描这个用户最近 7 天的记忆，发现以下模式：\n"
            "1. 被多次召回但缺少相互引用的记忆\n"
            "2. 同 category 下可能重复的记忆\n"
            "3. 可以聚合为 pattern 的 event 群\n"
            "4. 可能过期或冲突的 profile 字段\n"
            "输出候选整理任务列表。"
        )

    def _build_tools(self) -> list[ToolDefinition]:
        """Light Dreaming 需要的工具集。

        ⚠️ LightDreaming 决定传什么工具，BookKeeper 不关心用途。
        """
        return [
            # Layer 1 — 只读
            READ_TOOL, LIST_TOOL, GET_RELATIONS_TOOL, GET_ACCESS_STATS_TOOL,
            # Layer 2 — 分析
            FIND_DUPLICATES_TOOL, FIND_RELATED_TOOL, COMPUTE_IMPORTANCE_TOOL,
        ]

    def _parse_candidates(self, actions: list[dict]) -> list[DreamingCandidate]:
        """从 BookKeeper 输出的 actions 中解析候选。

        ⚠️ actions 的格式由 LightDreaming 的 instruction 决定。
        BookKeeper 只是忠实地传递 LLM 的输出。
        """
        candidates = []
        for action in actions:
            if action.get("type") in ("candidate_merge", "candidate_archive", "candidate_aggregate"):
                candidates.append(DreamingCandidate(
                    candidate_type=action["type"],
                    uris=action.get("uris", []),
                    reason=action.get("reason", ""),
                    evidence=action.get("evidence", ""),
                ))
        return candidates
```

**关键接口契约**：

| 契约项 | 说明 |
|--------|------|
| LightDreaming 提供 | `instruction`（prompt）、`tools`（工具列表）、`_parse_candidates`（输出解读） |
| BookKeeper 保证 | 按给定的 instruction + tools 执行 ReAct 循环，原样返回 LLM 的 actions |
| 解耦点 | BookKeeper 不理解"light dreaming"概念，只理解"任务"。LightDreaming 负责所有领域逻辑 |
| 隔离点 | `RequestContext` 保证租户隔离。LightDreaming 不需要关心 RLS |

#### ② /compose handler 调用 BookKeeper

```python
# retrieval/react_retriever.py

class ReactRetriever:
    def __init__(self, keeper: BookKeeper):
        self._keeper = keeper

    def search(
        self,
        query: str,
        pipeline_result: SearchMemoryResult,
        ctx: RequestContext,
    ) -> list[RetrievedBlock]:
        """pipeline 结果不足时，用 BookKeeper 追加搜索。"""
        task = BookKeeperTask(
            instruction=(
                f"原始查询: {query}\n\n"
                f"已有的搜索结果（不够完整）:\n{self._format_pipeline_result(pipeline_result)}\n\n"
                "请进一步搜索，找到与原始查询相关的记忆。"
            ),
            tools=[SEARCH_TOOL, READ_TOOL, LIST_TOOL, GET_RELATIONS_TOOL, FIND_RELATED_TOOL],
            context={"query": query, "existing_hits": [b.uri for b in pipeline_result.hits]},
            max_iterations=3,
            timeout_seconds=15.0,
        )
        result = self._keeper.run(task, ctx)
        return self._actions_to_blocks(result.actions)
```

#### ③ /after_turn handler 调用 BookKeeper

```python
# extraction/react_loop.py（改造后）

class ExtractionReActLoop:
    """向后兼容的抽取 ReAct 循环，内部委托给 BookKeeper。"""

    def __init__(self, keeper: BookKeeper, ...):
        self._keeper = keeper

    def run(self, conversation_text: str, ctx: RequestContext, ...) -> ReActResult:
        task = BookKeeperTask(
            instruction=self._build_extraction_prompt(conversation_text),
            tools=self._extraction_tools,  # 现有 extract_* + read/list
            context={"conversation": conversation_text},
            max_iterations=self._max_iterations,
            timeout_seconds=self._timeout_seconds,
        )
        bk_result = self._keeper.run(task, ctx)

        # 转换为现有 ReActResult 格式（向后兼容）
        candidates = [self._parse_candidate(a) for a in bk_result.actions if a.get("type", "").startswith("extract_")]
        return ReActResult(
            candidates=candidates,
            tools_used=[{"tool": t.tool_name, "params": t.tool_input, "result": t.tool_output} for t in bk_result.trace],
            iterations=bk_result.iterations,
            read_uris=bk_result.read_uris,
        )
```

### 7.4 下游接口 — BookKeeper 调用什么

BookKeeper 的工具实现对 `core/interfaces.py` 中已有 Protocol 的依赖：

| 工具 | 依赖的 Protocol | 调用的方法 |
|------|----------------|-----------|
| `read(uri)` | `ContextFS` | `read_node(uri, ctx)` |
| `list(uri)` | `ContextFS` | `list_children(uri, ctx)` |
| `get_relations(uri)` | `RelationStore` | `get_edges(uri, ctx)` / `get_one_hop(uri, ctx)` |
| `get_access_stats(uri)` | `ContextFS` | `read_node(uri, ctx)` → 从 metadata 取 |
| `search(query, filters)` | `VectorIndex` + `Embedder` | `embed_texts([query])` → `search_by_vector(...)` |
| `extract_*(fields)` | `SchemaRegistry` | `parse_tool_call(name, input, registry)` → `CandidateMemory` |
| `update_node(uri, fields)` | `ContextWriter` | `write_candidate(candidate, ctx)` |
| `merge_nodes(uris)` | `ContextWriter` + `ContextFS` | 读取多个节点 → LLM 合并 → 写入 |
| `archive_node(uri)` | `ContextFS` | `archive_node(uri, ctx)` |
| `create_relation(...)` | `RelationStore` | `upsert_edges([edge], ctx)` |

**新增工具（Layer 2 分析工具）的实现依赖**：

| 工具 | 实现方式 | 新依赖？ |
|------|---------|---------|
| `find_duplicates(category)` | `VectorIndex.search_by_vector` + 相似度阈值 | 否，复用现有 |
| `find_related(uri, max_hops)` | `RelationStore.get_one_hop` 递归 | 否，复用现有 |
| `compute_importance(uri)` | 读取 metadata 中的 access stats + 关系数计算 | 否，纯计算 |
| `aggregate(uris)` | 读取多个节点 → LLM 摘要 | 需要新逻辑但依赖现有接口 |
| `detect_conflicts(uris)` | 读取多个节点 → LLM 判断 | 需要新逻辑但依赖现有接口 |

### 7.5 边界规范

#### BookKeeper 不做的事

| 禁止 | 原因 | 谁负责 |
|------|------|--------|
| 解读 actions 的语义 | 不同场景的输出含义不同 | 调用方（LightDreaming / ReactRetriever / ExtractionReActLoop） |
| 管理 prompt 模板 | prompt 是调用方的领域知识 | 调用方 |
| 持久化候选任务 | 这是 Dreaming 的编排职责 | DreamingCandidateStore |
| 调度多个租户 | 这是系统级职责 | DreamingScheduler |
| 决定使用哪些工具 | 权限边界由调用方控制 | 调用方 |
| 直接操作数据库 | 必须通过 ContextFS / ContextWriter | — |

#### 调用方不需要关心的事

| 隐藏细节 | 说明 |
|----------|------|
| LLM 调用细节 | 调用方不关心 prompt 怎么拼、temperature 多少 |
| 工具调用循环 | 调用方不关心迭代了几次、调了什么工具 |
| 安全检查（refetch） | BookKeeper 内部处理，调用方只看最终结果 |
| 租户隔离 | RequestContext 穿透到底层，调用方无需额外处理 |

#### 模块间数据流图

```
LightDreaming（同事负责）          BookKeeper（你负责）
┌──────────────────────┐          ┌──────────────────────┐
│                      │          │                      │
│  产出：               │          │  接收：               │
│  - instruction       │ ───────→ │  - BookKeeperTask    │
│  - tools 列表        │          │  - RequestContext    │
│  - max_iterations    │          │                      │
│  - timeout           │          │  返回：               │
│                      │ ←─────── │  - BookKeeperResult  │
│  消费：               │          │    (actions + trace) │
│  - actions → 候选    │          │                      │
│  - trace → 日志      │          │  依赖（构造时注入）：  │
│                      │          │  - LLM               │
│  不关心：             │          │  - ContextFS         │
│  - LLM 怎么调用      │          │  - VectorIndex       │
│  - 迭代了几次        │          │  - RelationStore     │
│  - prompt 怎么拼     │          │  - Embedder          │
│                      │          │  - SchemaRegistry    │
│                      │          │  - URIResolver       │
│                      │          │  - ContextWriter     │
└──────────────────────┘          └──────────────────────┘
       ↑                                  ↑
       │                                  │
       │ DreamingCandidateStore           │ core/interfaces.py
       │ (持久化候选)                      │ (已有 Protocol)
       │                                  │
       └──────────────────────────────────┘
```

### 7.6 对接 checklist（与同事对齐）

LightDreaming 同事需要确认：

1. **输入格式**：`BookKeeperTask` 的 `instruction` 和 `tools` 字段是否满足需求
2. **输出格式**：`BookKeeperResult.actions` 是 `list[dict]`，LightDreaming 自行解读。LLM 输出的 actions 格式是否需要 BookKeeper 侧做结构化约束？（如强制 JSON schema）
3. **工具列表**：Light Dreaming 需要哪些工具？当前 Layer 1 + Layer 2 是否足够？
4. **错误处理**：`BookKeeperResult.completed=False` 时 LightDreaming 应该怎么处理？部分结果是否可用？
5. **上下文传递**：`BookKeeperTask.context` 是否需要更结构化的字段？（如 `time_range`、`target_uris`）
6. **工具实现**：`find_duplicates`、`compute_importance` 等新工具由谁实现？BookKeeper 模块内还是单独的工具模块？

---

## 八、与现有代码的关系

### 改动范围

```
bookkeeper/                    → 新增模块
  ├── __init__.py
  ├── keeper.py               → BookKeeper 通用引擎（从 react_loop.py 泛化）
  ├── config.py               → BookKeeperConfig + 预定义配置
  └── tools/                  → 工具定义（分层）
      ├── read_only.py        → Layer 1
      ├── analysis.py         → Layer 2
      └── write.py            → Layer 3

extraction/
  ├── react_loop.py           → 内部委托给 BookKeeper（extraction config）
  └── tools.py                → 基本不变，eager mode 独立使用

retrieval/
  ├── react_retriever.py      → 新增：包装 BookKeeper（retrieval config）
  └── pipeline.py             → 增加 BookKeeper fallback 逻辑

dreaming/                      → 新增模块
  ├── orchestrator.py         → DreamingOrchestrator
  ├── scheduler.py            → DreamingScheduler（系统级租户调度）
  ├── candidate_store.py      → 候选任务持久化（dreaming_candidates 表）
  ├── prompts.py              → Light/Deep phase 的 prompt 模板
  └── tool_configs.py         → Light/Deep phase 的工具集配置
```

### 数据流总图（目标状态）

```
┌──────────────────── 实时 API ─────────────────────┐
│                                                    │
│  POST /compose (检索)                               │
│    → pipeline.run() → 结果评估                      │
│    → [不足] BookKeeper.run(retrieval_config)        │
│    → 合并返回                                       │
│                                                    │
│  POST /after_turn (存储)                            │
│    → eager extraction → 直接写入                    │
│    → [需要 react] BookKeeper.run(extraction_config) │
│    → 带冲突检测的写入                                │
│                                                    │
└────────────────────────────────────────────────────┘

┌──────────────────── 后台调度 ─────────────────────┐
│                                                    │
│  DreamingScheduler.run_scheduled()                 │
│    → 遍历活跃租户                                   │
│    → 为每个租户启动 DreamingOrchestrator             │
│       ├── Light: BookKeeper.run(light_config)      │
│       │   → 候选 → dreaming_candidates 表           │
│       └── Deep: BookKeeper.run(deep_config)        │
│           → 写入操作 → ContextWriter → DB           │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## 九、待决问题

### Q1：BookKeeper 是否应该是通用的 ReAct 引擎？

当前设计是针对 memory 场景特化的（固定工具类型、固定安全机制）。是否应该做成通用 ReAct 引擎（类似 langchain Agent），让 BookKeeper 成为更上层的特化？

**倾向**：保持特化。oG-Memory 的场景足够具体，通用化会增加不必要的复杂度。

### Q2：Retrieval BookKeeper 的成本控制

复杂检索场景下，BookKeeper 的多轮 LLM 调用成本是否可接受？是否需要：
- 每次查询的 token 预算上限
- 仅在 API 调用方显式请求时才触发（如 `mode=deep`）
- 结果缓存（相似查询复用之前的 BookKeeper 结果）

### Q3：Dreaming Light 和 Deep 的调度频率

- Light：每小时？每天？
- Deep：每天一次？每周？
- Light 和 Deep 是否必须在同一次运行中执行？

### Q4：Deep Phase 的写入安全

Deep Phase 会更新已有节点，如何防止错误更新？

- **方案 D1**：干运行模式 — Deep Phase 输出"建议操作"，记录到表中，人工确认后执行
- **方案 D2**：自动执行 + 审计日志 — 记录 before/after snapshot，支持管理员回滚
- **方案 D3**：渐进信任 — 初期要求确认，随着准确率提升逐步放开

### Q5：团队级 Dreaming 的触发方式

团队共享空间的记忆整理：
- 由团队管理员通过 API 手动触发？
- 自动调度但频率低于个人空间？
- 需要团队成员的 visible_owner_spaces 联合授权？

### Q6：BookKeeper 的 LLM 资源消耗

多租户同时 dreaming 时，LLM 调用的并发和成本：
- 是否需要 LLM 调用配额（per-tenant rate limit）？
- BookKeeper 是否可以使用更便宜的模型（如用 haiku 做 light，sonnet 做 deep）？
- 是否需要区分"在线 BookKeeper"（实时场景，用高端模型）和"离线 BookKeeper"（dreaming，用便宜模型）？

### Q7：候选任务持久化的 schema

`dreaming_candidates` 表需要存储什么：
- 候选类型（merge / aggregate / archive / update_relation）
- 涉及的 URI 列表
- LLM 产出的理由/证据
- 状态（pending / approved / done / rejected）
- 是否需要人工审批环节？
