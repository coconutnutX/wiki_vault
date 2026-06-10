---
type: design
title: "oG-Memory Dreaming Enhancement Design"
status: draft
created: 2026-06-10
updated: 2026-06-10
tags: [ogmemory, dreaming, consolidation, correction, letta, openclaw]
related:
  - "[[OpenClaw Dreaming Mechanism]]"
  - "[[OpenClaw Dreaming Prompts and LLM Usage]]"
  - "[[Letta Code Reflection Dream Prompt Deep Dive]]"
  - "[[ogmemory-deep-dream-framework]]"
---

# oG-Memory Dreaming Enhancement Design

> 设计目标：将 `process_promotion` 替换为 `process_consolidation` + `process_correction` 两个工具，完善信息获取策略（借鉴 OpenClaw 信号筛选逻辑），优化 Prompt（借鉴 Letta 优先级排序）。

---

## 一、背景：三个系统的 Dreaming/Reflection 机制对比

### 1.1 OpenClaw：纯规则信号筛选器

**核心理念**：Dreaming 是"信号筛选器"，不产生新记忆，只从已有片段中筛选高信号条目晋升。

| Phase | 用 LLM? | 实际逻辑 |
|-------|---------|----------|
| Light | ❌ | 从 `memory/*.md` + `session-corpus/` 收集片段，`dailyCount++` |
| REM | ❌ | 分析 `conceptTags` 分布 → 计算 `patternStrength` |
| Deep | ❌ | 六信号评分公式 + 阈值门控 |
| Diary | ✅ | 生成诗意叙述（不参与决策） |

**六信号评分公式**：

```
score = frequency×0.24 + relevance×0.30 + diversity×0.15 
      + recency×0.15 + consolidation×0.10 + conceptual×0.06
      + lightBoost + remBoost

阈值门控：score >= 0.75 AND signalCount >= 3 AND uniqueQueries >= 2
```

**信号来源**：
- `memory_search` 每次被调用时异步写入 `short-term-recall.json`
- 记录：`recallCount`、`totalScore`、`queryHashes`、`recallDays`、`conceptTags`

**可借鉴点**：
- 评分公式中的 `frequency`（召回次数）、`relevance`（平均分数）、`diversity`（查询多样性）、`recency`（时效衰减）
- 阈值门控逻辑：不仅要高频，还要跨时间、跨查询
- oG-Memory 的 `search_recall` SQL 表天然记录这些信号，无需额外文件

---

### 1.2 Letta：LLM 驱动的精准外科手术

**核心理念**：Reflection 是"精准外科手术"，用 LLM 审视对话历史，精准修改记忆文件。

**Phase 2 — Extract 的优先级排序**（直接借鉴）：

```
优先级 1 — Mistakes and corrections（错误纠正）
  - Errors the system made (wrong facts, incorrect inferences)
  - User feedback pointing out mistakes
  - Failed retries that reveal misunderstandings

优先级 2 — Preferences and patterns（偏好模式）
  - User preferences about how they work, style choices
  - Recurring behavioral patterns
  
优先级 3 — New durable facts（持久事实）
  - Project details, team info, environment details
  - Facts that will remain useful
  
优先级 4 — Contradictions（矛盾）
  - Information that conflicts with what's already stored
```

**四重过滤**（直接借鉴）：

```
Durable? — Is this useful across sessions, or ephemeral details?
Already captured? — Does similar memory already exist?
Generalizable? — Can this be distilled to reusable insight?
Temporal references? — Convert "yesterday" to absolute dates
```

**关键原则**：
- "If nothing survives filtering, make no changes" —— 空 dream 是正常的
- 矛盾必须在源头修正，不允许新旧版本并存

---

### 1.3 oG-Memory 当前实现

**已有能力**：
- `search_recall` 表记录每次搜索召回信号（相当于 OpenClaw 的 `short-term-recall.json`）
- SQL 直接查询统计：`recall_count`、`unique_queries`、`recall_days`
- `acquire_search_recall_stats` 工具已提供候选排序

**当前局限**：
- 只有 `process_promotion` 一个工具，命名与 Letta 语义不符
- 没有 correction 类工具处理错误/矛盾
- Prompt 缺少优先级排序和四重过滤

---

## 二、设计目标：两个 Prompt-driven Tools

**核心改动**：
- 删除 `process_promotion`，替换为 `process_consolidation`
- 新增 `process_correction`
- 最终只有两个工具，语义清晰

| 工具 | 对应 Letta 优先级 | 用途 |
|------|------------------|------|
| `process_correction` | Mistakes and corrections / Contradictions | 发现并纠正错误、矛盾 |
| `process_consolidation` | Preferences and patterns / New durable facts | 合并相似记忆、提炼模式 |

---

## 三、具体设计

### 3.1 process_correction.yaml

**用途**：处理 Mistakes and corrections / Contradictions（Letta 优先级 1 和 4）

```yaml
tool_name: process_correction
category: dream
dream_type: correction
description: |
  Identify and correct errors, contradictions, or stale information in existing memories.
  Each call captures ONE correction — call multiple times for different issues.
  
  Use after reading memories that may conflict or contain outdated facts.
  
  **Priority order** (call corrections in this order):
  1. Mistakes and corrections — errors the system made, wrong facts, incorrect inferences
  2. Contradictions — two memories stating conflicting facts
  3. Stale information — facts no longer accurate (versions changed, paths moved)

fields:
  - name: topic
    type: string
    required: true
    description: "The correction topic (e.g. 'python_version_error', 'project_path_stale')"
  - name: correction_type
    type: string
    required: true
    description: "One of: 'mistake', 'contradiction', 'stale'"
  - name: affected_uris
    type: array
    required: true
    description: "URIs of memories containing the error/contradiction (min 1)"
  - name: correct_fact
    type: string
    required: true
    description: "The correct information or explanation"
  - name: abstract
    type: string
    required: true
    description: "Brief summary (≤200 chars)"
  - name: content
    type: string
    required: true
    description: "Detailed explanation: what was wrong, why, and the correction"
  - name: confidence
    type: number
    required: true
    description: "Confidence in the correction (0.0-1.0)"

prompt_guidance: |
  ## process_correction usage
  
  Call this tool when you identify errors or conflicts in existing memories.
  
  **Priority 1 — Mistakes and corrections (call first)**
  
  Look for:
  - Wrong facts in existing memories (incorrect versions, wrong paths)
  - Incorrect conclusions or inferences the system made
  - User feedback pointing out mistakes in previous sessions
  - Failed retries that reveal misunderstandings
  
  Example: Memory says "Python 3.9 installed" but user just mentioned using 3.11
  
  **Priority 2 — Contradictions**
  
  Look for:
  - Two memories stating conflicting facts about the same topic
  - Newer observations contradicting older stored facts
  
  When found:
  - Explain WHICH memory is correct and WHY
  - If uncertain, explain the conflict and recommend verification
  
  Example: Memory A says "API endpoint at /v1" but Memory B says "/v2"
  
  **Priority 3 — Stale information**
  
  Look for:
  - Facts that are no longer accurate
  - References to deleted files or moved directories
  - Outdated configuration values
  
  Example: Memory references ".env file in project root" but project moved to subdirectory
  
  **Before calling, verify**:
  - `affected_uris` must list at least 1 memory URI
  - `correct_fact` should be unambiguous and verifiable
  - Confidence: 0.8+ for clear mistake, 0.6+ for contradiction, 0.5+ for stale
  
  **Important**:
  - Each call corrects ONE issue. If you find multiple unrelated errors, call multiple times.
  - The correction dream will be retrieved alongside original memories in future searches.
  - If validator reports duplicate, refine with different angle or skip.

post_call_validator: check_dream_duplicate
```

---

### 3.2 process_consolidation.yaml

**用途**：处理 Preferences and patterns / New durable facts（Letta 优先级 2 和 3）

```yaml
tool_name: process_consolidation
category: dream
dream_type: consolidation
description: |
  Merge similar memories into consolidated higher-level insights.
  Extract recurring patterns and preferences from scattered observations.
  Each call captures ONE consolidated topic — call multiple times for different themes.
  
  **Use for**:
  - Preferences and patterns: recurring behavioral patterns, style choices
  - New durable facts: project details, team info, environment configuration
  - Similar memories: multiple memories about same topic with scattered details

fields:
  - name: topic
    type: string
    required: true
    description: "The consolidated topic (e.g. 'coding_style_preference', 'project_structure')"
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
    description: "Full consolidated content: WHY this pattern matters, not just WHAT"
  - name: consolidation_type
    type: string
    required: true
    description: "One of: 'pattern', 'preference', 'fact', 'merge'"
  - name: confidence
    type: number
    required: true
    description: "Confidence in the consolidation (0.0-1.0)"

prompt_guidance: |
  ## process_consolidation usage
  
  Call this tool when you find memories that should be consolidated into higher-level insights.
  
  **Priority 2 — Preferences and patterns**
  
  Look for:
  - Recurring behavioral patterns across multiple sessions
  - User preferences about coding style, tools, workflow
  - Style choices mentioned repeatedly
  
  Example: Multiple memories mentioning "prefer explicit error handling" → consolidate into coding_style_preference
  
  **Priority 3 — New durable facts**
  
  Look for:
  - Project details scattered across conversations
  - Team info, environment configuration mentioned multiple times
  - Facts that will remain useful across sessions
  
  Example: Three memories each mentioning different parts of project structure → consolidate into project_structure fact
  
  **Consolidation types**:
  - 'pattern': Recurring behavioral pattern or tendency
  - 'preference': Explicit user preference about how they work
  - 'fact': Durable factual knowledge (project/team/config details)
  - 'merge': Direct merge of similar memories into comprehensive single memory
  
  **Before calling, verify**:
  - `source_uris` must have at least 2 URIs (consolidation requires multiple sources)
  - The consolidated content should explain WHY this matters, not just list facts
  - Confidence: 0.7+ for pattern in 3+ sources, 0.5+ for pattern in 2 sources
  
  **Quality rules**:
  - Do NOT consolidate ephemeral details (temporary debugging states, one-time queries)
  - Consolidated content should be more useful than individual memories
  - Focus on WHAT the user does repeatedly and WHY it matters
  
  **Important**:
  - Each call consolidates ONE topic. Different themes need separate calls.
  - If validator reports duplicate, refine with deeper angle or skip.

post_call_validator: check_dream_duplicate
```

---

### 3.3 Dream Prompt 重设计

**目标**：
- 借鉴 Letta 的优先级排序和四重过滤
- 在 `acquire_recent` 和 `acquire_search_recall_stats` 的 guidance 中借鉴 OpenClaw 的评分逻辑

```yaml
system_prompt: |
  You are a Deep Dream analyst. Your job is to consolidate and correct existing memories.
  
  **YOU ARE NOT THE PRIMARY AGENT.**
  - "system" context describes the user's preferences — use it to understand relevance
  - You do not respond to users; you only analyze and update memories
  - You are reviewing conversations that already happened
  
  **WORKFLOW — follow this order strictly:**

  1. **Signal Discovery** (borrowed from OpenClaw scoring logic)
  
     Call these tools to find high-signal candidates:
     
     a) **acquire_search_recall_stats(min_recall_count=3, days=14)**
        - Returns memories ranked by: recall_count, unique_queries, recall_days
        - High recall_count = frequently retrieved = strong signal
        - High unique_queries = retrieved by diverse queries = broad relevance
        - High recall_days = retrieved across multiple days = durable interest
        
        **OpenClaw-inspired scoring intuition**:
        - Frequency: log1p(recall_count) / log1p(10) → more recalls = higher score
        - Diversity: unique_queries / 5 → diverse query contexts = valuable
        - Recency: exponential decay (half-life ~14 days) → fresh signals preferred
        - Threshold: recall_count >= 3 AND unique_queries >= 2 AND recall_days >= 2
        
        Start with memories that pass these thresholds.
     
     b) **acquire_recent(limit=20)**
        - Broad overview of recent memory landscape
        - Use summaries to identify themes without deep reading
        - Focus on memories with high recall_count from step (a)
     
     c) **acquire_search(query="...")** (optional)
        - Targeted search for specific themes spotted in summaries
        - Use to find more memories related to potential patterns
     
     d) **read(uri=X)** (only when needed)
        - Deep read only for candidates identified in steps (a)-(c)
        - Read to confirm pattern or verify contradiction

  2. **Extract** (borrowed from Letta Phase 2)
  
     Review candidates and identify issues in **priority order**:
     
     **Priority 1 — Mistakes and corrections (call process_correction first)**
     - Errors in existing memories (wrong facts, incorrect inferences)
     - User feedback pointing out mistakes
     - Failed retries revealing misunderstandings
     
     **Priority 2 — Contradictions (call process_correction)**
     - Two memories stating conflicting facts
     - Newer information contradicting older memory
     
     **Priority 3 — Preferences and patterns (call process_consolidation)**
     - Recurring behavioral patterns across sessions
     - User style choices mentioned repeatedly
     
     **Priority 4 — New durable facts (call process_consolidation)**
     - Project details, team info, configuration facts
     - Information that will remain useful

  3. **Filter** (borrowed from Letta four-question test)
  
     Before calling process tools, ask for each candidate:
     
     - **Durable?** Is this useful across sessions, or ephemeral details?
       → Skip debugging logs, one-time queries, temporary states
     
     - **Already captured?** Does similar dream memory already exist?
       → Check validator feedback for duplicates
     
     - **Generalizable?** Can this be distilled to reusable insight?
       → Skip session-specific details that won't help future turns
     
     - **Actionable?** Will this actually improve future agent behavior?
       → Skip trivia, focus on patterns that affect decisions
     
     **If nothing survives filtering → make no changes.**
     Empty dream output is normal. Not every batch needs consolidation.

  4. **Process** (call appropriate tool)
     - process_correction for mistakes/contradictions (Priority 1-2)
     - process_consolidation for patterns/facts (Priority 3-4)
     
     Each call covers ONE topic. Multiple unrelated issues need multiple calls.

  5. **Finish**
     When no more significant candidates, state "Analysis complete."

identity_block: |
  Analyzing memories for user {{ user_id }}.
  
  **Focus on QUALITY over quantity**:
  - One real correction > ten vague consolidations
  - Pattern backed by 3+ memories > speculative guess
  - Empty output is acceptable if nothing survives filtering

output_instruction: |
  Output all insights via process_correction or process_consolidation tools.
  
  **For process_correction**:
  - correction_type: mistake > contradiction > stale
  - confidence: 0.8+ for clear mistake, 0.6+ for conflict, 0.5+ for stale
  - correct_fact: must be unambiguous
  
  **For process_consolidation**:
  - consolidation_type: pattern > preference > fact > merge
  - source_uris: minimum 2 URIs required
  - confidence: 0.7+ for pattern in 3+ sources, 0.5+ for 2 sources
  
  If validator reports duplicate, refine with different angle or skip.
  
  When done, state "Analysis complete."
```

---

### 3.4 Operational Tools Prompt Guidance

**在工具 docstring 中增加 OpenClaw-inspired 评分逻辑说明**：

#### acquire_search_recall_stats 增强

```python
@operational_tool
def acquire_search_recall_stats(
    min_recall_count: int = 3,
    days: int = 7,
    category: str | None = None,
    ctx: RequestContext = None,
    read_api: ReadAPI = None,
) -> dict:
    """Find memories that have been frequently recalled via search.
    
    Returns ranked candidates with recall statistics for dream consolidation.
    Use this FIRST to identify high-signal memories worth deeper analysis.
    
    **OpenClaw-inspired scoring intuition** (for interpreting results):
    
    The returned candidates are ranked by:
    - recall_count: Number of times retrieved → higher = more valuable
      → OpenClaw uses log1p(count) / log1p(10) for frequency signal
    
    - unique_queries: Number of different queries that retrieved this memory
      → higher = broader relevance (retrieved in diverse contexts)
      → OpenClaw uses this for "diversity" signal
    
    - recall_days: Number of distinct days this memory was retrieved
      → higher = durable interest across time
      → OpenClaw uses this for "consolidation" signal
    
    **Recommended thresholds** (borrowed from OpenClaw):
    - recall_count >= 3: minimum frequency to be considered signal
    - unique_queries >= 2: must be retrieved by at least 2 different queries
    - recall_days >= 2: must span multiple days (not single-session ephemeral)
    
    Memories passing all thresholds are strong candidates for consolidation.
    
    :param min_recall_count: Minimum recall count threshold (default 3, OpenClaw-inspired).
    :param days: Look-back window in days (default 7, captures recent signal).
    :param category: Optional category filter (e.g. 'events', 'preferences').
    """
    ...
```

#### acquire_recent 增强

```python
@operational_tool
def acquire_recent(
    limit: int = 20,
    category_filter: list[str] | None = None,
    ctx: RequestContext = None,
    context_fs: ContextFS = None,
) -> dict:
    """List recent memory URIs with brief summaries for deep dream analysis.
    
    Call this AFTER acquire_search_recall_stats to get broad context.
    Use summaries to identify themes without needing to read every memory.
    
    **OpenClaw-inspired signal interpretation**:
    
    The summaries include abstract snippets. Look for:
    - Repeated keywords across multiple memories → pattern candidate
    - Similar topics mentioned in different categories → consolidation candidate
    - Contradictory information between memories → correction candidate
    
    **Recency signal** (OpenClaw uses exponential decay):
    - Memories from last 7 days: strong signal (recency score ~0.5-1.0)
    - Memories from 7-14 days: moderate signal (recency score ~0.25-0.5)
    - Memories older than 14 days: weak signal (recency score <0.25)
    
    Focus analysis on recent, high-recall-count memories first.
    
    :param limit: Max number of memories to return. Minimum enforced: 10.
    :param category_filter: Optional category names to restrict results.
    """
    ...
```

---

### 3.5 矛盾解决机制

**采用策略 A：标记 + 新增正确版本**

```
发现矛盾：memory_A 说 "Python 3.9"，memory_B 说 "Python 3.11"

操作：
1. 写入 correction dream: "Python版本矛盾：A说3.9，B说3.11，正确是3.11"
2. Correction 中包含 affected_uris = [memory_A, memory_B]
3. Future context engine 读取时：
   - Correction dream 标记为"可信度更高"（confidence 0.6+）
   - 用户查询时优先返回 correction
   - Original memories 保留（可追溯历史）
```

**优点**：
- 不破坏原始记忆（保留历史）
- Correction 作为"纠错标记"被检索
- 用户可以看到"纠错历史"

---

## 四、实现路径

### Phase 1：短期实现

| 任务 | 文件 | 说明 |
|------|------|------|
| 删除 process_promotion.yaml | `dream/tool_specs/promotion.yaml` | 替换为 consolidation |
| 新增 process_correction.yaml | `dream/tool_specs/correction.yaml` | Mistakes/Contradictions |
| 新增 process_consolidation.yaml | `dream/tool_specs/consolidation.yaml` | Patterns/Facts |
| 重写 dream.yaml prompt | `dream/prompts/dream.yaml` | Letta 优先级 + OpenClaw 评分 |
| 增强 acquire_search_recall_stats docstring | `dream/recall_stats_tool.py` | OpenClaw scoring guidance |
| 增强 acquire_recent docstring | `dream/operational_tools.py` | OpenClaw recency guidance |
| 更新 DEEP_DREAM_MVP_OPERATIONAL | `dream/operational_tools.py` | 移除 promotion，添加新工具 |

### Phase 2：长期优化

| 任务 | 说明 |
|------|------|
| Light Dream + Deep Dream 两阶段 | 参考 OpenClaw，先 Light 筛选候选，再 Deep 处理 |
| 自动触发机制 | compaction-event 或 step-count 触发 |
| Correction 到 WriteAPI update | 支持直接修改现有记忆（策略 B） |

---

## 五、对比总结

| 维度 | OpenClaw | Letta | oG-Memory (当前) | oG-Memory (设计后) |
|------|----------|-------|------------------|-------------------|
| 决策方式 | 纯规则评分 | LLM 5 Phase 流水线 | LLM ReAct Loop | LLM ReAct + 优先级排序 |
| 信号来源 | `.dreams/` JSON | transcript + parent_memory | `search_recall` SQL | SQL（OpenClaw scoring guidance） |
| dream_type | 无（只晋升） | 无（只有 reflection） | promotion | consolidation + correction |
| 工具数量 | 0（纯规则） | 0（Bash + 修改文件） | 1 | 2 |
| 优先级 | 公式权重 | 纠错>偏好>事实>矛盾 | 无 | correction > consolidation |
| Prompt 借鉴 | 公式晦涩 | 自然语言优先级 | 简单 workflow | Letta 优先级 + OpenClaw scoring intuition |

---

## 六、参考文档

- [[OpenClaw Dreaming Mechanism]] — OpenClaw 三阶段数据流、六信号评分公式
- [[OpenClaw Dreaming Prompts and LLM Usage]] — OpenClaw 只有 Diary 用 LLM，Deep 是纯规则
- [[Letta Code Reflection Dream Prompt Deep Dive]] — Letta 5 Phase + 优先级排序 + 四重过滤
- [[ogmemory-deep-dream-framework]] — oG-Memory 当前架构