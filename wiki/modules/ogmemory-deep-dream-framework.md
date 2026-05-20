# oG-Memory Deep Dream Framework 设计文档

> 版本: v1.0-draft | 日期: 2026-05-20 | 状态: 设计阶段

## 设计思路

### 为什么采用可组合的三阶段架构？

当前 AI Agent 记忆整合领域存在多种实现方式，尚未形成共识性最佳实践：

- **OpenClaw Dreaming**: Light/REM/Deep 三阶段，从短期召回存储中评分晋升
- **Stanford Generative Agents**: Reflection 机制（生成问题 → 搜索 → 回答）
- **MemGPT/Letta**: 分层记忆 + Agent 自管理（core/archival/recall）
- **Mem0**: 可配置衰减率 + 时间/频率/重要性评分
- **聚类压缩**: 按语义相似度聚类后摘要
- **时间窗口压缩**: 按时间分块压缩历史

这些方案各有优劣，适用场景不同，且仍处于快速演进阶段。为适应这种不确定性，Deep Dream 采用**可组合的三阶段流水线架构**：

```
Acquire（获取） → Process（处理） → Output（输出）
     ↓                  ↓               ↓
  多策略并行        多策略并行        标准动作
```

**核心优势**：

1. **快速迭代验证**: MVP 只需实现最简单的组合（LightRemAcquire + SimpleProcess），验证效果后可逐步替换或增加策略，无需重写整体架构

2. **策略独立演进**: 每个阶段独立变化，新增策略不影响其他阶段。例如：
   - Acquire 新增 Light+REM 输入源 → 只需新增 LightRemAcquire
   - Process 尝试聚类压缩 → 只需新增 ClusterSummarize
   - Output 增加合并逻辑 → 只需调整 OutputExecutor

3. **多策略组合实验**: 同一阶段可并行多个策略，观察不同组合效果。例如：
   ```python
   acquire = [RecentAcquire(100), ImportanceAcquire(20)]  # 最近 + 高重要性
   process = [StanfordReflection(), ClusterSummarize()]   # 反思 + 聚类并行
   ```

4. **向后兼容扩展**: 输出阶段使用现有 WriteAction 标准，复用 ContextWriter 存储逻辑，不破坏现有系统

5. **便于对比研究**: 不同策略组合可形成对照实验，量化评估哪种方式更有效

这种架构借鉴了 Unix Pipeline 的设计哲学：每个组件专注单一职责，组件间通过标准接口连接，组合方式由配置决定。

---

## 概述

Deep Dream 是 oG-Memory 的记忆整合与优化机制，灵感来自 OpenClaw Dreaming 和人类睡眠时的记忆巩固过程。

借鉴 OpenClaw 的 Light/REM/Deep 三阶段设计，但实现上有重要区别：
- **OpenClaw**: Light 和 REM 是内部模块，产出写入短期召回存储 `.dreams/`
- **oG-Memory**: Light 和 REM 由其他模块独立开发，产出作为 Deep Dream 的输入

Deep Dream 在 oG-Memory 中对应 OpenClaw 的 **Deep Phase**：读取上游阶段输出、评分筛选、生成新的记忆。

**核心目标**:
- 从大量具体记忆中提炼高层次洞察
- 聚合相似记忆，减少冗余
- 建立记忆间的关联关系
- 保留完整溯源链

**设计原则**:
- 三阶段流水线（Acquire → Process → Output）
- 每阶段策略可插拔、可组合
- 多策略并行执行，合并结果
- 输出使用标准动作，复用现有存储机制

---

## 架构总览

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          Deep Dream Pipeline                              │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│   ┌───────────────────┐    ┌───────────────────┐    ┌─────────────────┐  │
│   │    ACQUIRE        │───▶│    PROCESS        │───▶│    OUTPUT       │  │
│   │   (多策略并行)     │    │   (多策略并行)     │    │  (标准动作)     │  │
│   └───────────────────┘    └───────────────────┘    └─────────────────┘  │
│           │                        │                      │              │
│           ▼                        ▼                      ▼              │
│   [AcquireStrategy 1]      [ProcessStrategy 1]     WriteAction          │
│   [AcquireStrategy 2] ───▶ [ProcessStrategy 2] ───▶ ├─ CREATE           │
│   [AcquireStrategy 3]      [ProcessStrategy 3]     ├─ MERGE             │
│                            ...                     ├─ ARCHIVE           │
│                                                    └─ DELETE            │
│                                                                           │
│   输入合并: results = [s1.acquire(), s2.acquire(), ...]                  │
│   处理合并: outputs = [p1.process(), p2.process(), ...]                 │
│   输出执行: context_writer.execute(dream_outputs)                        │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 输出格式设计

### Category: dream

新增 `dream` category，与 `ProvenanceResolver` 预留的 `dream` source_type 对应。

**URI 格式**:
```
ctx://{account}/users/{user}/memories/dream/{dream_id}/
```

**dream_id 格式**:
```
{dream_type}_{timestamp}_{content_hash}

示例:
- reflection_20260520_a1b2c3
- cluster_summary_20260521_c4d5e6
```

### DreamOutput 数据结构

```python
@dataclass
class DreamOutput:
    """Deep Dream 处理输出
    
    可转换为 CandidateMemory 或 ContextNode，通过现有写入机制存储。
    """
    
    # === 内容字段 ===
    content: str           # 生成的反思/摘要内容（完整）
    abstract: str          # ≤200 字符摘要（用于索引）
    overview: str          # 结构化概述（可选，用于快速浏览）
    
    # === 分类字段 ===
    dream_type: str        # reflection | cluster_summary | compression | ...
    importance: float      # 重要性权重（dream 输出通常 > 1.0）
    
    # === 溯源字段 ===
    provenance_ids: list[str]   # 溯源链：来源记忆 URI + 继承的 provenance_ids
    
    # === 写入动作 ===
    action: str            # CREATE | MERGE | ARCHIVE | DELETE
    
    # === 额外元数据 ===
    extra_metadata: dict   # 扩展字段
```

### 溯源机制

Dream 输出的 `provenance_ids` 包含完整的溯源链：

1. **直接来源**: 被处理的记忆，通过 `ProvenanceResolver.build_id("memory", uri)` 生成
2. **继承溯源**: 来源记忆自身的 `provenance_ids`（追溯更早的来源，如 archive）

Provenance ID 格式中，`source_id` 字段即为记忆 URI，无需单独存储。

**Provenance ID 格式**:
```
prov:1:{source_type}:{urlencoded(source_id)}:{urlencoded(detail)}

示例:
- prov:1:dream:reflection_20260520_a1b2c3:              (dream 输出本身)
- prov:1:memory:ctx%3A%2F%2Facme%2Fmemories%2Fpref%2Fcoding:  (来源记忆 URI 在 source_id 中)
- prov:1:archive:20260513_100000_a1b2:msg_a3f8               (继承的 archive 来源)
```

### metadata 存储结构

ContextNode.metadata 字段：

```json
{
    "provenance_ids": [
        "prov:1:memory:ctx://acme/memories/pref/coding_style:",
        "prov:1:memory:ctx://acme/memories/entity/rust_project:",
        "prov:1:archive:20260513_100000_a1b2:msg_a3f8"
    ],
    "dream_type": "reflection",
    "importance": 1.5,
    "created_at": "2026-05-20T15:30:00Z",
    "processor": "stanford_reflection",
    "acquire_strategy": "recent_100"
}
```

---

## 三阶段接口设计

### Phase 1: Acquire（获取）

```python
class AcquireStrategy(Protocol):
    """从记忆存储中获取待处理的记忆
    
    多策略并行执行，结果合并。
    """
    
    def acquire(
        self,
        ctx: RequestContext,       # 请求上下文（account_id, user_id 等）
        context_fs: ContextFS,     # 文件系统接口（读取 ContextNode）
        vector_index: VectorIndex, # 向量索引（语义搜索）
    ) -> list[ContextNode]:
        """获取待处理的记忆列表
        
        Returns:
            ContextNode 列表，包含完整内容和 metadata
        """
        ...
```

**基础策略**:

| 策略 | 功能 | 参数 |
|------|------|------|
| `RecentAcquire` | 获取最近 N 条记忆 | `limit`, `category_filter` |
| `SearchAcquire` | 基于关键词搜索 | `query`, `top_k` |
| `TimeRangeAcquire` | 获取时间范围内记忆 | `start_time`, `end_time` |
| `ImportanceAcquire` | 获取高重要性记忆 | `min_importance`, `limit` |
| `CategoryAcquire` | 获取特定 category 记忆 | `categories` |

**组合策略**:

```python
class CompositeAcquire:
    """组合多个获取策略，合并结果并去重"""
    
    def __init__(self, strategies: list[AcquireStrategy]):
        self.strategies = strategies
    
    def acquire(self, ctx, context_fs, vector_index) -> list[ContextNode]:
        # 执行所有策略，合并结果
        results = []
        for strategy in self.strategies:
            nodes = strategy.acquire(ctx, context_fs, vector_index)
            results.extend(nodes)
        return self._deduplicate(results)  # 按 URI 去重
```

### Phase 2: Process（处理）

```python
class ProcessStrategy(Protocol):
    """处理获取的记忆，生成 DreamOutput
    
    多策略并行执行，结果合并。
    """
    
    def process(
        self,
        memories: list[ContextNode],   # Acquire 阶段输出
        ctx: RequestContext,           # 请求上下文
        llm: LLM,                      # LLM 接口（反思/生成）
        vector_index: VectorIndex,     # 向量索引（二次搜索）
    ) -> list[DreamOutput]:
        """处理记忆，生成输出列表
        
        Returns:
            DreamOutput 列表，包含内容、溯源、写入动作
        """
        ...
```

**基础策略**:

| 策略 | 功能 | dream_type |
|------|------|------------|
| `StanfordReflection` | Stanford 反思机制：生成问题 → 搜索 → 回答 | `reflection` |
| `ClusterSummarize` | 聚类后生成类别摘要 | `cluster_summary` |
| `DirectCompress` | 直接压缩/总结 | `compression` |
| `TimeWindowCompress` | 按时间窗口压缩 | `time_compression` |

**组合策略**:

```python
class ParallelProcess:
    """多个处理策略并行执行，合并结果"""
    
    def __init__(self, strategies: list[ProcessStrategy]):
        self.strategies = strategies
    
    def process(self, memories, ctx, llm, vector_index) -> list[DreamOutput]:
        # 并行执行所有策略，合并结果
        results = []
        for strategy in self.strategies:
            outputs = strategy.process(memories, ctx, llm, vector_index)
            results.extend(outputs)
        return results
```

### Phase 3: Output（输出）

```python
class OutputExecutor:
    """将 DreamOutput 转换并执行写入
    
    复用现有的 ContextWriter 和 ContextFS。
    """
    
    def __init__(self, context_writer: ContextWriter, uri_resolver: URIResolver):
        self.writer = context_writer
        self.uri_resolver = uri_resolver
    
    def execute(
        self,
        outputs: list[DreamOutput],
        ctx: RequestContext,
    ) -> list[WritePlan]:
        """执行输出动作
        
        流程:
        1. 将 DreamOutput 转换为 CandidateMemory
        2. 通过 ContextWriter.write_candidate() 写入
        3. 返回 WritePlan 列表（记录执行结果）
        
        Returns:
            WritePlan 列表，包含 action 和 target_uri
        """
        ...
```

---

## MVP 实现：简单晋升机制

### 设计背景

借鉴 OpenClaw Dreaming 的 Deep Phase 思路，但简化实现：

**OpenClaw Deep Phase**:
- 从短期召回存储 `.dreams/` 读取候选条目
- 六信号评分（frequency/relevance/diversity/recency/consolidation/conceptual）
- Phase Reinforcement 加成（Light/REM 命中次数）
- 阈值门控后晋升到 MEMORY.md

**oG-Memory Deep Dream MVP**:
- 从 Light+REM 模块输出读取候选（由其他同事开发）
- 简化评分或直接晋升
- 生成新的 `dream` category 记忆

**与 OpenClaw 的关键区别**:
- OpenClaw Light/REM 是内部模块，产出写入 `.dreams/`
- oG-Memory Light+REM 独立开发，产出作为 Deep Dream 输入
- OpenClaw 晋升到 MEMORY.md（追加现有内容）
- oG-Memory 创建新的 dream category 记忆（独立 URI）

### LightRemAcquire

```python
class LightRemAcquire:
    """从 Light+REM 模块输出获取候选记忆
    
    Light+REM 模块由其他同事开发，产出格式待定义。
    预期输出可能包含:
    - 高信号候选条目列表
    - 条目的评分信息
    - 条目的来源溯源
    """
    
    def __init__(self, input_source: str = "default"):
        """
        Args:
            input_source: 输入源标识（支持多 Light+REM 实例）
        """
        self.input_source = input_source
    
    def acquire(self, ctx, context_fs, vector_index) -> list[ContextNode]:
        # === 从 Light+REM 输出获取候选 ===
        # 1. 读取 Light+REM 模块的输出文件或 API
        # 2. 解析候选条目（格式待 Light+REM 模块定义）
        # 3. 转换为 ContextNode 格式
        # 4. 返回候选列表
        
        # 注：Light+REM 输出格式需与 Light+REM 开发者协调
        ...
```

### SimplePromotionProcess

```python
class SimplePromotionProcess:
    """简化晋升处理
    
    MVP 版本不做复杂评分，直接将 Light+REM 输出转换为 dream 记忆。
    后续版本可添加评分机制、去重、合并等。
    """
    
    def __init__(self, min_score_threshold: float = 0.0):
        """
        Args:
            min_score_threshold: 最低评分阈值（MVP 默认 0，即全部晋升）
        """
        self.min_score_threshold = min_score_threshold
    
    def process(self, memories, ctx, llm, vector_index) -> list[DreamOutput]:
        outputs = []
        for memory in memories:
            # === 简化处理：直接转换 ===
            # MVP 不做额外处理，后续可添加:
            # - 内容摘要/压缩
            # - 相似记忆合并
            # - 重复/错误记忆检测
            
            outputs.append(DreamOutput(
                content=memory.content,
                abstract=memory.abstract,
                overview=memory.overview,
                dream_type="promotion",  # 从 Light+REM 晋升
                importance=memory.metadata.get("score", 1.0),
                provenance_ids=memory.metadata.get("provenance_ids", []),
                action=WriteAction.CREATE,
            ))
        
        return outputs
```

### 扩展能力：删除/更新重复记忆

MVP 不实现删除和更新，但架构支持后续扩展：

**依赖 ContextWriter 能力**:
- `WriteAction.DELETE`: 删除重复/错误记忆
- `WriteAction.MERGE`: 合并相似记忆
- `WriteAction.ARCHIVE`: 归档过期记忆

**潜在实现思路**:
```python
class DeduplicationProcess:
    """检测并处理重复记忆
    
    流程:
    1. 对候选记忆进行语义相似度检测
    2. 发现与现有记忆高度相似（>threshold）
    3. 生成 DreamOutput:
       - action=DELETE 删除重复的旧记忆
       - action=MERGE 合并相似记忆
       - 或 action=CREATE 创建新的聚合记忆
    
    依赖 ContextWriter 提供的 DELETE/MERGE 能力。
    """
    
    def process(self, memories, ctx, llm, vector_index) -> list[DreamOutput]:
        # 1. 对每个候选记忆，搜索相似记忆
        # 2. 根据相似度决定 action
        # 3. 构建 DreamOutput
        ...
```

---

## Pipeline 编排器

```python
class DeepDreamPipeline:
    """Deep Dream 流水线编排器
    
    组合三阶段策略，执行完整流程。
    """
    
    def __init__(
        self,
        acquire_strategies: list[AcquireStrategy],
        process_strategies: list[ProcessStrategy],
        output_executor: OutputExecutor,
    ):
        """
        Args:
            acquire_strategies: 获取策略列表（并行执行，合并结果）
            process_strategies: 处理策略列表（并行执行，合并结果）
            output_executor: 输出执行器
        """
        self.acquire = CompositeAcquire(acquire_strategies)
        self.process = ParallelProcess(process_strategies)
        self.output = output_executor
    
    def run(
        self,
        ctx: RequestContext,
        context_fs: ContextFS,
        vector_index: VectorIndex,
        llm: LLM,
    ) -> DeepDreamReport:
        """执行完整的 Deep Dream 流程
        
        流程:
        1. Acquire: 多策略获取记忆，合并去重
        2. Process: 多策略处理记忆，合并输出
        3. Output: 执行写入动作
        
        Returns:
            DeepDreamReport，包含各阶段统计和执行结果
        """
        # Phase 1: Acquire
        memories = self.acquire.acquire(ctx, context_fs, vector_index)
        
        # Phase 2: Process
        outputs = self.process.process(memories, ctx, llm, vector_index)
        
        # Phase 3: Output
        plans = self.output.execute(outputs, ctx)
        
        return DeepDreamReport(
            acquired_count=len(memories),
            processed_outputs=len(outputs),
            executed_plans=plans,
            dream_outputs=outputs,  # 保留原始输出用于调试
        )
```

```python
@dataclass
class DeepDreamReport:
    """Deep Dream 执行报告"""
    
    acquired_count: int                 # 获取的记忆数量
    processed_outputs: int              # 处理生成的输出数量
    executed_plans: list[WritePlan]     # 执行的写入计划
    dream_outputs: list[DreamOutput]    # 原始输出（可选，用于调试）
    timestamp: datetime                 # 执行时间
    duration_ms: int                    # 执行耗时
```

---

## HTTP 端点

### POST /api/v1/deepdream

触发 Deep Dream 处理。

**Request**:
```json
{
    "config": {
        "acquire": [
            {"type": "recent", "params": {"limit": 100}}
        ],
        "process": [
            {"type": "stanford_reflection", "params": {"num_questions": 3}}
        ]
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
        "acquired_count": 100,
        "processed_outputs": 3,
        "plans": [
            {"action": "create", "uri": "ctx://acme/users/alice/memories/dream/reflection_20260520_a1b2c3/"},
            {"action": "create", "uri": "ctx://acme/users/alice/memories/dream/reflection_20260520_c4d5e6/"},
            {"action": "create", "uri": "ctx://acme/users/alice/memories/dream/reflection_20260520_e7f8g9/"}
        ],
        "duration_ms": 15234
    }
}
```

---

## 需修改的现有代码

### 1. core/validation.py

```python
# 添加 dream 到有效 category 列表
valid_categories = [
    "profile", "preference", "entity", "event",
    "case", "pattern", "skill", "tool", "dream"  # 新增
]
```

### 2. core/uri_resolver.py

```python
# 支持 dream category 的 URI 解析
# 格式: ctx://{account}/users/{user}/memories/dream/{dream_id}/
```

### 3. commit/merge_policies.py

```python
class DreamPolicy:
    """Dream 记忆的写入策略
    
    dream 类型总是 CREATE，不合并。
    URI 由 dream_id 生成，确保唯一性。
    """
    
    def plan(self, candidate: CandidateMemory, ctx: RequestContext) -> WritePlan:
        # dream 总是创建新节点
        return WritePlan(
            action=WriteAction.CREATE,
            target_uri=self._resolve_uri(candidate, ctx),
        )
```

### 4. extraction/tool_schemas.py（可选）

```python
class DreamSchema:
    """Dream 记忆的提取 schema
    
    dream_type: reflection | cluster_summary | compression | ...
    """
    
    category: ClassVar[str] = "dream"
    
    dream_type: str  # reflection, cluster_summary, etc.
    source_uris: list[str]
    insight_content: str
```

---

## 目录结构

```
oG-Memory/
├── deepdream/
│   ├── __init__.py              # 模块入口
│   ├── pipeline.py              # DeepDreamPipeline, DeepDreamReport
│   ├── models.py                # DreamOutput 数据结构
│   ├── strategies/
│   │   ├── __init__.py
│   │   ├── acquire.py           # AcquireStrategy, RecentAcquire, CompositeAcquire
│   │   ├── process.py           # ProcessStrategy, SimplePromotionProcess, DeduplicationProcess
│   │   └── output.py            # OutputExecutor（复用 ContextWriter）
│   └── factory.py               # PipelineFactory（从配置构建 Pipeline）
│   └── config.py                # 配置解析
│
├── server/
│   └── app.py                   # 新增 POST /api/v1/deepdream 端点
│
├── core/
│   ├── validation.py            # 修改: valid_categories 添加 "dream"
│   ├── uri_resolver.py          # 修改: 支持 dream URI
│   ├── provenance_resolver.py   # 无修改（已支持 dream）
│
├── commit/
│   └── merge_policies.py        # 新增: DreamPolicy
```

---

## 扩展路径

### 未来策略

| Phase | 策略 | 功能 |
|-------|------|------|
| Acquire | `LightDreamAcquire` | 使用 LightDream 输出作为输入 |
| Acquire | `RelationAcquire` | 获取相关记忆（通过 RelationEdge） |
| Process | `ClusterSummarize` | 聚类后生成类别摘要 |
| Process | `ReactLoopProcess` | 使用 React Loop 复杂推理 |
| Process | `TimeWindowCompress` | 按时间窗口压缩记忆 |

### 触发机制

- HTTP 端点（当前）
- LightDream 触发（未来集成）
- 定时触发（可选）
- 记忆数量阈值触发（可选）

---

## 参考

- [[OpenClaw Dreaming Mechanism]] - OpenClaw Light/REM/Deep 三阶段设计，评分算法，phase reinforcement
- [[ogmemory-architecture]] - oG-Memory 整体架构
- [[agentic-memory-evaluation-survey]] - Agent 记忆调研报告
- [Stanford Generative Agents Paper](https://arxiv.org/abs/2304.03442)
- [MemGPT/Letta GitHub](https://github.com/letta-ai/letta)

---

## 变更日志

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.1-draft | 2026-05-20 | 改用 OpenClaw-inspired MVP：LightRemAcquire + SimplePromotionProcess |
| v1.0-draft | 2026-05-20 | 初版设计（Stanford Reflection MVP）|