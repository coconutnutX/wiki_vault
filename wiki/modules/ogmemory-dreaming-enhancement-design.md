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

> 设计目标：将 `process_promotion` 替换为 `process_consolidation` + `process_correction` 两个工具，完善信息获取策略（借鉴 OpenClaw 信号筛选逻辑），优化 Prompt（借鉴 Letta 优先级排序 + 四重过滤，但作为 ReAct agent 的策略指引而非线性流水线）。写入发生在 ReAct loop 外面（loop 内只收集 DreamOutput，loop 外统一批量写入，现阶段 add_only）。新增字段（sub_type、correct_fact、confidence）通过 ContextNode.metadata 传递。provenance_ids 语义修正为 LLM 明确指定的来源记忆（affected_uris / source_uris），不再是访问过的所有 URI。

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
  
  **Priority 1 — Preferences and patterns**
  
  Look for:
  - Recurring behavioral patterns across multiple sessions
  - User preferences about coding style, tools, workflow
  - Style choices mentioned repeatedly
  
  Example: Multiple memories mentioning "prefer explicit error handling" → consolidate into coding_style_preference
  
  **Priority 2 — New durable facts**
  
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

### 3.3 字段映射链路：LLM 输出 → ContextNode

新工具引入了 process_promotion 没有的字段（correction_type、affected_uris、correct_fact、consolidation_type、source_uris），**必须确保这些字段能沿整条链路写入最终的 ContextNode**。

现有链路：

```
LLM tool_call input
  → _tool_input_to_dream_output()     (deep_dream_react_loop.py:344-373)
  → DreamOutput                        (dream/models.py:14-33)
  → .to_candidate_memory()             (dream/models.py:35-51)
  → CandidateMemory                    (core/models.py:145-166)
  → ArchiveBuilder.build()             (commit/archive_builder.py:40-145)
  → ContextNode                        (core/models.py:86-103)
    → ContextNode.metadata: dict[str, Any]  ← 所有额外字段写到这里
  → SQLContextFS.write_node_with_outbox()
```

ContextNode 已经有 `metadata: dict[str, Any]`，ArchiveBuilder 也是直接往 metadata 里塞 key（routing_key、provenance_ids、tool_stats 等），SQL 端 metadata 是 JSONB 列。**新字段直接写 metadata 即可，不需要改 CandidateMemory 或 ContextNode 的结构。**

#### 现有映射（process_promotion）

| LLM 填的字段 | `_tool_input_to_dream_output()` | DreamOutput | `to_candidate_memory()` | CandidateMemory | ArchiveBuilder → ContextNode |
|---|---|---|---|---|---|
| topic | ✅ `tool_input.get("topic")` | `topic` | ✅ routing_key = `f"{dream_type}_{sanitized_topic}"` | `routing_key` | ✅ metadata.routing_key + URI |
| abstract | ✅ `tool_input.get("abstract")` | `abstract` | ✅ 直接赋值 | `abstract` | ✅ node.abstract |
| content | ✅ `tool_input.get("content")` | `content` | ✅ 直接赋值 | `content` | ✅ node.content |
| confidence | ✅ `tool_input.get("confidence", 0.5)` | `confidence` | ✅ 直接赋值 | `confidence` | ✅ metadata.confidence（需补充） |

> **注**：现有 ArchiveBuilder.build() 中 confidence 没有被写入 metadata，需要补充。
> confidence 对未来检索排序有价值（"可信度更高的 correction 优先返回"）。

#### provenance_ids 语义修正

现有实现中，DreamOutput.provenance_ids 来自 `_read_uris`（LLM 在 loop 中访问过的所有 URI）。**这个语义不对**——在 dream 语境下，provenance_ids 应该是 LLM **明确指定的来源记忆**（即 affected_uris / source_uris），不是"碰巧读过的所有 URI"。

修正后：
- **correction**：provenance_ids = `affected_uris`（LLM 明确指出被纠正的记忆）
- **consolidation**：provenance_ids = `source_uris`（LLM 明确指出被合并的记忆）

不再自动从 `_read_uris` 赋值 provenance_ids。`_read_uris` 可以保留为 loop 内部调试日志，但不写入 ContextNode。

#### 新字段映射方案：DreamOutput → metadata

不需要给 CandidateMemory 加 extra_metadata 字段。新字段在 DreamOutput 层收集，`to_candidate_memory()` 把它们放进 CandidateMemory 的 `provenance_ids`（语义已修正）和现有 metadata 传递路径。具体做法：

**1. DreamOutput 新增字段**（dream/models.py）：

```python
@dataclass
class DreamOutput:
    content: str
    abstract: str
    dream_type: str                     # "correction" | "consolidation"
    confidence: float
    provenance_ids: list[str] = field(default_factory=list)  # LLM 指定的来源 URI
    topic: str | None = None
    # New fields for correction/consolidation:
    sub_type: str | None = None         # "mistake"|"contradiction"|"stale" for correction;
                                        # "pattern"|"preference"|"fact"|"merge" for consolidation
    correct_fact: str | None = None     # only for correction
```

> `target_uris` 不需要单独字段——它就是 provenance_ids（affected_uris / source_uris），
> 统一为 provenance_ids，语义已修正为"LLM 明确指定的来源记忆"。

**2. `_tool_input_to_dream_output()` 提取新字段**（deep_dream_react_loop.py:344-373）：

```python
def _tool_input_to_dream_output(self, tool_input, spec) -> DreamOutput | None:
    ...
    sub_type = tool_input.get("correction_type") or tool_input.get("consolidation_type")
    source_uris = tool_input.get("affected_uris") or tool_input.get("source_uris") or []
    correct_fact = tool_input.get("correct_fact")

    # provenance_ids = LLM 明确指定的来源记忆（不是访问过的所有 URI）
    provenance_ids = [ProvenanceResolver.build_id("memory", uri) for uri in source_uris]

    return DreamOutput(
        content=content, abstract=abstract, confidence=float(confidence),
        dream_type=spec.dream_type, topic=topic,
        provenance_ids=provenance_ids,  # 直接从 LLM 指定的 affected_uris/source_uris
        sub_type=sub_type, correct_fact=correct_fact,
    )
```

> **关键变化**：不再从 `_read_uris` 自动赋值 provenance_ids。
> provenance_ids 直接从 LLM tool_call 的 affected_uris / source_uris 构建。

**3. `to_candidate_memory()` 映射**（dream/models.py:35-51）：

CandidateMemory 不需要加新字段。DreamOutput 的 sub_type 和 correct_fact 通过一个简单机制传递到 ContextNode.metadata：

方案是让 `to_candidate_memory()` 把这些额外信息放进 CandidateMemory 的 `tool_stats` 字段（现有字段，目前 dream category 不用）。ArchiveBuilder 已经会把 tool_stats 写入 metadata：

```python
def to_candidate_memory(self) -> CandidateMemory:
    routing_key = f"{self.dream_type}_{sanitized_topic}" if sanitized_topic else self.dream_type
    # Pack dream-specific metadata into tool_stats (ArchiveBuilder writes it to ContextNode.metadata)
    dream_meta = {}
    if self.sub_type:
        dream_meta["sub_type"] = self.sub_type
    if self.correct_fact:
        dream_meta["correct_fact"] = self.correct_fact
    dream_meta["confidence"] = self.confidence

    return CandidateMemory(
        category="dream", owner_scope="user",
        routing_key=routing_key,
        abstract=self.abstract, overview=self.abstract,
        content=self.content, confidence=self.confidence,
        provenance_ids=self.provenance_ids,
        tool_stats=dream_meta,  # dream metadata → ArchiveBuilder → ContextNode.metadata
    )
```

ArchiveBuilder.build() 中已有逻辑（archive_builder.py:120-122）：

```python
# Preserve tool usage stats for tool category
if candidate.tool_stats:
    metadata["tool_stats"] = candidate.tool_stats
```

这样最终 ContextNode.metadata 会包含：

```json
{
  "routing_key": "correction_python_version_error",
  "provenance_ids": ["memory:ctx://acme/users/alice/memories/facts/python_version"],
  "tool_stats": {
    "sub_type": "mistake",
    "correct_fact": "Python 3.11 is installed, not 3.9",
    "confidence": 0.85
  },
  "created_at": "2026-06-10T..."
}
```

> **tool_stats 的语义**：对 tool category 它是真正的工具使用统计，对 dream category
> 它是一个已存在的 metadata 传递通道。ArchiveBuilder 已有代码把 tool_stats 写入
> ContextNode.metadata，不需要改 ArchiveBuilder 或 ContextNode 的结构。
> 未来如果想更清晰，可以把 metadata key 从 `tool_stats` 改为 `dream_meta`，
> 但现阶段复用 tool_stats 通道是最小改动方案。

**4. 删除 `_read_uris` 自动赋值 provenance_ids 的逻辑**（deep_dream_react_loop.py:175-184）：

```python
# 现有代码（需删除）：
# dream_output.provenance_ids = self._build_provenance_ids(current_uris)

# 新代码：provenance_ids 在 _tool_input_to_dream_output() 中已从
# affected_uris/source_uris 构建，不再需要自动赋值。
```

#### confidence 写入 ContextNode

confidence 通过 tool_stats 传递到 ContextNode.metadata（见上面的 `dream_meta["confidence"] = self.confidence`）。现阶段不用于检索逻辑，仅存储，为未来检索排序预留。

### 3.4 Dream Prompt 重设计

**目标**：
- 给 ReAct agent 策略指引而非线性流水线（agent 按需自主决策，不机械执行步骤序列）
- 借鉴 Letta 的优先级排序和四重过滤（作为判断标准，不作为强制步骤）
- 借鉴 OpenClaw 的评分逻辑（作为数据解读指引，帮助 LLM 理解返回数据的含义）

**设计原则**：
- ReAct agent = 自主决策 + 策略指引，不是 pipeline
- 入手明确（两个信息获取工具），深入灵活（按需 read/search），输出随时（不必等"步骤 4"）
- 四重过滤是质量门槛，不是流程步骤
- 写入发生在 ReAct loop 外面（loop 内只收集 DreamOutput，loop 外统一批量写入）

```yaml
system_prompt: |
  You are a Deep Dream analyst. Your job is to review existing memories,
  identify patterns worth consolidating and errors worth correcting, then
  output structured insights via the two process tools.

  **YOU ARE NOT THE PRIMARY AGENT.**
  - You do not respond to users; you only analyze and update memories.
  - You are reviewing conversations that already happened.
  - Calling process_correction or process_consolidation records your insight
    as structured data. The actual database write happens after your analysis
    completes — your job is to produce quality insights, not to manage writes.

  **STRATEGY GUIDANCE — not a rigid sequence. Decide your next action
  based on what you learn from each tool call.**

  **Starting point — acquire signal overview:**

  Begin by calling one or both of these tools to get a picture of the
  memory landscape and identify high-signal candidates:

  - **acquire_search_recall_stats** — returns memories ranked by recall
    statistics. Use it to find memories that are frequently retrieved,
    retrieved by diverse queries, or retrieved across multiple days.
    These are strong signals for consolidation.

  - **acquire_recent** — returns recent memory URIs with brief summaries
    (category + abstract). Use it to get a broad overview and spot themes.

  Combine the two: recall_stats tells you *which* memories matter;
  acquire_recent tells you *what* they are about.

  **Dive deeper — follow signals on demand:**

  Once you spot candidates from the overview, use these tools flexibly:

  - **read(uri)** — read a memory's full content to confirm a pattern or
    verify a contradiction. Only read what you need; summaries from
    acquire_recent may be enough for many candidates.

  - **acquire_search(query)** — search for related memories when you find
    a potential pattern or contradiction and want to see if more evidence
    exists. For example, if you spot a preference theme, search for
    similar terms to find additional instances.

  **Output — call process tools as you confirm findings:**

  You may call process tools at any point during analysis — you do not
  need to wait until you have reviewed everything. Each call covers ONE
  topic; different themes need separate calls.

  - **process_correction** — for mistakes, contradictions, or stale info.
    Correction priority: mistake > contradiction > stale.

  - **process_consolidation** — for patterns, preferences, durable facts,
    or merging similar memories.
    Consolidation priority: pattern > preference > fact > merge.

  **Quality gate — four-question filter before each process call:**

  Before calling any process tool, check the candidate against these
  criteria. Skip anything that doesn't pass:

  - **Durable?** — Useful across sessions, not ephemeral (skip debugging
    logs, one-time queries, temporary states).
  - **Already captured?** — Not already covered by an existing dream.
    Validator feedback will flag duplicates.
  - **Generalizable?** — Can be distilled to a reusable insight (skip
    session-specific details).
  - **Actionable?** — Will actually improve future agent behavior (skip
    trivia).

  **If nothing survives filtering → make no changes.**
  Empty dream output is normal. Not every batch needs consolidation.

  **Finish:** When no more significant candidates remain, state
  "Analysis complete."

identity_block: |
  Analyzing memories for user {{ user_id }}.

  **Focus on QUALITY over quantity:**
  - One real correction > ten vague consolidations
  - Pattern backed by 3+ sources > speculative guess from 1 source
  - Empty output is acceptable if nothing survives filtering

output_instruction: |
  Output insights via process_correction or process_consolidation.

  **For process_correction**:
  - correction_type: mistake > contradiction > stale
  - affected_uris: list at least 1 URI containing the error
  - confidence: 0.8+ for clear mistake, 0.6+ for contradiction, 0.5+ for stale
  - correct_fact: must be unambiguous

  **For process_consolidation**:
  - consolidation_type: pattern > preference > fact > merge
  - source_uris: minimum 2 URIs required
  - confidence: 0.7+ for pattern in 3+ sources, 0.5+ for 2 sources

  If validator reports duplicate, refine with a different angle or skip.

  When done, state "Analysis complete."
```

---

### 3.5 Operational Tools Prompt Guidance

**在工具 docstring 中增加信号解读指引**（帮助 LLM 理解返回数据的含义，而非规定调用顺序）：

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

    Returns ranked candidates with recall statistics.
    Use this to identify high-signal memories worth deeper analysis.

    **How to interpret the results:**

    Each candidate has three statistics — here's what they mean:

    - recall_count: Number of times this memory was retrieved by search.
      Higher count = more frequently needed = stronger signal.
      OpenClaw equivalent: frequency signal (log1p(count) / log1p(10)).

    - unique_queries: Number of different search queries that retrieved
      this memory. Higher = broader relevance — it's useful in diverse
      contexts, not just one narrow query.
      OpenClaw equivalent: diversity signal.

    - recall_days: Number of distinct days this memory was retrieved.
      Higher = durable interest across time, not a single-session spike.
      OpenClaw equivalent: consolidation signal.

    **Signal thresholds** — candidates that pass all three are strong:
    - recall_count >= 3: enough frequency to be signal, not noise
    - unique_queries >= 2: retrieved in at least 2 different contexts
    - recall_days >= 2: interest spans multiple days, not ephemeral

    :param min_recall_count: Minimum recall count threshold (default 3).
    :param days: Look-back window in days (default 7).
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
    """List recent memory URIs with brief summaries for dream analysis.

    Returns URIs with category + abstract snippets — use these to spot
    themes without needing to read every memory in full.

    **How to interpret the summaries:**

    Look for these patterns across the returned summaries:
    - Repeated keywords across multiple memories → consolidation candidate
    - Similar topics in different categories → pattern spanning domains
    - Contradictory information between memories → correction candidate

    **Recency intuition:**
    - Last 7 days: strong signal (recently created/updated, likely relevant)
    - 7-14 days: moderate signal
    - Older than 14 days: weak signal unless high recall_count

    :param limit: Max number of memories to return. Minimum enforced: 10.
    :param category_filter: Optional category names to restrict results.
    """
    ...
```

---

### 3.6 矛盾解决机制

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
| 删除 process_promotion.yaml | `dream/tool_specs/promotion.yaml` | 替换为 consolidation + correction |
| 新增 process_correction.yaml | `dream/tool_specs/correction.yaml` | Mistakes/Contradictions/Stale |
| 新增 process_consolidation.yaml | `dream/tool_specs/consolidation.yaml` | Patterns/Preferences/Facts/Merge |
| DreamOutput 新增字段 | `dream/models.py` | sub_type, correct_fact（provenance_ids 语义修正为 LLM 指定的来源 URI） |
| `_tool_input_to_dream_output()` 提取新字段 | `dream/deep_dream_react_loop.py` | 从 tool_input 提取 correction_type/consolidation_type→sub_type, affected_uris/source_uris→provenance_ids, correct_fact |
| 删除 `_read_uris` 自动赋值 provenance_ids | `dream/deep_dream_react_loop.py` | provenance_ids 应来自 LLM 指定的来源，不是访问过的所有 URI |
| `to_candidate_memory()` 映射新字段 | `dream/models.py` | sub_type/correct_fact/confidence → CandidateMemory.tool_stats → ArchiveBuilder → ContextNode.metadata |
| 重写 dream.yaml prompt | `dream/prompts/dream.yaml` | ReAct 策略指引（不是线性流水线） |
| 增强 acquire_search_recall_stats docstring | `dream/recall_stats_tool.py` | 信号解读指引（帮助 LLM 理解数据含义） |
| 增强 acquire_recent docstring | `dream/operational_tools.py` | 信号解读指引（帮助 LLM 理解数据含义） |
| 更新 DEEP_DREAM_MVP_OPERATIONAL | `dream/operational_tools.py` | 移除 promotion，确认新工具列表 |

> **写入架构说明**：ReAct loop 内只收集 DreamOutput，写入发生在 loop 外
> （memory_service.py 在 `loop.run()` 返回后遍历 candidates 调用
> `write_api.write_memory()`）。现阶段 add_only 模式直接写新节点，
> 以后扩展支持 update/merge 等操作也只需在 commit pipeline 层修改，
> 不影响 ReAct loop 内的工具设计。

---

## 五、对比总结

| 维度 | OpenClaw | Letta | oG-Memory (当前) | oG-Memory (设计后) |
|------|----------|-------|------------------|-------------------|
| 决策方式 | 纯规则评分 | LLM 5 Phase 流水线 | LLM ReAct Loop | LLM ReAct + 策略指引（自主决策） |
| 信号来源 | `.dreams/` JSON | transcript + parent_memory | `search_recall` SQL | SQL（信号解读指引） |
| dream_type | 无（只晋升） | 无（只有 reflection） | promotion | consolidation + correction |
| 工具数量 | 0（纯规则） | 0（Bash + 修改文件） | 1 | 2 |
| 优先级 | 公式权重 | 纠错>偏好>事实>矛盾（流水线） | 无 | correction > consolidation（判断标准，非强制步骤） |
| Prompt 风格 | 公式晦涩 | 自然语言步骤序列 | 简单 linear workflow | 策略指引 + 信号解读（ReAct agent） |
| 写入时机 | 规则触发即写入 | Phase 内写入 | loop 外批量写入 | loop 外批量写入（add_only，后续扩展 update/merge） |

---

## 六、参考文档

- [[OpenClaw Dreaming Mechanism]] — OpenClaw 三阶段数据流、六信号评分公式
- [[OpenClaw Dreaming Prompts and LLM Usage]] — OpenClaw 只有 Diary 用 LLM，Deep 是纯规则
- [[Letta Code Reflection Dream Prompt Deep Dive]] — Letta 5 Phase + 优先级排序 + 四重过滤
- [[ogmemory-deep-dream-framework]] — oG-Memory 当前架构