---
type: design
title: "oG-Memory Dreaming Enhancement Design"
status: draft
created: 2026-06-10
updated: 2026-06-10
tags: [ogmemory, dreaming, consolidation, reflection, letta, openclaw]
related:
  - "[[OpenClaw Dreaming Mechanism]]"
  - "[[OpenClaw Dreaming Prompts and LLM Usage]]"
  - "[[Letta Code Reflection Dream Prompt Deep Dive]]"
  - "[[ogmemory-deep-dream-framework]]"
---

# oG-Memory Dreaming Enhancement Design

> 设计目标：在现有 `DeepDreamReActLoop` + operational_tools + prompt-driven_tools 框架下，完善信息获取策略和处理逻辑，增加更多 dream 类型，优化 prompts。

---

## 一、背景：三个系统的 Dreaming/Reflection 机制对比

### 1.1 OpenClaw：纯规则筛选器

**核心理念**：Dreaming 是"信号筛选器"，不产生新记忆，只从已有片段中筛选高信号条目晋升。

| Phase | 用 LLM? | 实际逻辑 |
|-------|---------|----------|
| Light | ❌ | 从 `memory/*.md` + `session-corpus/` 收集片段，`dailyCount++` |
| REM | ❌ | 分析 `conceptTags` 分布 → 计算 `patternStrength` |
| Deep | ❌ | 六信号评分公式 + 阈值门控 |
| Diary | ✅ | 生成诗意叙述（不参与决策） |

**信号来源**：
- `memory_search` 每次被调用时异步写入 `short-term-recall.json`
- 记录：`recallCount`、`totalScore`、`queryHashes`、`recallDays`、`conceptTags`
- 评分：`frequency×0.24 + relevance×0.30 + diversity×0.15 + recency×0.15 + consolidation×0.10 + conceptual×0.06`

**局限**：
- 片段分块在 SQLite 索引层，Deep Phase 不做拆分
- 不改写内容，只追加 snippet + 行号引用
- OpenClaw 的存储方式（MEMORY.md + memory/*.md）需要额外的 `.dreams/` 目录记录信号

---

### 1.2 Letta：LLM 驱动的精准外科手术

**核心理念**：Reflection 是"精准外科手术"，用 LLM 审视对话历史，精准修改记忆文件。

**5 Phase 流水线**：

```
Phase 1 — Investigate（调查现有记忆）
  → 读 <parent_memory> 快照 + Bash 读外部文件

Phase 2 — Extract（提取候选学习）
  → 优先级排序：
     1. Mistakes and corrections（错误纠正）
     2. Preferences and patterns（偏好模式）
     3. New durable facts（持久事实）
     4. Contradictions（矛盾）
  → 四重过滤：Durable? Already captured? Generalizable? Temporal?

Phase 3 — Update（精准更新）
  → 层级路由：system/ vs 外部
  → 整合优先于新建
  → 矛盾在源头修正（不允许新旧并存）
  → 维护 [[path]] 交叉引用

Phase 4 — Review（检查）
  → 清理过时内容
  → 检查引用链完整性
  → 层级归位

Phase 5 — Commit（提交）
  → git add/commit/push
  → 溯源：Agent-ID + Parent-Agent-ID
```

**Prompt 设计亮点**：
- 身份边界声明："You are NOT the primary agent"
- 优先级排序明确：纠错 > 偏好 > 新事实 > 矛盾
- 矛盾必须在源头修正，不允许新旧版本并存
- "If nothing survives filtering, make no changes" —— 空 dream 是正常的

**关键设计决策**：
- 子代理只用 Bash 工具，通过 shell 操作文件
- `memoryBlocks: none` —— 不带入父 Agent 的 persona/human，避免"自我认同混淆"
- 预算硬限制：16k tokens startup context

---

### 1.3 oG-Memory 当前实现：ReAct Loop + 单一 Promotion

**架构**：
```
DeepDreamReActLoop
  ├── Operational tools（确定性函数）
  │   ├── acquire_recent → 列出近期记忆 + summaries
  │   ├── acquire_search → 向量搜索 + summaries
  │   ├── read → 读取完整内容
  │   └── acquire_search_recall_stats → SQL 查询 recall 统计
  │
  ├── Prompt-driven tool（YAML schema + validator）
  │   └── process_promotion → 提取 pattern，生成 DreamOutput
  │       → validator: check_dream_duplicate
  │
  └── Output
      → DreamOutput → CandidateMemory → WriteAPI → commit pipeline
```

**已有能力**：
- `search_recall` 表记录每次搜索召回信号（相当于 OpenClaw 的 `short-term-recall.json`）
- SQL 直接查询统计：`recall_count`、`unique_queries`、`recall_days`
- `acquire_search_recall_stats` 工具已提供候选排序
- ReAct Loop 让 LLM 自主迭代决策

**当前局限**：
- 只有一个 `process_promotion` 工具，只做 promotion 类型
- 没有错误纠正、矛盾检测、记忆整合等类型
- Prompt 比简单，缺少 Letta 的优先级排序和四重过滤
- 没有矛盾解决机制

---

## 二、设计目标

### 2.1 短期目标（Phase 1）

1. **增加 dream_type**：在 `promotion` 之外增加：
   - `reflection`：发现错误和矛盾
   - `error_correction`：修正过时记忆
   - `consolidation`：合并相似记忆

2. **优化 Prompt**：
   - 参考 Letta 的优先级排序
   - 增加四重过滤逻辑
   - 明确矛盾处理策略

3. **完善信息获取策略**：
   - 不只是 `acquire_recent`，优先使用 `acquire_search_recall_stats` 发现高信号候选
   - 增加 pattern discovery 工具（自动发现相似/矛盾）

### 2.2 长期目标（Phase 2）

- Light Dream + Deep Dream 两阶段设计（参考 OpenClaw）
- 自动触发机制（compaction-event 或 step-count）
- 记忆老化清理机制

---

## 三、具体设计

### 3.1 新增 Prompt-driven Tools

#### 3.1.1 process_reflection.yaml

**用途**：发现错误、矛盾、过时信息

```yaml
tool_name: process_reflection
category: dream
dream_type: reflection
description: |
  Identify errors, contradictions, or stale information in existing memories.
  Use after reading memories that may conflict or contradict newer observations.
fields:
  - name: topic
    type: string
    required: true
    description: "The error/contradiction topic (e.g. 'rust_version_error')"
  - name: error_type
    type: string
    required: true
    description: "One of: 'contradiction', 'stale', 'mistake', 'incorrect'"
  - name: affected_uris
    type: array
    required: true
    description: "URIs of memories that contain the error/contradiction"
  - name: correction
    type: string
    required: true
    description: "The correct information or explanation of the error"
  - name: abstract
    type: string
    required: true
    description: "Brief summary (≤200 chars)"
  - name: content
    type: string
    required: true
    description: "Detailed explanation of the error and correction"
  - name: confidence
    type: number
    required: true
    description: "Confidence in the correction (0.0-1.0)"
prompt_guidance: |
  ## process_reflection usage

  Call this tool when you identify:
  
  **Priority 1 — Mistakes and corrections**
  - Errors the system made (wrong facts, incorrect inferences)
  - User feedback pointing out mistakes
  - Failed retries that reveal misunderstandings
  
  **Priority 2 — Contradictions**
  - Two memories that state conflicting facts
  - Newer information that contradicts older memory
  - When found, explain WHICH memory is correct and WHY
  
  **Priority 3 — Stale information**
  - Facts that are no longer accurate (versions changed, project moved)
  - References to deleted files or outdated configurations
  
  **Important**: 
  - Always specify `affected_uris` — the memories that need correction
  - The `correction` field should clearly state the correct information
  - Use confidence 0.8+ for clear contradictions, 0.5 for ambiguous cases
  
post_call_validator: check_dream_duplicate
```

#### 3.1.2 process_consolidation.yaml

**用途**：合并相似记忆，提炼高层模式

```yaml
tool_name: process_consolidation
category: dream
dream_type: consolidation
description: |
  Merge similar memories into a consolidated higher-level insight.
  Use when multiple memories share the same topic but with scattered details.
fields:
  - name: topic
    type: string
    required: true
    description: "The consolidated topic name"
  - name: source_uris
    type: array
    required: true
    description: "URIs of memories to consolidate (min 2)"
  - name: abstract
    type: string
    required: true
    description: "Brief summary of the consolidated insight (≤200 chars)"
  - name: content
    type: string
    required: true
    description: "Full consolidated content synthesizing all sources"
  - name: consolidation_type
    type: string
    required: true
    description: "One of: 'merge', 'summarize', 'generalize'"
  - name: confidence
    type: number
    required: true
    description: "Confidence in the consolidation (0.0-1.0)"
prompt_guidance: |
  ## process_consolidation usage

  Call this tool when you find memories that should be merged:
  
  **When to consolidate**:
  - 2+ memories about the same topic but with different details
  - Scattered observations that form a coherent pattern
  - Redundant information across multiple memories
  
  **Consolidation types**:
  - 'merge': Combine all details into one comprehensive memory
  - 'summarize': Extract the key points, discard specifics
  - 'generalize': Lift to a higher-level pattern or preference
  
  **Important**:
  - `source_uris` must have at least 2 URIs
  - The consolidated content should be more useful than individual memories
  - Use confidence 0.7+ for clear pattern, 0.5 for tentative grouping
  
post_call_validator: check_dream_duplicate
```

---

### 3.2 新增 Operational Tools

#### 3.2.1 acquire_pattern_candidates

**用途**：自动发现相似/矛盾的候选记忆组

```python
@operational_tool
def acquire_pattern_candidates(
    min_similarity: float = 0.75,
    category: str | None = None,
    limit: int = 10,
    ctx: RequestContext = None,
    context_fs: ContextFS = None,
    vector_index: VectorIndex = None,
    embedder: Embedder = None,
) -> dict:
    """Find clusters of similar memories that may need consolidation.
    
    Uses vector similarity to find memory pairs/groups that are semantically
    similar but have different URIs — candidates for consolidation or
    contradiction detection.
    
    Returns grouped candidates with similarity scores.
    """
    # 1. Get recent memories by recall stats (high signal first)
    # 2. For each, search for similar memories (excluding self)
    # 3. Group pairs with similarity >= min_similarity
    # 4. Return clusters with summaries
    ...
```

#### 3.2.2 acquire_conflict_candidates

**用途**：发现可能矛盾的记忆对

```python
@operational_tool
def acquire_conflict_candidates(
    days: int = 14,
    category: str | None = None,
    limit: int = 10,
    ctx: RequestContext = None,
    read_api: ReadAPI = None,
) -> dict:
    """Find memories that may contradict each other.
    
    Strategy:
    1. Find memories with similar topics but different content
    2. Find memories updated recently that conflict with older ones
    3. Return pairs with conflict indicators
    
    Returns candidate pairs with conflict explanation hints.
    """
    ...
```

---

### 3.3 Dream Prompt 重设计

**目标**：参考 Letta 的 5 Phase，增加优先级排序和过滤逻辑

```yaml
system_prompt: |
  You are a Deep Dream analyst. Your job is to consolidate, correct, and 
  organize memories — NOT the primary agent. You are reviewing memories 
  that already exist.
  
  **YOU ARE NOT THE PRIMARY AGENT.**
  - "system" context describes the user's preferences — use it to understand relevance
  - You do not respond to users; you only analyze and update memories
  
  **WORKFLOW — follow this order strictly:**

  1. **Signal Discovery** (start here)
     - Call acquire_search_recall_stats first → see which memories are frequently recalled
     - High recall_count + unique_queries indicates strong signal
     - Call acquire_pattern_candidates → find similar memories for consolidation
     - Call acquire_conflict_candidates → find potential contradictions
  
  2. **Deep Read** (only when needed)
     - Call read on memories identified as candidates
     - Focus on content that may conflict or contain errors
  
  3. **Extract** (prioritize in this order)
     
     **Priority 1 — MISTAKES AND CORRECTIONS (highest)**
     - Errors in existing memories (wrong facts, incorrect conclusions)
     - Contradictions between memories (which one is correct?)
     - Stale information (versions changed, paths moved, projects deleted)
     
     **Priority 2 — CONSOLIDATION**
     - Similar memories that should be merged
     - Scattered details forming a coherent pattern
     
     **Priority 3 — NEW PATTERNS (promotion)**
     - Recurring themes across multiple memories
     - Preferences and behavioral patterns
     
  4. **Filter** (before calling process tools)
     
     **Four questions for each candidate**:
     - **Durable?** Is this useful across sessions, or just ephemeral details?
     - **Already captured?** Does a similar dream memory already exist?
     - **Generalizable?** Can this be distilled to a reusable insight?
     - **Actionable?** Will this actually improve future agent behavior?
     
     **If nothing survives filtering → make no changes.**
     Empty dream is normal. Not every batch needs consolidation.
  
  5. **Process** (call appropriate tool for each candidate)
     - process_reflection for errors/contradictions
     - process_consolidation for similar memories
     - process_promotion for new patterns
  
  6. **Finish** 
     When no more significant candidates, state "Analysis complete."

identity_block: |
  Analyzing memories for user {{ user_id }}.
  
  **Focus on QUALITY over quantity**:
  - Few meaningful corrections > many trivial ones
  - One real contradiction fix > ten vague promotions
  - Empty output is acceptable if nothing needs consolidation

output_instruction: |
  Output all insights via process_* tools.
  
  **Quality rules**:
  - Do NOT call process tools until you have reviewed enough memories
  - Each output should explain WHY it matters, not just WHAT
  - Confidence: 
    - 0.8+ for clear contradiction/error
    - 0.7+ for pattern in 3+ sources
    - 0.5 for tentative observation
  
  If validator reports duplicate, refine with different angle or skip.
  
  When done, state "Analysis complete."
```

---

### 3.4 矛盾解决机制设计

**参考 Letta 的"在源头修正"原则**：

当前 oG-Memory 的 dream category 使用 `operation_mode: add_only`，不直接修改现有记忆。

**两种策略**：

#### 策略 A：标记 + 新增正确版本（推荐）

```
发现矛盾：memory_A 说 "Python 3.9"，memory_B 说 "Python 3.11"

操作：
1. 写入 dream reflection: "Python版本矛盾：A说3.9，B说3.11，正确是3.11"
2. Reflection 中包含 affected_uris = [memory_A, memory_B]
3. Future context engine 读取时：
   - 检查记忆是否有关联的 reflection
   - Reflection 标记为"可信度更高"
   - 用户查询时优先返回 reflection
```

**优点**：
- 不破坏原始记忆（保留历史）
- Reflection 作为"纠错标记"被检索
- 用户可以看到"纠错历史"

#### 策略 B：直接修改（Letta 模式）

```
发现矛盾：memory_A 说 "Python 3.9"

操作：
1. 直接调用 WriteAPI 修改 memory_A 的内容
2. 添加 provenance_ids 指向发现矛盾的来源
```

**优点**：
- 记忆库保持正确
- 不增加冗余 dream 条目

**缺点**：
- 丢失历史版本（无法回溯）
- 需要 WriteAPI 支持 update 模式（当前是 add_only）

---

### 3.5 信息获取策略优化

**当前**：`acquire_recent` 列出近期记忆 → LLM 自己判断哪些值得关注

**优化**：

```
启动顺序：

1. acquire_search_recall_stats(min_recall_count=3, days=14)
   → 获得"高信号候选"：recall_count 高、unique_queries 多
   
2. acquire_pattern_candidates(min_similarity=0.75)
   → 获得"相似候选"：可能需要 consolidation
   
3. acquire_conflict_candidates(days=14)
   → 获得"矛盾候选"：可能需要 reflection
   
4. acquire_recent(limit=20)
   → 补充"近期概览"：获得更广泛的上下文
   
5. read(uri=X) 
   → 只在确定候选后深入阅读
```

**设计意图**：
- 先让 SQL 做粗筛（高效）
- LLM 只做精细决策（高价值）
- 减少 LLM 需要阅读的条目数

---

## 四、实现路径

### Phase 1：短期实现（1-2 周）

| 任务 | 文件 | 说明 |
|------|------|------|
| 新增 process_reflection.yaml | `dream/tool_specs/reflection.yaml` | 错误/矛盾检测 |
| 新增 process_consolidation.yaml | `dream/tool_specs/consolidation.yaml` | 记忆合并 |
| 新增 acquire_pattern_candidates | `dream/operational_tools.py` | 相似候选发现 |
| 新增 acquire_conflict_candidates | `dream/operational_tools.py` | 矛盾候选发现 |
| 重写 dream.yaml prompt | `dream/prompts/dream.yaml` | 优先级排序 + 四重过滤 |
| 更新 DEEP_DREAM_MVP_OPERATIONAL | `dream/operational_tools.py` | 添加新工具 |

### Phase 2：长期优化（后续）

| 任务 | 说明 |
|------|------|
| Light Dream + Deep Dream 两阶段 | 参考 OpenClaw，先 Light 筛选候选，再 Deep 处理 |
| 自动触发机制 | compaction-event 或 step-count 触发 |
| 记忆老化清理 | TTL 或 usage-based eviction |
| Reflection 到 WriteAPI update | 支持直接修改现有记忆 |

---

## 五、对比总结

| 维度 | OpenClaw | Letta | oG-Memory (当前) | oG-Memory (设计后) |
|------|----------|-------|------------------|-------------------|
| 决策方式 | 纯规则评分 | LLM 5 Phase 流水线 | LLM ReAct Loop | LLM ReAct + 多工具 |
| 信号来源 | `.dreams/` JSON | transcript + parent_memory | `search_recall` SQL | SQL + pattern/conflict 工具 |
| dream_type | 无（只晋升） | 无（只有 reflection） | 只有 promotion | promotion + reflection + consolidation |
| 优先级 | 频率+相关性 | 纠错>偏好>事实>矛盾 | 无 | 纠错>整合>新模式 |
| 矛盾处理 | 无 | 在源头修正 | 无 | 标记 + Reflection（策略A） |
| Prompt 风格 | 公式晦涩 | 自然语言清晰 | 较简单 | 参考 Letta，明确优先级 |

---

## 六、参考文档

- [[OpenClaw Dreaming Mechanism]] — OpenClaw 三阶段数据流
- [[OpenClaw Dreaming Prompts and LLM Usage]] — OpenClaw 只有 Diary 用 LLM
- [[Letta Code Reflection Dream Prompt Deep Dive]] — Letta 5 Phase + 优先级排序
- [[ogmemory-deep-dream-framework]] — oG-Memory 当前架构