---
type: module
status: draft
created: 2026-05-21
updated: 2026-05-21
tags: [ogmemory, lazy-mode, extraction, optimization, react-loop]
related:
  - "[[Lazy Mode Empty Database Bug Fix]]"
  - "[[oG-Memory ReactLoop Abstract Base Class]]"
  - "[[ogmemory-architecture]]"
---

# oG-Memory Lazy Mode Extraction Issues and Optimization Plan

Lazy 模式（ReAct loop）在提取记忆时面临多个问题，包括类型判断错误、重复提取、冲突处理不当等。本文档分析现有实现的问题并提出优化方案。

---

## 问题背景

Lazy 模式使用 ReAct loop（ExtractionReActLoop）进行记忆提取，流程如下：

```
Phase 1 (Span Identification)
  → 分析对话，预测可能相关的 categories
  → 只用于 prefetch（预读取哪些 category 的已有记忆）

Phase 2 (Span Structuring)
  → 所有 extract_* 工具定义都传给 LLM
  → LLM 自己决定调用哪个工具
  → LLM 生成 tool_input（包含 routing_key 和记忆内容）
  → parse_tool_call() 验证 schema 并生成 CandidateMemory
```

关键设计：**LLM 主导类型选择和 routing_key 生成**，系统只做 schema validation 和兜底生成。

---

## 已知问题分析

### 问题 1：类型判断缺少指导

**现象**：LLM 容易混淆相似类型，例如：
- 提取为 pattern，但实际更适合 case（有完整的问题解决过程）
- 提取为 entity，但实际是 profile（用户自己的属性）
- 提取为 event，但实际是 case（包含问题解决）

**根本原因**：

**Phase 1 的类型预测粗糙**：
- `_SPAN_PROMPT` 只提供枚举列表：`profile, preference, entity, event, case, pattern, skill, tool`
- 没有详细的类型判断指导
- 没有说明类型边界和区分标准

**Phase 2 的类型选择缺少决策树**：
- Schema description 不够详细（只有简单示例）
- Lazy mode prompt 没有类型选择指导
- LLM 只能根据 tool name 的简单描述猜测类型

**边界模糊的场景示例**：

```python
# 场景："我发现用户周五都喜欢跟进项目"
# → pattern: "friday_followup_pattern" (只是观察)
# → NOT skill: 缺少明确的 trigger + steps + completion
# → NOT case: 没有具体问题解决

# 场景："我帮助用户解决了 API timeout 问题，通过添加超时重试机制"
# → case: "debug_api_timeout" (有 problem + solution + outcome)
# → NOT pattern: 这不是观察，是实际解决的问题
# → NOT skill: 这是单一案例，不是可复用的 workflow

# 场景："Alice 说她在 engineering 团队"
# → profile: "occupation" (用户自己的属性)
# → NOT entity: Alice 是用户自己，不是"外部"实体
```

---

### 问题 2：Routing Key 规范化不足

**现象**：
- 同一实体提取为多个 routing_key：`alice`、`alice_engineer`、`alice_from_marketing`
- 同一主题提取为多个 routing_key：`coffee`、`coffee_preference`、`alice_likes_coffee`
- Routing key 不稳定：abstract 变化 → routing_key 变化

**根本原因**：

**LLM 主导生成缺少规范化规则**：
- Schema description 只是简单示例："normalized name, e.g., 'alice'"
- 没有强调"canonical name"（唯一且稳定）
- 没有禁止"过度细化"（例如：`alice_engineer` vs `alice`）
- 没有要求先检查已有 routing_key

**系统兜底生成不稳定**：

```python
# extraction/tool_builder.py:137
slug = "_".join(abstract.lower().split()[:3]).replace(",", "").replace(".", "")

# abstract "Alice is engineer" → routing_key = "alice_is_engineer"
# abstract "Alice works as engineer" → routing_key = "alice_works_as"
# 不同的 abstract → 不同的 routing_key → 语义重复
```

**缺少已有检查**：
- Lazy mode prompt 没有要求 LLM 先 `list` 查看已有 routing_key
- 没有提供工具让 LLM 检查语义相似的已有记忆

---

### 问题 3：重复检测机制缺失

**现象**：
- 语义重复：routing_key 不同但内容相似（`entity/alice` 和 `entity/alice_engineer`）
- 时间维度重复：新旧记忆描述同一事物（已有 `preference/coffee` "likes espresso"，新提取 "likes latte"）
- Upsert 类型未正确合并：应该更新已有记忆，但创建了新的 routing_key

**根本原因**：

**缺少语义相似度检测**：
- Lazy mode 没有 `search_similar` 工具
- 只有 `read` 和 `list` 工具（无法搜索相似内容）
- LLM 无法发现语义相似的已有记忆

**缺少时间维度分析**：
- 没有明确指导 LLM 分析新旧记忆的时间关系
- 没有指导 LLM 判断"偏好变化" vs "补充信息"

---

### 问题 4：冲突分析机制缺失

**现象**：
- 偏好变化未正确处理：用户说 "I no longer like X"，应该更新已有 preference，但可能创建了新的
- 信息矛盾未正确处理：已有 `entity/alice` "Alice is engineer"，新提取 "Alice is manager"，应该更新或标注矛盾

**根本原因**：

**缺少冲突处理指导**：
- Lazy mode prompt 没有指导 LLM 如何处理冲突
- 没有区分 upsert vs add_only 的冲突处理策略

**缺少 temporal 分析指导**：
- 没有指导 LLM 比较新旧记忆的时间戳
- 没有指导 LLM 判断"更新" vs "新记录" vs "标注矛盾"

---

## 现有实现分析

### 类型划分机制（Phase 1）

**代码位置**：`extraction/tools.py:111-137` (`_SPAN_PROMPT`)

**关键代码**：

```python
_SPAN_PROMPT = """\
For each span, specify:
- start: message index
- end: message index
- reason: what extractable information this span contains
- categories: likely categories from: profile, preference, entity, event, \
case, pattern, skill, tool
"""
```

**问题**：只有枚举列表，没有详细的类型判断指导。

---

### Routing Key 生成机制

**代码位置**：`extraction/tool_builder.py:134-141, 216-229`

**流程**：

```
1. LLM 生成的 tool_input
   ├─ routing_key: LLM 可提供（或缺失）
   ├─ abstract: 记忆摘要
   └─ ... 其他字段

2. parse_tool_call() 处理 routing_key
   ├─ LLM 提供了 routing_key？
   │  ├─ Yes → 使用 LLM 提供的值
   │  └─ No  → 从 abstract 自动生成
   │           (取前3个单词，lowercase，用_连接)
   │
   ├─ 是否是 event 或 case 类型？
   │  ├─ Yes → 追加时间戳保证唯一性
   │           (routing_key_20260521120000)
   │  └─ No  → 不追加
   │
   └─ routing_key 为空？
      ├─ Yes → 使用 "unnamed"
      └─ No  → 使用上述生成的值

3. 最终 routing_key → 用于生成 URI
```

**关键代码**：

```python
# Line 137：系统兜底生成
slug = "_".join(abstract.lower().split()[:3]).replace(",", "").replace(".", "")

# Line 220-223：event/case 追加时间戳
if schema.memory_type in ("event", "case"):
    ts = datetime.now(timezone.utc).strftime("%Y%m%d%H%M%S")
    routing_key = f"{routing_key}_{ts}" if routing_key else ts
```

**问题**：
- LLM 主导生成缺少规范化规则
- 系统兜底生成不稳定（依赖 abstract）
- 缺少已有 routing_key 检查

---

### Schema 定义示例

**entity.yaml**：

```yaml
- name: routing_key
  type: string
  required: true
  description: "Entity identifier (normalized name, e.g., 'alice', 'coffee_shop_downtown')"
```

**问题**：description 只是简单示例，缺少规范化规则。

---

## 记忆类型操作模式对比

| 类型 | Operation Mode | 文件模式 | Routing Key 含义 | 可合并/更新 | 典型场景 |
|------|---------------|----------|-----------------|------------|---------|
| **profile** | upsert | 单文件 | 属性名（如 `name`、`location`） | ✅ | 用户说 "I now live in NYC" → 更新 location |
| **entity** | upsert | 多文件 | 实体名（如 `alice`） | ✅ | 新信息补充实体描述 |
| **pattern** | upsert | 多文件 | 模式名（如 `friday_followup_pattern`） | ✅ | 观察新实例补充模式 |
| **skill** | upsert | 多文件 | 技能名（如 `debug_protocol`） | ✅ | 完善技能步骤 |
| **preference** | upsert | 多文件 | 主题名（如 `coffee`） | ✅ | 偏好更新（"I no longer like X"） |
| **event** | add_only | 多文件 | 事件名（如 `meeting_with_team`）+ 时间戳 | ❌ | 会议、旅行等时间事件 |
| **case** | add_only | 多文件 | 问题类型（如 `debug_api_timeout`）+ 时间戳 | ❌ | 问题解决的完整案例 |
| **session_summary** | add_only | ？ | ？ | ❌ | 会话摘要 |
| **session_archive** | add_only | ？ | ？ | ❌ | 会话归档 |

**关键差异**：
- **Upsert**：routing_key 决定唯一性，相同 routing_key 的提取会合并/更新
- **Add_only**：每个提取都是新记录，不修改已有内容（可能追加到同一文件）

---

## 优化方案

### P0：Prompt 增强（立即实施）

**目标**：添加类型判断决策树和 routing key 规范化指导。

**实施点 1：修改 Phase 1 prompt**（`extraction/tools.py:111-137`）

```python
_SPAN_PROMPT = """\
Analyze this conversation and identify contiguous message ranges (spans) \
that contain personal information worth remembering.

...

For each span, specify:
- start: message index (the [N] prefix number)
- end: message index (inclusive)
- reason: what extractable information this span contains
- categories: likely categories from the following types:

**Type Selection Criteria**:

**Case vs Pattern vs Skill**:
- Case: Complete problem-resolution pair (problem → steps → outcome). MUST be RESOLVED.
- Pattern: Observation without solution ("I noticed...", "It seems...")
- Skill: Reusable workflow with trigger + steps + completion criteria

**Entity vs Profile vs Preference**:
- Entity: External things (people, places, organizations mentioned)
- Profile: User's own attributes (name, location, occupation)
- Preference: User's likes/dislikes about topics

**Event vs Case**:
- Event: Time-bounded occurrence (meeting, trip) - NO problem-solving
- Case: Problem-solving incident - MUST have resolution

If multiple types fit → choose the most specific one.
"""
```

**实施点 2：修改 Lazy mode prompt**（`extraction/extraction_react_loop.py:234-268`）

在 `_build_system_prompt()` 中添加：

```python
## Type Selection Guide

Before calling extract_* tools, analyze which type fits best:

**Case vs Pattern vs Skill**:
- Case: Complete problem-resolution pair (problem → steps → outcome). Must be RESOLVED.
- Pattern: Observation without solution ("I noticed...", "It seems...")
- Skill: Reusable workflow with trigger + steps + completion criteria

**Entity vs Profile vs Preference**:
- Entity: External things (people, places, organizations)
- Profile: User's own attributes (name, location, occupation)
- Preference: User's likes/dislikes about topics

**Event vs Case**:
- Event: Time-bounded occurrence (meeting, trip) - NO problem-solving
- Case: Problem-solving incident - MUST have resolution

If unsure, choose the most specific type. Multiple possible types → choose the one with clearer criteria.

## Routing Key Rules

Routing_key must be unique and stable:
- Use canonical identifier (e.g., 'alice' not 'alice_engineer' or 'alice_from_marketing')
- Check existing routing_keys via `list` tool before deciding
- For upsert types: same routing_key → merge/update existing
- For add_only types: different routing_key → create new entry

If similar routing_key exists:
- Read existing → decide: update existing OR create new routing_key
- Do NOT create duplicate routing_keys for same entity/topic
```

**预期效果**：
- 减少类型判断错误
- Routing key 更规范稳定
- 减少语义重复

---

### P1：Schema Description 增强（短期实施）

**目标**：在 schema definition 中增强 routing_key 的规范化指导。

**实施：修改各 schema 的 routing_key description**

```yaml
# entity.yaml
- name: routing_key
  type: string
  required: true
  description: "Entity identifier (normalized CANONICAL name, e.g., 'alice' not 'alice_engineer' or 'alice_from_marketing'). Use stable identifier that won't change when new info added. Check existing routing_keys via list tool before creating new one."

# preference.yaml
- name: routing_key
  type: string
  required: true
  description: "Topic identifier (lowercase, underscored, e.g., 'coffee' not 'coffee_preference' or 'alice_likes_coffee'). Use stable topic name that won't change when preference evolves. Check existing routing_keys via list tool."

# pattern.yaml
- name: routing_key
  type: string
  required: true
  description: "Pattern identifier (lowercase, underscored, e.g., 'friday_followup_pattern'). Use descriptive pattern name that captures the essence. Check existing patterns to avoid duplicate observations."
```

**预期效果**：
- LLM 看到 schema 时即获得规范化指导
- 减少过度细化或不稳定的 routing_key

---

### P2：重复检测机制（中期实施）

**目标**：添加语义相似度检测能力。

**方案 1：添加 `search_similar` 工具**

```python
# 新增工具定义
_SIMILAR_SEARCH_TOOL = {
    "name": "search_similar",
    "description": "Search for semantically similar existing memories. Use before extract to check duplication.",
    "input_schema": {
        "type": "object",
        "properties": {
            "query": {"type": "string", "description": "Search query (abstract or content)"},
            "category": {"type": "string", "description": "Optional category filter"},
            "top_k": {"type": "integer", "default": 5}
        },
        "required": ["query"]
    }
}

# Lazy mode prompt 中添加指导
"Before extract_* call, use search_similar to check if similar memory exists.
If found (similarity > 0.8):
  - For upsert types: read existing → decide merge vs new routing_key
  - For add_only types: check if truly different event/case"
```

**依赖**：需要 VectorIndex 支持（需确认是否已有）。

**方案 2：改进系统兜底生成的 routing_key**

在 `parse_tool_call()` 中：
- 使用更稳定的 routing_key 生成策略（例如：使用 schema-specific 的 canonical identifier）
- 对于 upsert 类型，尝试匹配已有 routing_key（通过语义相似度）

**预期效果**：
- 检测语义重复（routing_key 不同但内容相似）
- 减少 duplicate entries

---

### P3：冲突分析机制（长期实施）

**目标**：添加冲突处理指导。

**实施：在 Lazy mode prompt 中添加**

```python
## Conflict Resolution Guide

When reading existing memory that conflicts with new extraction:

**For Upsert Types (profile, entity, preference)**:
- Temporal analysis: Which is more recent? (use 'when' field)
- Context analysis: Different contexts? (e.g., work vs home preference)
- Confidence comparison: Which is more confident?
- Decision: Update existing if new info is more recent/confident, OR create new routing_key if different context

**For Add_only Types (event, case)**:
- Check if truly different: Different time? Different problem?
- If same: Skip extraction (already recorded)
- If different: Proceed with new event/case

**Special Cases**:
- Preference change: User says "I no longer like X" → Update preference routing_key
- Entity evolution: Person changed role → Update entity with new info + timestamp
- Conflict evidence: Include both old and new in content with timestamps
```

**预期效果**：
- 正确处理偏好变化
- 正确处理信息矛盾
- 标注冲突证据

---

## 实施优先级

| 优先级 | 方案 | 实施难度 | 预期效果 | 依赖 |
|--------|------|----------|---------|------|
| **P0** | Prompt 增强（Phase 1 + Phase 2） | 低（修改 prompt） | 高（减少类型错误、routing_key 重复） | 无 |
| **P1** | Schema description 增强 | 低（修改 YAML） | 中（增强规范化指导） | 无 |
| **P2** | 重复检测机制（search_similar 工具） | 中（新增工具 + handler） | 高（检测语义重复） | VectorIndex |
| **P3** | 冲突分析机制 | 低（修改 prompt） | 中（正确处理冲突） | 无 |

**建议**：先实施 P0 和 P1，观察效果后再决定是否实施 P2 和 P3。

---

## 测试验证策略

### 测试场景 1：类型判断准确性

**输入**：
```
对话："我注意到用户每次周五都会要求跟进项目进度，然后我帮助他制定了一个周五跟进流程：先检查进度 → 发送提醒 → 记录反馈 → 确认完成"
```

**预期**：
- Phase 1: categories = ["pattern", "skill"]（两个都可能）
- Phase 2: LLM 应选择 `extract_skill`（有 trigger + steps + completion criteria）

**验证**：检查 LLM 是否正确选择 skill 类型。

---

### 测试场景 2：Routing Key 规范化

**输入**：
```
对话："Alice 提到她现在住在 NYC"
已有记忆：entity/alice（描述：Alice from engineering team）
```

**预期**：
- LLM 应先 `list` 查看已有 routing_key
- 发现 `alice` 已存在 → `read` 查看内容
- 决策：更新 entity/alice（添加 location 信息）

**验证**：检查是否正确更新已有 routing_key，而非创建 `alice_nyc`。

---

### 测试场景 3：重复检测

**输入**：
```
对话："Alice 说她喜欢 coding"
已有记忆：preference/coding（描述：User likes programming）
```

**预期**：
- LLM 使用 `search_similar` 发现已有 preference/coding
- 决策：更新 preference/coding（添加新的 evidence）

**验证**：检查是否正确更新而非创建 `alice_likes_coding`。

---

### 测试场景 4：冲突处理

**输入**：
```
对话："Alice 说她不再喜欢 coffee 了"
已有记忆：preference/coffee（描述：User likes espresso）
```

**预期**：
- LLM `read` 已有 preference/coffee
- Temporal 分析：新对话更近期
- 决策：更新 preference/coffee（标注："Previously liked espresso, now no longer likes coffee"）

**验证**：检查是否正确标注偏好变化。

---

## 相关文档

- [[Lazy Mode Empty Database Bug Fix]]：修复 Lazy 模式空数据库场景的 bug
- [[oG-Memory ReactLoop Abstract Base Class]]：ReAct 模式抽象基类设计
- [[ogmemory-architecture]]：oG-Memory 整体架构
- [[oG-Memory Extraction and Storage Analysis]]：Extraction 和 Storage 分析

---

## 变更日志

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.0-draft | 2026-05-21 | 初版：问题分析 + 优化方案 |