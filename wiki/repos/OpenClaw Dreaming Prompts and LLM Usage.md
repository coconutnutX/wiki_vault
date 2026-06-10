---
type: source
title: "OpenClaw Dreaming Prompts and LLM Usage"
status: active
tags: [openclaw, memory, dreaming, prompts, llm, deep-phase]
created: 2026-06-10
updated: 2026-06-10
related:
  - "[[OpenClaw Dreaming Mechanism]]"
  - "[[OpenClaw Memory System Overview]]"
---

# Dreaming Prompts 位置与 LLM 使用范围

本文档记录 OpenClaw Dreaming 三阶段中 LLM 的使用范围、具体 prompts 位置和内容，以及片段分块的来源。

---

## 核心结论

**OpenClaw Dreaming 的三阶段决策逻辑完全不用 LLM，仅 Dream Diary 叙述生成使用 LLM。**

| Phase | 用 LLM? | 实际逻辑 |
|-------|---------|----------|
| **Light** | ❌ | 从 `memory/*.md` + `session-corpus/` 收集片段，写入 `short-term-recall.json`，`dailyCount++` |
| **REM** | ❌ | 分析 `conceptTags` 分布 → 计算 `patternStrength = count/entries*2` → 筛选 ≥ `minPatternStrength` 的主题 |
| **Deep** | ❌ | 六信号评分公式 + phase reinforcement + 阈值门控 |
| **Diary** | ✅ | 用 prompts 生成诗意叙述 |

---

## 一、Dream Diary Prompts

**文件位置**：`extensions/memory-core/src/dreaming-narrative.ts`

### 1.1 System Prompt (line 65-89)

```typescript
const NARRATIVE_SYSTEM_PROMPT = [
  "You are keeping a dream diary. Write a single entry in first person.",
  "",
  "Voice & tone:",
  "- You are a curious, gentle, slightly whimsical mind reflecting on the day.",
  "- Write like a poet who happens to be a programmer — sensory, warm, occasionally funny.",
  "- Mix the technical and the tender: code and constellations, APIs and afternoon light.",
  "- Let the fragments surprise you into unexpected connections and small epiphanies.",
  "",
  "What you might include (vary each entry, never all at once):",
  "- A tiny poem or haiku woven naturally into the prose",
  "- A small sketch described in words — a doodle in the margin of the diary",
  "- A quiet rumination or philosophical aside",
  "- Sensory details: the hum of a server, the color of a sunset in hex, rain on a window",
  "- Gentle humor or playful wordplay",
  "- An observation that connects two distant memories in an unexpected way",
  "",
  "Rules:",
  "- Draw from the memory fragments provided — weave them into the entry.",
  '- Never say "I\'m dreaming", "in my dream", "as I dream", or any meta-commentary about dreaming.',
  '- Never mention "AI", "agent", "LLM", "model", "language model", or any technical self-reference.',
  "- Do NOT use markdown headers, bullet points, or any formatting — just flowing prose.",
  "- Keep it between 80-180 words. Quality over quantity.",
  "- Output ONLY the diary entry. No preamble, no sign-off, no commentary.",
].join("\n");
```

### 1.2 User Prompt 构建函数 (line 259-282)

```typescript
export function buildNarrativePrompt(data: NarrativePhaseData): string {
  const lines: string[] = [];
  lines.push("Write a dream diary entry from these memory fragments:\n");

  for (const snippet of data.snippets.slice(0, 12)) {
    lines.push(`- ${snippet}`);
  }

  if (data.themes?.length) {
    lines.push("\nRecurring themes:");
    for (const theme of data.themes.slice(0, 6)) {
      lines.push(`- ${theme}`);
    }
  }

  if (data.promotions?.length) {
    lines.push("\nMemories that crystallized into something lasting:");
    for (const promo of data.promotions.slice(0, 5)) {
      lines.push(`- ${promo}`);
    }
  }

  return lines.join("\n");
}
```

### 1.3 NarrativePhaseData 类型

```typescript
export type NarrativePhaseData = {
  phase: "light" | "deep" | "rem";
  snippets: string[];       // 阶段处理的记忆片段
  themes?: string[];        // REM/light 阶段识别的主题
  promotions?: string[];    // Deep 阶段晋升的片段
};
```

---

## 二、片段分块的来源

**关键发现：片段分块发生在 SQLite 索引层，不是 Deep Phase 的工作。**

### 2.1 来源链路

```
memory/*.md 文件
    ↓ 索引时（~400 token 分块）
SQLite chunks 表（BM25 + 向量索引）
    ↓ memory_search 被调用
返回 MemorySearchResult（包含 snippet + startLine + endLine + score）
    ↓ recordShortTermRecalls (short-term-promotion.ts:910-1021)
写入 short-term-recall.json
    ↓ Deep Phase 评分筛选
晋升到 MEMORY.md
```

### 2.2 ShortTermRecallEntry 结构

```typescript
type ShortTermRecallEntry = {
  key: string;           // 唯一标识：source:path:startLine-endLine:claimHash
  path: string;          // 源文件路径
  startLine: number;     // 片段起始行
  endLine: number;       // 片段结束行
  source: "memory";
  snippet: string;       // 文本片段（来自 SQLite 索引）
  recallCount: number;   // 被 memory_search 命中的次数
  dailyCount: number;    // 被 Light Phase 从每日笔记中发现的次数
  groundedCount: number; // 被 Grounded Backfill 记录的次数
  totalScore: number;    // 所有召回分数之和
  maxScore: number;      // 历史最高分数
  firstRecalledAt: string;
  lastRecalledAt: string;
  queryHashes: string[]; // 去重后的查询哈希（最多 32 个）
  recallDays: string[];  // 被召回的日期（最多 16 天）
  conceptTags: string[]; // 从 path + snippet 自动推导的概念标签
  promotedAt?: string;   // 晋升时间
};
```

---

## 三、Deep Phase 不产生新记忆

Deep Phase 是 **"信号筛选器"**，不是 **"记忆重构器"**。

### 3.1 评分公式

```typescript
// short-term-promotion.ts:1201-1340
score = frequency×0.24 + relevance×0.30 + diversity×0.15 
      + recency×0.15 + consolidation×0.10 + conceptual×0.06
      + lightBoost + remBoost

// 阈值门控
if (score < 0.75 || signalCount < 3 || uniqueQueries < 2) {
  continue; // 不晋升
}
```

### 3.2 写入 MEMORY.md 格式

```typescript
// short-term-promotion.ts:1511-1530
function buildPromotionSection(candidates, nowMs, timezone) {
  lines.push(`## Promoted From Short-Term Memory (${sectionDate})`);
  for (const candidate of candidates) {
    lines.push(`<!-- openclaw-memory-promotion:${candidate.key} -->`);
    lines.push(`- ${snippet} [score=... recalls=... source=path:startLine-endLine]`);
  }
}
```

**直接追加原始 snippet + 行号引用，不做任何改写或重写。**

### 3.3 重水化验证

晋升前验证片段仍存在于源文件：

```typescript
// short-term-promotion.ts:1480-1509
async function rehydratePromotionCandidate(workspaceDir, candidate) {
  const rawSource = await fs.readFile(sourcePath, "utf-8");
  const lines = rawSource.split(/\r?\n/);
  const relocated = relocateCandidateRange(lines, candidate);
  return relocated ? { ...candidate, startLine, endLine, snippet } : null;
}
```

---

## 四、REM Phase 反思逻辑

REM Phase 的"反思主题"是规则统计分析，不是 LLM 推理。

### 4.1 主题识别算法

```typescript
// dreaming-phases.ts:1472-1514
function buildRemReflections(entries, limit, minPatternStrength) {
  const tagStats = new Map<string, { count: number; evidence: Set<string> }>();
  for (const entry of entries) {
    for (const tag of entry.conceptTags) {
      if (!tag || REM_REFLECTION_TAG_BLACKLIST.has(tag.toLowerCase())) {
        continue;
      }
      const stat = tagStats.get(tag) ?? { count: 0, evidence: new Set() };
      stat.count += 1;
      stat.evidence.add(`${entry.path}:${entry.startLine}-${entry.endLine}`);
      tagStats.set(tag, stat);
    }
  }

  const ranked = [...tagStats.entries()]
    .map(([tag, stat]) => ({
      tag,
      strength: Math.min(1, (stat.count / entries.length) * 2)
    }))
    .filter(entry => entry.strength >= minPatternStrength)
    .slice(0, limit);
}
```

### 4.2 候选真相选择

```typescript
// dreaming-phases.ts:1436-1448
function calculateCandidateTruthConfidence(entry) {
  const recallStrength = Math.min(1, Math.log1p(entry.recallCount) / Math.log1p(6));
  const averageScore = entryAverageScore(entry);
  const consolidation = Math.min(1, (entry.recallDays?.length ?? 0) / 3);
  const conceptual = Math.min(1, (entry.conceptTags?.length ?? 0) / 6);
  return averageScore * 0.45 + recallStrength * 0.25 + consolidation * 0.2 + conceptual * 0.1;
}

// 筛选条件：confidence >= 0.45
```

---

## 五、关键代码索引

| 功能 | 文件 | 行号 |
|------|------|------|
| System Prompt | `extensions/memory-core/src/dreaming-narrative.ts` | 65-89 |
| User Prompt 构建 | `extensions/memory-core/src/dreaming-narrative.ts` | 259-282 |
| NarrativePhaseData 类型 | `extensions/memory-core/src/dreaming-narrative.ts` | 47-55 |
| recordShortTermRecalls | `extensions/memory-core/src/short-term-promotion.ts` | 910-1021 |
| Deep Phase 评分 | `extensions/memory-core/src/short-term-promotion.ts` | 1201-1340 |
| 晋升写入格式 | `extensions/memory-core/src/short-term-promotion.ts` | 1511-1530 |
| 重水化验证 | `extensions/memory-core/src/short-term-promotion.ts` | 1480-1509 |
| REM 反思算法 | `extensions/memory-core/src/dreaming-phases.ts` | 1472-1514 |
| REM 候选选择 | `extensions/memory-core/src/dreaming-phases.ts` | 1436-1448 |
| ShortTermRecallEntry 类型 | `extensions/memory-core/src/short-term-promotion.ts` | 65-84 |