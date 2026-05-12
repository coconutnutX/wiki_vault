---
type: source
title: "OpenClaw Dreaming Mechanism"
status: active
tags: [openclaw, memory, dreaming, phases, recall-store]
created: 2026-05-12
updated: 2026-05-12
---

# Dreaming 机制分析

Dreaming 是 memory-core 的后台记忆巩固系统，负责从短期召回中筛选高信号条目晋升到 MEMORY.md。本分析侧重 `memory/.dreams/` 中的数据变化和三阶段的实际操作。

---

## 总览

```
memory_search 被调用
       ↓
每次搜索结果被记录到 short-term-recall.json
（recallCount++、totalScore 累加、queryHashes 追加）
       ↓
cron / heartbeat 触发 Dreaming sweep
       ↓
┌─────────────────────────────────────────┐
│  Light Phase：收集 + 暂存短期信号       │
│  REM Phase：反思模式 + 主题            │
│  Deep Phase：评分 + 晋升到 MEMORY.md   │
└─────────────────────────────────────────┘
       ↓
MEMORY.md 追加晋升条目
DREAMS.md 追加 Dream Diary
```

---

## 一、短期召回存储（`memory/.dreams/`）

### 目录结构

```
memory/.dreams/
├── short-term-recall.json    ← 核心存储：所有短期召回条目
├── phase-signals.json        ← 各阶段命中计数（用于评分加成）
├── short-term-promotion.lock ← 晋升时的文件锁
├── session-corpus/           ← 会话 transcript 分片
│   └── YYYY-MM-DD.txt
├── session-ingestion.json    ← 会话 ingest 状态
└── daily-ingestion.json      ← 每日笔记 ingest 状态
```

### short-term-recall.json 结构

```typescript
type ShortTermRecallStore = {
  version: 1;
  updatedAt: string;
  entries: Record<string, ShortTermRecallEntry>;
};

type ShortTermRecallEntry = {
  key: string;           // 唯一标识：source:path:startLine-endLine:claimHash
  path: string;          // 源文件路径
  startLine: number;
  endLine: number;
  source: "memory";
  snippet: string;       // 文本片段
  recallCount: number;   // 被 memory_search 命中的次数
  dailyCount: number;    // 被 Light Phase 从每日笔记中发现的次数
  groundedCount: number; // 被 Grounded Backfill 记录的次数
  totalScore: number;    // 所有召回分数之和
  maxScore: number;      // 历史最高分数
  firstRecalledAt: string;
  lastRecalledAt: string;
  queryHashes: string[]; // 去重后的查询哈希（最多 32 个）
  recallDays: string[];  // 被召回的日期（最多 16 天）
  conceptTags: string[]; // 从 path + snippet 推导的概念标签
  promotedAt?: string;   // 晋升时间（已晋升条目保留此字段用于去重）
};
```

**关键点**：`recallCount`、`dailyCount`、`groundedCount` 是三种不同的信号来源，累加后作为 Deep Phase 评分中的 frequency 信号。

### phase-signals.json 结构

```typescript
type ShortTermPhaseSignalStore = {
  version: 1;
  updatedAt: string;
  entries: Record<string, ShortTermPhaseSignalEntry>;
};

type ShortTermPhaseSignalEntry = {
  key: string;
  lightHits: number;   // 被 Light Phase 处理的次数
  remHits: number;     // 被 REM Phase 处理的次数
  lastLightAt?: string;
  lastRemAt?: string;
};
```

Light 和 REM 阶段命中条目时递增对应计数，Deep Phase 评分时读取这些计数作为额外加成。

---

## 二、信号如何进入 `.dreams/`

### memory_search 触发记录

每次 `memory_search` 被调用时（[[OpenClaw MEMORY.md Lifecycle#二、读取路径]]），搜索结果会被异步记录到短期召回存储。代码在 `extensions/memory-core/src/tools.ts:106-122`：

```typescript
function queueShortTermRecallTracking(params) {
  const trackingResults = resolveRecallTrackingResults(params.rawResults, params.surfacedResults);
  void recordShortTermRecalls({
    workspaceDir: params.workspaceDir,
    query: params.query,
    results: trackingResults,
    timezone: params.timezone,
  }).catch(() => {
    // Recall tracking is best-effort and must never block memory recall.
  });
}
```

**关键特征**：记录是 best-effort、异步、不阻塞搜索返回。搜索结果中只有 `source === "memory"` 且路径匹配短期记忆模式的条目才会被记录。

### recordShortTermRecalls 的更新逻辑

代码在 `short-term-promotion.ts:910-1007`。对于每个匹配的搜索结果：

1. **去重**：先尝试用 `claimHash`（snippet 内容哈希）匹配已有条目，找到则更新，否则新建
2. **累加信号**：`recallCount++`、`totalScore += score`、`maxScore = max(existing, score)`
3. **去重查询**：同一天同一查询不重复计数（`dedupeByQueryPerDay`）
4. **保留窗口**：`queryHashes` 最多 32 个，`recallDays` 最多 16 天
5. **概念标签**：从 `path + snippet` 自动推导（`deriveConceptTags`）

---

## 三、Light Phase（收集 + 暂存）

**代码**：`extensions/memory-core/src/dreaming-phases.ts:1553-1650`

### 做什么

Light Phase 从三个来源收集短期信号，写入短期召回存储。

```
memory/*.md（每日笔记）
    ↓ 提取片段
memory/.dreams/session-corpus/（会话 transcript）
    ↓ 提取片段
已有的 short-term-recall.json
    ↓ 过滤近期条目
合并、去重、限制数量
    ↓
更新 short-term-recall.json（dailyCount 递增）
更新 phase-signals.json（lightHits 递增）
```

### 具体操作

1. **收集每日笔记信号**：读取近期 `memory/*.md` 文件，提取片段，调用 `recordShortTermRecalls` 并设置 `signalType: "daily"`
2. **收集会话 transcript 信号**：从 `session-corpus/` 中读取近期 transcript 分片，同样记录
3. **过滤**：只保留在 lookback 窗口内的条目（默认配置）
4. **去重**：使用 `dedupeSimilarity` 过滤相似度过高的条目
5. **限制数量**：不超过 `limit` 配置值
6. **更新 phase-signals.json**：每个被处理的条目 `lightHits++`
7. **生成 Dream Diary 叙述**：如果配置了 subagent model，生成 Light 阶段的日记条目

### 不做什么

- **不写入 MEMORY.md**
- **不修改已有条目的 recallCount**（只递增 `dailyCount`）
- **不做评分或晋升**

---

## 四、REM Phase（反思模式 + 主题）

**代码**：`extensions/memory-core/src/dreaming-phases.ts:1652-1749`

### 做什么

REM Phase 分析概念标签的分布模式，识别跨条目的主题和重复出现的概念。

```
short-term-recall.json 中的近期条目
    ↓ 提取 conceptTags
    ↓ 统计每个 tag 在多少条目中出现
    ↓ 计算 patternStrength = count / totalEntries * 2
    ↓ 筛选 strength >= minPatternStrength 的模式
识别出重复主题
    ↓
选择"候选真相"（confidence >= 0.45）
    ↓
更新 phase-signals.json（remHits 递增）
```

### "反思主题"的具体含义

代码分析每个条目的 `conceptTags` 数组。如果一个 tag 在多个不同条目中出现，说明这个概念是跨条目的重复主题。例如多条记忆都带有 "typescript" 标签，REM Phase 会将其识别为一个主题模式。

### 候选真相选择

`selectRemCandidateTruths`（代码在 `dreaming-phases.ts:1450-1470`）从 REM 反思结果中选择值得关注的条目，筛选条件是 `confidence >= 0.45`，评分基于 recall strength、average score、consolidation 和 conceptual tags。

### 不做什么

- **不写入 MEMORY.md**
- **不修改 recallCount 或 dailyCount**
- 只更新 `phase-signals.json` 中的 `remHits`，为 Deep Phase 评分提供加成信号

---

## 五、Deep Phase（评分 + 晋升）

Deep Phase 的晋升和去重机制已在 ([[OpenClaw MEMORY.md Lifecycle#2. Dreaming Deep Phase（自动写入）]]) 中详细分析。这里补充评分算法。

### 六个评分信号

**代码**：`short-term-promotion.ts:56-63`（权重常量）和 `short-term-promotion.ts:1201-1340`（评分计算）

| Signal | Weight | 计算方式 |
|--------|--------|----------|
| Frequency | 0.24 | `log1p(signalCount) / log1p(10)`，signalCount = max(recallCount, dailyCount, groundedCount) |
| Relevance | 0.30 | `totalScore / signalCount`（平均召回质量） |
| Diversity | 0.15 | `contextDiversity / 5`，contextDiversity = max(uniqueQueries, recallDays.length) |
| Recency | 0.15 | 指数衰减，半衰期 14 天 |
| Consolidation | 0.10 | 基于 recallDays 数量，多天重复出现的条目得分更高 |
| Conceptual | 0.06 | `conceptTags.length / 6`，概念标签密度 |

### Phase Reinforcement（阶段加成）

**代码**：`short-term-promotion.ts:591-614`

Light 和 REM 阶段在 `phase-signals.json` 中记录的命中次数会为 Deep Phase 评分提供额外加成：

```
lightBoost = min(lightHits / 3, 1) × 0.06 × recencyDecay(lightHits)
remBoost = min(remHits / 2, 1) × 0.09 × recencyDecay(remHits)
totalBoost = lightBoost + remBoost
```

半衰期 14 天（`PHASE_SIGNAL_HALF_LIFE_DAYS`）。这意味着即使一个条目没有足够的 organic recall，通过 Dreaming 自身的重复访问也可以积累足够分数达到晋升门槛。

### 评分总公式

```
score = (frequency × 0.24) + (relevance × 0.30) + (diversity × 0.15)
      + (recency × 0.15) + (consolidation × 0.10) + (conceptual × 0.06)
      + lightBoost + remBoost
```

然后通过阈值门控：`score >= 0.75 AND signalCount >= 3 AND uniqueQueries >= 2`。

---

## 六、Grounded Backfill（历史回溯）

Grounded Backfill 是 Dreaming 的一个补充机制，用于从**历史** `memory/*.md` 文件中重新提取可能被遗漏的长期记忆候选。

### 与普通 Dreaming 的区别

| 特性 | 普通 Dreaming | Grounded Backfill |
|------|---------------|-------------------|
| 数据源 | `.dreams/` 短期召回 + 近期每日笔记 | 历史 `memory/*.md` 文件 |
| 触发 | 自动（cron/heartbeat） | 手动（`openclaw memory rem-backfill`） |
| 候选存储 | 现有 short-term-recall.json | 同一个 short-term-recall.json（`groundedCount` 递增） |
| 晋升路径 | Deep Phase 直接晋升 | 先 staging，再由普通 Deep Phase 晋升 |

### 工作流程

1. 读取指定的历史 `memory/*.md` 文件
2. 提取片段，调用 `recordShortTermRecalls` 设置 `signalType: "grounded"`
3. 候选条目被写入 `short-term-recall.json`，`groundedCount` 递增
4. **不直接晋升**——候选进入短期召回存储后，等待下一次普通 Dreaming sweep 的 Deep Phase 评估
5. 如果不需要，可以用 `--rollback` 清除 grounded-only 条目

---

## 七、数据流：`.dreams/` 在一个 Sweep 中的变化

以一次完整的 cron 触发 sweep 为例：

```
Sweep 开始
│
├── Light Phase
│   ├── 读取 memory/*.md（近期每日笔记）
│   ├── 读取 session-corpus/（近期 transcript）
│   ├── 更新 short-term-recall.json：
│   │   - 新条目：创建（dailyCount=1）
│   │   - 已有条目：dailyCount++
│   ├── 更新 phase-signals.json：
│   │   - 每个处理过的条目 lightHits++
│   └── 生成 Dream Diary 到 DREAMS.md
│
├── REM Phase
│   ├── 读取 short-term-recall.json（近期条目）
│   ├── 分析 conceptTags 分布 → 识别主题模式
│   ├── 更新 phase-signals.json：
│   │   - 每个处理过的条目 remHits++
│   └── 生成 Dream Diary 到 DREAMS.md
│
├── Deep Phase
│   ├── 读取 short-term-recall.json（全部条目）
│   ├── 读取 phase-signals.json（阶段加成）
│   ├── 评分：6 个基础信号 + phase reinforcement
│   ├── 阈值过滤 + 重水化验证
│   ├── 写入 MEMORY.md（追加晋升 section）
│   ├── 更新 short-term-recall.json：
│   │   - 已晋升条目设置 promotedAt
│   └── 生成 Dream Diary 到 DREAMS.md
│
└── Sweep 结束
```

---

## 七点五、存储增长：无自动清理机制

`short-term-recall.json` 和 `phase-signals.json` **没有自动清理机制**。代码中不存在任何 pruning、eviction、TTL 或 size cap 逻辑。

### 已晋升条目的处理

条目被 Deep Phase 晋升后会设置 `promotedAt` 字段，但**不会被删除**。它们只是被排除在后续晋升候选之外：

```typescript
// short-term-promotion.ts:1241-1243 — 读取时过滤已晋升条目
if (!includePromoted && entry.promotedAt) {
  continue;
}

// short-term-promotion.ts:1579-1581 — 晋升候选筛选
if (candidate.promotedAt) {
  return false;
}
```

### 信号字段的有限约束

虽然存储本身无上限，但部分信号字段有窗口约束（`short-term-promotion.ts:136-142`）：

- `queryHashes`：最多 32 个（超出时 FIFO 裁剪）
- `recallDays`：最多 16 天（超出时 FIFO 裁剪）

这两个约束限制的是单条目内的数组大小，不是存储的总条目数。

### 影响评估

在正常使用中，短期召回条目的增长速度受限于 `memory_search` 的调用频率和 Dreaming sweep 的写入频率。每个唯一的内容片段对应一个条目（通过 `claimHash` 去重），已晋升条目虽不清理但不再产生新信号。长期运行时 `short-term-recall.json` 会持续增大，目前只能通过手动删除 `.dreams/` 目录重置。

---

## 八、关键代码索引

| 功能 | 文件 | 行号 |
|------|------|------|
| Sweep 入口 | `extensions/memory-core/src/dreaming.ts` | 549-565 |
| Phase 执行顺序 | `extensions/memory-core/src/dreaming-phases.ts` | 1751-1794 |
| Light Phase | `extensions/memory-core/src/dreaming-phases.ts` | 1553-1650 |
| REM Phase | `extensions/memory-core/src/dreaming-phases.ts` | 1652-1749 |
| REM 主题分析 | `extensions/memory-core/src/dreaming-phases.ts` | 1516-1551 |
| REM 候选选择 | `extensions/memory-core/src/dreaming-phases.ts` | 1450-1470 |
| Deep Phase 评分 | `extensions/memory-core/src/short-term-promotion.ts` | 1201-1340 |
| Phase Reinforcement | `extensions/memory-core/src/short-term-promotion.ts` | 591-614 |
| 评分权重常量 | `extensions/memory-core/src/short-term-promotion.ts` | 56-63 |
| 晋升写入 | `extensions/memory-core/src/short-term-promotion.ts` | 1551-1686 |
| Recall 记录入口 | `extensions/memory-core/src/tools.ts` | 106-122 |
| recordShortTermRecalls | `extensions/memory-core/src/short-term-promotion.ts` | 910-1007 |
| 存储读写 | `extensions/memory-core/src/short-term-promotion.ts` | 441 (read), 852 (write) |
| RecallEntry 类型 | `extensions/memory-core/src/short-term-promotion.ts` | 65-84 |
| PhaseSignal 类型 | `extensions/memory-core/src/short-term-promotion.ts` | 92-104 |
| .dreams/ 路径常量 | `extensions/memory-core/src/short-term-promotion.ts` | 30-32 |
| Grounded Backfill CLI | `extensions/memory-core/src/cli.runtime.ts` | — |
| Dream Diary 生成 | `extensions/memory-core/src/dreaming-narrative.ts` | — |
| 已晋升条目过滤 | `extensions/memory-core/src/short-term-promotion.ts` | 1241-1243, 1579-1581 |
| 信号字段窗口约束 | `extensions/memory-core/src/short-term-promotion.ts` | 136-142 |
