---
type: source
title: "OpenClaw MEMORY.md Lifecycle"
status: active
tags: [openclaw, memory, lifecycle, write, read, truncation]
created: 2026-05-12
updated: 2026-05-12
related:
  - "[[OpenClaw MEMORY.md Implementation]]"
  - "[[OpenClaw Memory System Overview]]"
  - "[[OpenClaw Memory Architecture Analysis]]"
  - "[[OpenClaw Memory Research Corrections]]"
---

# MEMORY.md 生命周期

从 MEMORY.md 文件本身的视角，分析它何时被写入、如何写入、何时被读取、如何截断。

---

## 总览时间线

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        新会话启动                                       │
│                                                                         │
│  ① Bootstrap 加载：读取 MEMORY.md → 注入 prompt（带截断）              │
│  ② Startup Context：读取 memory/YYYY-MM-DD.md（今天+昨天）             │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                        对话进行中                                       │
│                                                                         │
│  ③ memory_search：搜索索引中的 MEMORY.md 片段                          │
│  ④ memory_get：精确读取 MEMORY.md 指定行                               │
│  ⑤ Agent 手动编辑：可增/删/改 MEMORY.md 内容                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                   Compaction 前（Memory Flush）                         │
│                                                                         │
│  ⑥ Flush turn：MEMORY.md 视为只读，只写 memory/YYYY-MM-DD.md           │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                   Heartbeat / Cron（Dreaming）                          │
│                                                                         │
│  ⑦ Rehydrate：读取 memory/*.md 验证候选条目                            │
│  ⑧ Deep Phase：读取 MEMORY.md → 追加新 section → 全文件覆写           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 一、写入路径

### 1. Agent 手动编辑（正常对话）

**时机**：Agent 在正常对话 turn 中，通过通用 `write` 工具直接编辑。

**操作类型**：可增、可删、可改（无限制）。

官方文档原话：
> "Over time, the agent is expected to distill useful material from daily notes into MEMORY.md and remove stale long-term entries."

**特点**：
- Agent 自主决定写入内容
- 可以编辑或删除已有条目（不同于 Dreaming 的只追加）
- 这是 MEMORY.md 唯一可以被**删除或修改已有内容**的路径

**代码**：Agent 通过 `api.registerTool` 注册的通用 write 工具操作。

### 2. Dreaming Deep Phase（自动写入）

**时机**：cron 定时 或 heartbeat 检测到 pending dreaming event 时。

**代码**：`extensions/memory-core/src/short-term-promotion.ts:1551-1686`

**操作流程**：

```
读取 .dreams/ 短期召回存储（候选条目）
    ↓
评分 + 阈值过滤（minScore=0.75, minRecallCount=3, minUniqueQueries=2）
    ↓
重水化：从 memory/*.md 活文件验证候选内容是否仍存在
    ↓
读取现有 MEMORY.md，提取已有 HTML 注释标记用于去重
    ↓
过滤掉已晋升的候选（通过标记比对）
    ↓
构建新 section（buildPromotionSection）
    ↓
全文件覆写：原有内容原样保留 + 末尾追加新 section
```

**去重机制详解**：

每次从 `.dreams/` 晋升候选条目到 MEMORY.md 时，会同时写入一条 HTML 注释标记。下次晋升前，从 MEMORY.md 读回这些标记来过滤 `.dreams/` 的新候选。

标记格式：`<!-- openclaw-memory-promotion:${source}:${path}:${startLine}-${endLine}:${claimHash} -->`

从 MEMORY.md 提取已有标记（`extractPromotionMarkers`，第 1539-1548 行）：

```typescript
function extractPromotionMarkers(memoryText: string): Set<string> {
  const markers = new Set<string>();
  const matches = memoryText.matchAll(/<!--\s*openclaw-memory-promotion:([^\n]+?)\s*-->/gi);
  for (const match of matches) {
    markers.add(match[1]?.trim());
  }
  return markers;
}
```

过滤候选（第 1635-1639 行）：

```typescript
const existingMarkers = extractPromotionMarkers(existingMemory);
const alreadyWritten = rehydratedSelected.filter(c => existingMarkers.has(c.key));
const toAppend = rehydratedSelected.filter(c => !existingMarkers.has(c.key));
```

此外还有**双重去重**：`.dreams/` 中已被晋升的条目会被设置 `promotedAt` 字段，在源头就过滤掉（第 1579-1580 行）。两道防线作用相同但位置不同：
- **MEMORY.md 中的 HTML 注释**：防止同一内容被重复追加（即使 `.dreams/` 中的 `promotedAt` 丢失也能兜底）
- **.dreams/ 中的 promotedAt**：在源头过滤，避免不必要的重水化操作

**全文件覆写但旧内容不变**：

实际写入操作是读取整个 MEMORY.md，原样保留原有内容，在末尾追加新 section，然后一次性写回：

```typescript
// short-term-promotion.ts:1629-1648
const existingMemory = await fs.readFile(memoryPath, "utf-8")
  .catch(err => err.code === "ENOENT" ? "" : throw err);
// ...
if (toAppend.length > 0) {
  const header = existingMemory.trim().length > 0 ? "" : "# Long-Term Memory\n\n";
  const section = buildPromotionSection(toAppend, nowMs, timezone);
  await fs.writeFile(memoryPath,
    `${header}${withTrailingNewline(existingMemory)}${section}`, "utf-8");
}
```

之所以用 `writeFile` 而非 `appendFile`，是因为需要在首次写入时添加 `# Long-Term Memory` 标题，且需要保证尾部换行格式。旧的 section 原封不动保留，只有新 section 追加到末尾。

**写入格式**：

```markdown
## Promoted From Short-Term Memory (2026-05-12)

<!-- openclaw-memory-promotion:memory:2024-01-15.md:10-20:a1b2c3 -->
- 用户偏好 TypeScript 和 React [score=0.850 recalls=5 avg=0.850 source=memory/2024-01-15.md:10-20]
<!-- openclaw-memory-promotion:memory:2024-01-16.md:5-12:d4e5f6 -->
- 项目使用 pnpm 管理 [score=0.780 recalls=4 avg=0.780 source=memory/2024-01-16.md:5-12]
```

**重要**：虽然逻辑上是"追加"，但实际实现是**全文件覆写**：
```typescript
// short-term-promotion.ts:1644-1648
const header = existingMemory.trim().length > 0 ? "" : "# Long-Term Memory\n\n";
const section = buildPromotionSection(toAppend, nowMs, timezone);
await fs.writeFile(memoryPath,
  `${header}${withTrailingNewline(existingMemory)}${section}`, "utf-8");
```

### 3. CLI 手动晋升

**命令**：`openclaw memory promote --apply`

**代码**：`extensions/memory-core/src/cli.runtime.ts:1285-1301`

最终调用 `applyShortTermPromotions`，与 Dreaming Deep Phase 完全相同。

### 4. Flush 期间：明确禁止写入 MEMORY.md

**Flush 是什么**：compaction 前的自动保存机制。当对话上下文接近 compaction 阈值（即将被摘要压缩）时，系统插入一个**静默的 agent turn**，提醒 agent 把重要内容保存到文件中，避免压缩导致信息丢失。

```
对话上下文接近 compaction 阈值
    ↓
插入隐藏的 flush turn
    ↓
Agent 决定哪些内容值得保存
    ↓
写入 memory/YYYY-MM-DD.md（追加，只写每日笔记）
    ↓
MEMORY.md 在此期间被视为只读，禁止修改
```

**为什么禁止写 MEMORY.md**：Flush 发生在对话中途，如果允许直接改 MEMORY.md，可能导致不一致。因此只允许向每日笔记追加内容。

**触发条件**（`flush-plan.ts`）：
- 距 compaction 阈值 4000 tokens 时触发
- 或 transcript 达到 2MB 时强制触发

**代码**：`extensions/memory-core/src/flush-plan.ts:17-18`

Flush prompt 明确指示：
> "Treat workspace bootstrap/reference files such as MEMORY.md, DREAMS.md, SOUL.md, TOOLS.md, and AGENTS.md as read-only during this flush; never overwrite, replace, or edit them."

**实现**：write 工具被 `wrapToolMemoryFlushAppendOnlyWrite` 包装，只能向 `memory/YYYY-MM-DD.md` 追加。

### 写入路径对比

| 路径 | 触发 | 操作 | 可修改已有内容 | 去重 |
|------|------|------|----------------|------|
| Agent 手动 | 正常对话 | 增/删/改 | **是** | 无 |
| Dreaming Deep | cron/heartbeat | 追加 section | 否 | HTML 注释标记 |
| CLI promote | 手动命令 | 追加 section | 否 | HTML 注释标记 |
| Flush | compaction 前 | **禁止** | — | — |

---

## 二、读取路径

### 1. Bootstrap 加载（每次会话启动）

**代码**：`src/agents/workspace.ts:615-680`

**加载顺序**（MEMORY.md 排第 8，最后）：

```
AGENTS.md → SOUL.md → TOOLS.md → IDENTITY.md → USER.md → HEARTBEAT.md → BOOTSTRAP.md → MEMORY.md
```

**缓存机制**：
- 缓存 key：`${canonicalPath}|${dev}:${ino}:${size}:${mtimeMs}`
- 文件未变化时命中缓存，不重复读取
- 最大读取 2MB（`MAX_WORKSPACE_BOOTSTRAP_FILE_BYTES`）
- MEMORY.md 不存在时直接跳过（不报错）

### 2. 注入时截断

**代码**：`src/agents/pi-embedded-helpers/bootstrap.ts:132-228`

MEMORY.md 内容加载后，注入 prompt 前需经过截断。

#### 单文件截断

```
上限：12,000 chars
比例：75% head + 25% tail
算法：迭代 3 次确定 head/tail 各保留多少字符
标记：[...truncated, read MEMORY.md for full content...]
详细：…(truncated MEMORY.md: kept 9000+3000 chars of 50000)…
紧凑：[…truncated 9000+3000/50000]（空间极度紧张时）
```

#### 总预算截断

```
上限：60,000 chars（所有 8 个 bootstrap 文件合计）
分配：按加载顺序依次分配
影响：MEMORY.md 排最后，剩余预算不足 64 chars 时跳过
```

**实际影响**：如果前 7 个文件占用了大部分预算，MEMORY.md 可能被大幅截断甚至完全跳过。

### 3. 搜索索引

**代码**：`extensions/memory-core/src/memory/manager.ts`

**索引过程**：
```
MEMORY.md 内容
    ↓ 分块（~400 token, 80 token 重叠）
    ↓ 提取 embedding
    ↓ 存入 SQLite
索引条目：{path, startLine, endLine, snippet, embedding}
```

**文件监听**：
- 使用 chokidar 监听 MEMORY.md 变更
- 1.5 秒防抖后触发重新索引
- embedding provider/model/chunking 配置变化时全量重建

### 4. memory_get 精确读取

**代码**：`packages/memory-host-sdk/src/host/read-file.ts:23-108`

```
memory_get(path="MEMORY.md", from=10, lines=5)
    ↓ 路径验证（必须在 workspace 内）
    ↓ 直接从文件系统读取（不经过 SQLite）
    ↓ 返回指定行范围的完整文本
```

### 5. Dreaming 重水化

**代码**：`extensions/memory-core/src/short-term-promotion.ts:1480-1509`

```
晋升前：
    ↓ 从活文件重新读取候选内容
    ↓ 验证文件是否存在、内容是否匹配
    ↓ 文件已删除 → 跳过该候选（防止"僵尸记忆"）
    ↓ 内容已变化 → 更新候选内容
```

**注意**：重水化主要读取 `memory/*.md`（短期召回的源文件），而非直接读取 MEMORY.md。

### 读取路径对比

| 路径 | 触发 | 截断 | 缓存 | 用途 |
|------|------|------|------|------|
| Bootstrap 加载 | 会话启动 | 75/25, 12K/60K | inode 缓存 | 注入 prompt |
| 搜索索引 | 文件变更 | 分块 ~400 token | — | memory_search |
| memory_get | 工具调用 | 无 | 无 | 精确读取 |
| 重水化 | Dreaming | 无 | 无 | 验证候选 |

---

## 三、截断机制详解

### Bootstrap 注入（MEMORY.md → prompt）

```
原始文件 50,000 chars
    ↓ 单文件截断
保留 9,000 (head) + 3,000 (tail) = 12,000 chars
丢弃 38,000 chars
标记插入：[...truncated, read MEMORY.md for full content...]
    ↓ 总预算截断
如果前 7 个文件已用 55,000 chars
MEMORY.md 预算 = 60,000 - 55,000 = 5,000 chars
    ↓ 进一步截断
保留 3,750 (head) + 1,250 (tail) = 5,000 chars
```

### Startup Context（每日笔记 → session prelude）

```
memory/YYYY-MM-DD.md（今天 + 昨天）
    ↓ 单文件：1,200 chars 上限
    ↓ 总预算：2,800 chars
    ↓ 回溯：2 天
    ↓ 超出时二分搜索裁剪
    ↓ 标注 [Untrusted daily memory: ...]
```

### 搜索索引分块

```
MEMORY.md 内容
    ↓ 分块器
~400 token/块，80 token 重叠
    ↓ FTS5 索引
BM25 关键词索引
    ↓ 向量索引（可选）
embedding 向量搜索
    ↓ 结果合并
混合搜索：向量语义 + 关键词匹配
```

---

## 四、写入格式与去重

### Dreaming 晋升的写入格式

首次写入时（MEMORY.md 为空或不存在）：
```markdown
# Long-Term Memory

## Promoted From Short-Term Memory (2026-05-12)

- 用户偏好 TypeScript [score=0.85 recalls=5 source=memory/2024-01-15.md:10-20]
<!-- openclaw-memory-promotion:memory:2024-01-15.md:10-20:a1b2c3 -->
```

后续晋升时（追加新 section）：
```markdown
（... 原有内容 ...）

## Promoted From Short-Term Memory (2026-05-13)

- 项目使用 React Router v6 [score=0.82 recalls=4 source=memory/2024-01-16.md:5-12]
<!-- openclaw-memory-promotion:memory:2024-01-16.md:5-12:d4e5f6 -->
```

### 去重流程

```typescript
// 1. 提取已有标记
const existingMarkers = extractPromotionMarkers(existingMemory);
// 正则：/<!--\s*openclaw-memory-promotion:([^\n]+?)\s*-->/gi

// 2. 过滤已晋升的候选
for (const candidate of candidates) {
  if (existingMarkers.has(candidate.key)) {
    continue; // 跳过重复
  }
  // ...
}

// 3. 写入新标记
section += `<!-- openclaw-memory-promotion:${candidate.key} -->\n`;
```

### Agent 手动编辑的内容

无固定格式。Agent 自主决定如何组织 MEMORY.md 内容。可能包含：
- 偏好列表
- 项目决策记录
- 重要事实摘要
- 任何 Agent 认为值得长期保留的信息

---

## 五、MEMORY.md 的"双重身份"与设计张力

MEMORY.md 身兼两个职责，这是它的核心设计张力：

| 职责 | 期望 | 矛盾 |
|------|------|------|
| **完整存储** | 所有记忆原文都在此文件中，不丢失 | 文件不断增长 |
| **Prompt 注入** | 紧凑、高信号，在预算内提供最有价值的上下文 | 预算有限，只能机械地 head+tail |

SQLite 索引虽然通过分块保留了完整内容的 embedding，但 **bootstrap 注入路径不走索引**，只读原始文件并机械截断。截断算法不选择"最重要的"内容，而是保留头部 75% 和尾部 25%，中间部分直接丢弃。

### 当前的妥协方案

OpenClaw 没有在架构层面解决这个问题，而是把"热信息摘要"的职责交给 Agent 的判断力：

> "Treat that as a signal to move detailed material back into memory/*.md, keep only the durable summary in MEMORY.md."

两个消费者对 MEMORY.md 的期望互相冲突：

| 消费者 | 期望 | 截断影响 |
|--------|------|----------|
| **Bootstrap prompt** | 紧凑、高信号的长期摘要 | 文件过大时被截断，丢失中间内容 |
| **搜索索引** | 完整、详细的记忆内容 | 分块索引，不丢失任何内容 |

- **Agent 手动编辑**倾向于精简 MEMORY.md（保持可 bootstrap 加加载的紧凑度）
- **Dreaming 自动晋升**倾向于追加内容（不删除已有条目）
- 文件增长到超出 bootstrap 预算时，截断导致中间内容丢失

### 理论上的改进方向

将存储和注入拆分为独立层：

```
当前设计（一个文件两个用途）：
MEMORY.md ← 完整存储 + bootstrap 注入（截断）

拆分设计：
memory-store.md      ← 完整存储（供搜索索引、memory_get）
                       不受 prompt 预算限制，可无限增长
memory-hot.md        ← 精选摘要（供 bootstrap 注入）
                       始终控制在预算内，不需要截断
```

这样 `memory-hot.md` 可以始终控制在预算内，不需要截断算法；而 `memory-store` 不受 prompt 预算限制，完整保留所有记忆。两者之间的关系类似于 CPU 缓存与主存：热数据在缓存中，完整数据在主存中。

---

## 六、数据流总图

```
                         写入 MEMORY.md
                              │
            ┌─────────────────┼───────────────────┐
            │                 │                     │
     Agent 手动编辑    Dreaming Deep Phase    CLI promote
     （增/删/改）      （追加 section）      （同 Deep）
            │                 │                     │
            │          ┌──────┘                     │
            │          │ Rehydrate 验证             │
            │          │ 从 memory/*.md 读取        │
            │          └──────┐                     │
            ▼                 ▼                     ▼
      ┌──────────────────────────────────────────────────┐
      │                  MEMORY.md                        │
      │                                                   │
      │  # Long-Term Memory                               │
      │  （Agent 手动写入的内容）                           │
      │  ...                                              │
      │  ## Promoted From Short-Term Memory (date1)       │
      │  - item1 <!-- promotion marker -->                │
      │  ## Promoted From Short-Term Memory (date2)       │
      │  - item2 <!-- promotion marker -->                │
      └──────────────────────────────────────────────────┘
            │                 │                 │
            ▼                 ▼                 ▼
      Bootstrap 加载    搜索索引分块      memory_get
      （75/25 截断     （~400 token）    （精确读取）
       12K/60K 预算）
            │
            ▼
      注入 Agent prompt
      （带截断标记）

      ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
      Flush 期间：MEMORY.md 只读，写 memory/*.md
      ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
```

---

## 七、关键代码索引

| 功能 | 文件 | 行号 |
|------|------|------|
| Bootstrap 加载 | `src/agents/workspace.ts` | 615-680 |
| 单文件截断 | `src/agents/pi-embedded-helpers/bootstrap.ts` | 132-228 |
| 总预算截断 | `src/agents/pi-embedded-helpers/bootstrap.ts` | 271-330 |
| 每日笔记加载 | `src/auto-reply/reply/startup-context.ts` | 309-386 |
| Deep Phase 写入 | `extensions/memory-core/src/short-term-promotion.ts` | 1551-1686 |
| 晋升 section 构建 | `extensions/memory-core/src/short-term-promotion.ts` | 1511-1530 |
| 去重标记提取 | `extensions/memory-core/src/short-term-promotion.ts` | 1535-1549 |
| 重水化 | `extensions/memory-core/src/short-term-promotion.ts` | 1480-1509 |
| Flush 只读限制 | `extensions/memory-core/src/flush-plan.ts` | 17-18 |
| Flush write 包装 | `src/agents/pi-tools.ts` | 880-892 |
| 文件监听 | `extensions/memory-core/src/memory/manager-sync-ops.ts` | 424 |
| memory_get | `packages/memory-host-sdk/src/host/read-file.ts` | 23-108 |
| CLI promote | `extensions/memory-core/src/cli.runtime.ts` | 1285-1301 |

---

### 相关 Wiki 页面

- [[OpenClaw MEMORY.md Implementation]] — 写入/读取/截断的源码级分析
- [[OpenClaw Memory System Overview]] — 系统综合介绍
- [[OpenClaw Memory Architecture Analysis]] — 架构与代码对应
- [[OpenClaw Memory Data Flow]] — 数据流时间点
- [[OpenClaw Memory Research Corrections]] — 调研误解与纠正（含重水化解释）
