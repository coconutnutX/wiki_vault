---
type: source
title: "OpenClaw Memory Data Flow"
status: active
tags: [openclaw, memory, data-flow, timing]
created: 2026-05-12
updated: 2026-05-12
related:
  - "[[OpenClaw Memory System Overview]]"
  - "[[OpenClaw Memory Backend Comparison]]"
  - "[[OpenClaw Memory Architecture Analysis]]"
  - "[[OpenClaw Memory Research Corrections]]"
  - "[[OpenClaw MEMORY.md Implementation]]"
---

# OpenClaw Memory 数据流分析：组合与时间点

本文详细分析 Enhancement Layer 的组合可能性，以及各组合下数据流转的具体时间点。

---

## 一、组合可行性

### Enhancement Layer 可以自由组合

**是的**，Enhancement Layer 的各模块可以与任何 Memory 插件组合：

| 组合 | 可行性 | 说明 |
|------|--------|------|
| memory-core + QMD + Active Memory | ✓ 可行 | QMD 是 memory-core 的搜索后端，Active Memory 调用 memory-core 工具 |
| memory-lancedb + Commitments | ✓ 可行 | memory-lancedb 有自己的 auto hooks，Commitments 是独立功能 |
| memory-core + Active Memory + Commitments | ✓ 可行 | 三者共存 |
| memory-lancedb + Active Memory + Commitments | ✓ 可行 | Active Memory 自动适配 memory-lancedb 的工具 |

### 禁止的组合

| 组合 | 原因 |
|------|------|
| memory-core + memory-lancedb | **互斥**：两者是并列的 Memory 插件，通过槽位选择其一 |
| builtin + qmd 后端 | **互斥**：两者是 memory-core 的搜索后端，通过 `memory.backend` 选择 |
| Dreaming + memory-lancedb | **功能绑定**：Dreaming 是 memory-core 的功能，依赖 MEMORY.md |
| Flush + memory-lancedb | **功能绑定**：Flush 是 memory-core 的功能（`flush-plan.ts`），替换后丢失 |

> **补充**（2026-05-12）：Flush 机制位于 `extensions/memory-core/src/flush-plan.ts`，是 memory-core 的一部分。替换为 memory-lancedb 后，compaction 前的自动保存功能将不可用。参见 [[OpenClaw Memory Backend Comparison]]。

---

## 二、组合一：memory-core + QMD + Active Memory

### 架构

```
┌─────────────────────────────────────────────────────────────┐
│                    memory-core 插件                          │
│                                                              │
│  存储层：MEMORY.md + memory/*.md（人类可编辑文件）            │
│  索引层：QMD sidecar（reranking + query expansion）          │
│  工具：memory_search, memory_get                            │
└─────────────────────────────────────────────────────────────┘
                    +
┌─────────────────────────────────────────────────────────────┐
│                    active-memory 插件                        │
│                                                              │
│  Hook: before_prompt_build                                  │
│  调用：memory_search → memory_get                           │
│  输出：prependContext（隐藏注入到 prompt）                   │
└─────────────────────────────────────────────────────────────┘
```

### 数据流时间点

```
┌─────────────────────────────────────────────────────────────────────┐
│                          用户发送消息                                │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│ T1: before_prompt_build（Active Memory hook）                       │
│                                                                      │
│  输入：latest user message + recent turns                           │
│  操作：                                                              │
│    1. 构建搜索 query                                                 │
│    2. 启动阻塞式子 Agent                                             │
│    3. 子 Agent 调用 memory_search（通过 QMD）                        │
│    4. 子 Agent 调用 memory_get 读取文件片段                          │
│    5. 判断相关性 → 返回 NONE 或摘要                                  │
│  输出：prependContext（隐藏系统上下文）                              │
│  存储：无（只检索）                                                  │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│ T2: 主 Agent 生成回复                                                │
│                                                                      │
│  Prompt 包含：                                                      │
│    - 用户消息                                                        │
│    - Active Memory 注入的隐藏上下文                                  │
│    - 启动时加载的 MEMORY.md                                          │
│    - 今日/昨日的 memory/*.md                                         │
│  Agent 可主动调用：                                                  │
│    - memory_search（搜索索引）                                       │
│    - memory_get（读取文件）                                          │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│ T3: agent_end（可选）                                                │
│                                                                      │
│  memory-core 本身无 agent_end hook                                   │
│  但 Agent 可手动调用 write 工具写入：                                 │
│    - memory/YYYY-MM-DD.md（每日笔记）                                │
│    - MEMORY.md（长期记忆，手动编辑）                                  │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│ T4: Compaction 前（Memory Flush）                                    │
│                                                                      │
│  触发：会话接近 auto-compaction 阈值                                  │
│  操作：                                                              │
│    1. 运行一个隐藏的 memory flush turn                                │
│    2. Agent 决定是否有需要保存的内容                                  │
│    3. 写入 memory/YYYY-MM-DD.md（追加，不覆盖）                       │
│  限制：                                                              │
│    - MEMORY.md 只读（flush 不修改）                                  │
│    - 只追加到 memory/*.md                                            │
│  注意：Flush 是 memory-core 功能，替换插件后不可用                    │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│ T5: Heartbeat / Cron（Dreaming，需启用）                             │
│                                                                      │
│  触发：heartbeat 或 cron（如 "0 3 * * *"）                           │
│  操作：                                                              │
│    1. Light Phase：收集短期信号 → memory/.dreams/                    │
│    2. REM Phase：反思模式 → memory/.dreams/                         │
│    3. Deep Phase：评分候选 → 写入 MEMORY.md                          │
│    4. 写入 DREAMS.md（Dream Diary，人类可审查）                      │
│  输出：                                                              │
│    - MEMORY.md 追加长期记忆                                          │
│    - DREAMS.md 追加日记条目                                          │
└─────────────────────────────────────────────────────────────────────┘
```

### 检索与存储总结

| 时间点 | 模块 | 操作 | 存储位置 |
|--------|------|------|----------|
| 启动 | memory-core | 加载 MEMORY.md + 今日/昨日 memory/*.md | 无存储，加载到 prompt |
| T1 (before_prompt_build) | Active Memory | 检索 | 无存储 |
| T2 (主回复) | Agent | 可主动调用 memory_search/get | 无存储 |
| T3 (agent_end) | Agent | 可手动写入 | memory/YYYY-MM-DD.md |
| T4 (Compaction 前) | Memory Flush | 自动写入 | memory/YYYY-MM-DD.md |
| T5 (Heartbeat/Cron) | Dreaming | 自动提升 | MEMORY.md |

### 配置示例

```json5
{
  plugins: {
    slots: {
      memory: "memory-core"
    },
    entries: {
      "memory-core": {
        config: {
          dreaming: {
            enabled: true,
            frequency: "0 3 * * *"
          }
        }
      },
      "active-memory": {
        enabled: true,
        config: {
          agents: ["main"],
          allowedChatTypes: ["direct"],
          queryMode: "recent",
          timeoutMs: 15000
        }
      }
    }
  },
  memory: {
    backend: "qmd",
    qmd: {
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }]
    }
  }
}
```

---

## 三、组合二：memory-lancedb + Commitments

### 架构

```
┌─────────────────────────────────────────────────────────────┐
│                    memory-lancedb 插件                       │
│                                                              │
│  存储层：LanceDB 表（MemoryEntry）                            │
│  工具：memory_recall, memory_store, memory_forget           │
│                                                              │
│  Auto Hooks：                                                │
│    - before_prompt_build: auto-recall                       │
│    - agent_end: auto-capture                                 │
└─────────────────────────────────────────────────────────────┘
                    +
┌─────────────────────────────────────────────────────────────┐
│                    commitments（核心功能）                    │
│                                                              │
│  存储层：~/.openclaw/commitments/commitments.json            │
│                                                              │
│  触发：                                                      │
│    - 提取：agent 回复后（queue + debounce）                   │
│    - 交付：heartbeat 检查到期                                 │
└─────────────────────────────────────────────────────────────┘
```

### 数据流时间点

```
┌─────────────────────────────────────────────────────────────────────┐
│                          用户发送消息                                │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│ T1: before_prompt_build（memory-lancedb auto-recall）               │
│                                                                      │
│  触发条件：config.autoRecall = true                                  │
│  输入：latest user message                                           │
│  操作：                                                              │
│    1. embedding(query)                                               │
│    2. LanceDB.vectorSearch(vector, 3, 0.3)                          │
│    3. 转换 L2 distance → similarity score                           │
│  输出：prependContext（格式化的记忆条目）                            │
│  存储位置：无（只检索 LanceDB）                                       │
│  结果格式：                                                          │
│    "Found N memories:\n                                             │
│     1. [preference] 用户偏好 TypeScript...                          │
│     2. [fact] 项目使用 React..."                                     │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│ T2: 主 Agent 生成回复                                                │
│                                                                      │
│  Prompt 包含：                                                      │
│    - 用户消息                                                        │
│    - auto-recall 注入的记忆                                          │
│  Agent 可主动调用：                                                  │
│    - memory_recall（搜索 LanceDB）                                   │
│    - memory_store（存储新记忆）                                      │
│    - memory_forget（删除记忆）                                       │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│ T3: agent_end（memory-lancedb auto-capture）                         │
│                                                                      │
│  触发条件：config.autoCapture = true                                 │
│  输入：event.messages（完整对话）                                    │
│  操作：                                                              │
│    1. 提取用户消息文本                                               │
│    2. 判断是否值得捕获（shouldCapture）                              │
│    3. detectCategory(text) → preference/fact/decision/other         │
│    4. embedding(text)                                                │
│    5. 检查重复（search(vector, 1, 0.95)）                            │
│    6. store({ text, vector, importance: 0.7, category })            │
│  存储位置：LanceDB 表                                                │
│  条目格式：                                                          │
│    {                                                                │
│      id: UUID,                                                      │
│      text: "原始用户文本",                                           │
│      vector: [0.1, 0.2, ...],                                       │
│      importance: 0.7,                                               │
│      category: "preference",                                        │
│      createdAt: timestamp                                           │
│    }                                                                │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│ T4: 回复后 1-2 秒（Commitments 提取）                                │
│                                                                      │
│  触发条件：commitments.enabled = true                                │
│  输入：                                                              │
│    - userText: 用户消息                                              │
│    - assistantText: Agent 回复                                       │
│  操作：                                                              │
│    1. enqueueCommitmentExtraction → queue                           │
│    2. debounce (~秒级延迟)                                           │
│    3. drainCommitmentExtractionQueue                                 │
│    4. 启动 embedded Pi agent（隐藏 LLM pass）                        │
│    5. LLM 推断是否有 follow-up commitment                           │
│  存储位置：                                                          │
│    ~/.openclaw/commitments/commitments.json                         │
│  Commitment 格式：                                                   │
│    {                                                                │
│      id: "cm_xxx",                                                  │
│      kind: "event_check_in",                                        │
│      sensitivity: "routine",                                        │
│      suggestedText: "How did the interview go?",                    │
│      dueWindow: { earliestMs, latestMs, timezone },                 │
│      agentId, sessionKey, channel（作用域）                          │
│    }                                                                │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│ T5: Heartbeat（Commitments 交付）                                    │
│                                                                      │
│  触发：周期性 heartbeat run                                           │
│  操作：                                                              │
│    1. listPendingCommitmentsForScope                                │
│    2. 筛选到期且 maxPerDay 未超限                                     │
│    3. 添加到 heartbeat prompt                                        │
│    4. Agent 发送 check-in 或回复 HEARTBEAT_OK                       │
│  交付条件：                                                          │
│    - nowMs >= earliestMs                                             │
│    - maxPerDay 未用完                                                │
│    - 同 agent + 同 channel                                           │
│  状态变更：pending → sent/dismissed/snoozed                         │
└─────────────────────────────────────────────────────────────────────┘
```

### 检索与存储总结

| 时间点 | 模块 | 操作 | 存储位置 |
|--------|------|------|----------|
| T1 (before_prompt_build) | memory-lancedb | auto-recall | LanceDB（检索） |
| T2 (主回复) | Agent | 可调用 memory_recall/store/forget | LanceDB |
| T3 (agent_end) | memory-lancedb | auto-capture | LanceDB（存储） |
| T4 (回复后) | Commitments | 提取推断 | commitments.json |
| T5 (Heartbeat) | Commitments | 交付 check-in | commitments.json（状态更新） |

### 配置示例

```json5
{
  plugins: {
    slots: {
      memory: "memory-lancedb"
    },
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            provider: "openai",
            model: "text-embedding-3-small"
          },
          autoRecall: true,
          autoCapture: true,
          captureMaxChars: 500,
          recallMaxChars: 1000
        }
      }
    }
  },
  commitments: {
    enabled: true,
    maxPerDay: 3
  }
}
```

---

## 四、组合对比

### 检索时机

| 组合 | 检索时机 | 检索方式 | 返回内容 |
|------|----------|----------|----------|
| memory-core + Active Memory | before_prompt_build | 子 Agent 调用 memory_search/get | 文件片段引用+摘要 |
| memory-lancedb | before_prompt_build | plugin 内置 hook | 独立条目文本 |
| commitments | heartbeat | 检查 commitments.json | check-in 建议 |

### 存储时机

| 组合 | 存储时机 | 存储方式 | 存储位置 |
|------|----------|----------|----------|
| memory-core | Compaction 前 | Memory Flush turn | memory/*.md |
| memory-core | Heartbeat/Cron | Dreaming Deep Phase | MEMORY.md |
| memory-core | Agent 手动 | write 工具 | MEMORY.md 或 memory/*.md |
| memory-lancedb | agent_end | auto-capture hook | LanceDB 表 |
| memory-lancedb | Agent 手动 | memory_store 工具 | LanceDB 表 |
| commitments | 回复后 debounce | hidden LLM extraction | commitments.json |

### 存储内容类型

| 组合 | 存储内容 | 结构 |
|------|----------|------|
| memory-core | 任意笔记、事实、决策 | Markdown 文本（人类可编辑） |
| memory-lancedb | 分类记忆条目 | `{id, text, category, importance}` |
| commitments | 推断的 follow-up | `{kind, suggestedText, dueWindow}` |

---

## 五、代码路径参考

### Active Memory

```
触发点：extensions/active-memory/index.ts:3017
    api.on("before_prompt_build", ...)

检索逻辑：extensions/active-memory/index.ts:3095
    maybeResolveActiveRecall → 调用 memory_search/get

输出：extensions/active-memory/index.ts:3115
    return { prependContext: promptPrefix }
```

### memory-lancedb auto hooks

```
auto-recall：extensions/memory-lancedb/index.ts:988
    api.on("before_prompt_build", async (event) => {
        vector = embeddings.embed(event.prompt)
        results = db.search(vector, 3, 0.3)
        return { prependContext: formatRelevantMemoriesContext(results) }
    })

auto-capture：extensions/memory-lancedb/index.ts:1038
    api.on("agent_end", async (event) => {
        for text in extractUserTextContent(message):
            if shouldCapture(text):
                db.store({ text, vector, importance: 0.7, category })
    })
```

### Memory Flush（memory-core）

```
触发点：extensions/memory-core/src/flush-plan.ts
    Compaction 前的 silent turn

写入：memory/YYYY-MM-DD.md（追加）
限制：MEMORY.md 只读
```

### Commitments

```
提取触发：src/auto-reply/reply/agent-runner.ts:1534
    enqueueCommitmentExtractionForTurn →
    src/commitments/runtime.ts:101 enqueueCommitmentExtraction

提取逻辑：src/commitments/runtime.ts:266
    drainCommitmentExtractionQueue → embedded Pi agent

存储：src/commitments/store.ts
    ~/.openclaw/commitments/commitments.json

交付：src/infra/heartbeat-runner.ts:1303
    resolveHeartbeatRunPrompt → hasDueCommitments
```

### Dreaming（memory-core）

```
触发点：extensions/memory-core/src/dreaming.ts:866
    api.on("gateway_start", ...)  // 注册 cron
    api.on("before_agent_reply", ...) // heartbeat 检查

写入：extensions/memory-core/src/dreaming-phases.ts
    Deep Phase → 追加到 MEMORY.md
    DREAMS.md → Dream Diary
```

---

## 六、关键设计点

### 1. Active Memory 的阻塞特性

- **阻塞式**：before_prompt_build 必须在 timeoutMs 内完成
- **子 Agent**：独立上下文，只能调用 memory 工具
- **隐藏注入**：prependContext 不暴露给用户，直接进入主 Agent prompt

### 2. memory-lancedb 的非阻塞 auto hooks

- auto-recall 有独立 timeout（DEFAULT_AUTO_RECALL_TIMEOUT_MS）
- 超时时跳过注入，不阻塞主 Agent
- auto-capture 在 agent_end 后异步执行

### 3. Commitments 的异步提取

- **队列化**：enqueue → debounce → drain
- **隐藏 LLM pass**：不暴露给用户
- **作用域绑定**：同 agent + 同 channel
- **不立即交付**：至少一个 heartbeat 间隔后

### 4. Memory Flush vs Dreaming

| 特性 | Memory Flush | Dreaming |
|------|--------------|----------|
| 触发 | Compaction 前 | Heartbeat/Cron |
| 写入位置 | memory/*.md | MEMORY.md |
| 内容类型 | 当前会话笔记 | 长期记忆（评分后） |
| Agent 参与 | 是（flush turn） | 是（Deep Phase） |
| 所属插件 | memory-core | memory-core |
| 替换影响 | 丢失 | 丢失 |

