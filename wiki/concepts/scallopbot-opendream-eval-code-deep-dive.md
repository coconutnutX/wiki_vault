---
type: concept
title: "ScallopBot 与 OpenDream 评估代码详解"
created: 2026-05-26
updated: 2026-05-26
tags:
  - research
  - ai/memory
  - dreaming
  - evaluation
  - code-analysis
related:
  - "[[dreaming-consolidation-validation-survey]]"
  - "[[ogmemory-deep-dream-framework]]"
---

# ScallopBot 与 OpenDream 评估代码详解

## ScallopBot 评估代码

### 核心文件

| 文件 | 作用 |
|---|---|
| `src/eval/locomo-eval.ts` | LoCoMo 评估主入口：F1/EM 打分、QA 回答、报告生成 |
| `src/eval/metrics.ts` | 30 天模拟指标收集：P@5, Recall, MRR, 记忆健康度 |
| `src/eval/modes.ts` | 模式配置与模式特定检索实现 |
| `src/eval/eval-runner.ts` | 30 天模拟编排器、provider factory |
| `src/eval/scenarios.ts` | 30 天对话场景 + ground-truth 查询 |
| `src/memory/dream.ts` | Dream cycle orchestrator (NREM + REM) |
| `src/memory/nrem-consolidation.ts` | NREM deep sleep consolidation |

### 评估流程详解

**入口**: `runLoCoMo()` (locomo-eval.ts:751)

```
1. loadLoCoMoData()  → 读取 locomo10.json，解析 session datetime
                      → 默认选 5 个对话 (conv-26/41/42/44/48)，1049 QA items

2. 对每个 mode × 每个 conversation:
   evaluateMode()  → 创建独立 SQLite DB: /tmp/locomo-eval-{mode}-{convId}-{ts}.db

3. INGEST PHASE:
   逐 session 按时间顺序注入，在 day boundary 运行 cognitive tick:
   - enableDecay  → gardener.lightTick()   (衰减)
   - enableFusion → gardener.deepTick()     (融合/NREM)
   - enableDreams || enableReflection → gardener.sleepTick() at 3AM (dreams + reflection + gap scan)

4. QA PHASE (并行化):
   每个 QA item:
   - searchFn(question, retrievalLimit=8/5)  → 检索记忆
   - score-gate: score >= 0.25  → 保留
   - answerQA()  → LLM 带检索上下文生成答案
   - adversarial(category=5): scoreAdversarial() 检查拒绝模式
   - 其他: computeF1() + computeEM()

5. AGGREGATE:
   Overall F1 = mean(all QA F1)
   Overall EM = mean(all QA EM)
   Per-category: 按 category 字段 (1-5) 分组聚合
```

### F1 / EM 打分算法

**F1** (`computeF1`, line 253) — token-level F1:

```typescript
export function computeF1(predicted: string, label: string): number {
  const predTokens = normalizeAnswer(predicted);
  const labelTokens = normalizeAnswer(label);

  if (predTokens.length === 0 && labelTokens.length === 0) return 1.0;
  if (predTokens.length === 0 || labelTokens.length === 0) return 0.0;

  const labelSet = new Set(labelTokens);
  const common = predTokens.filter(t => labelSet.has(t)).length;

  if (common === 0) return 0.0;

  const precision = common / predTokens.length;
  const recall = common / labelTokens.length;
  return (2 * precision * recall) / (precision + recall);
}
```

**Normalization** (`normalizeAnswer`, line 241):

```typescript
export function normalizeAnswer(text: string): string[] {
  return text
    .toLowerCase()
    .replace(/[^\w\s]/g, ' ')
    .split(/\s+/)
    .filter(w => w.length > 0 && !ARTICLES.has(w))
    .map(porterStem);  // minimal Porter stemmer
}
```

**EM** (`computeEM`, line 273) — binary exact match:

```typescript
export function computeEM(predicted: string, label: string): number {
  const predTokens = normalizeAnswer(predicted);
  const labelTokens = normalizeAnswer(label);
  if (predTokens.length !== labelTokens.length) return 0;
  return predTokens.join(' ') === labelStr ? 1 : 0;
}
```

### 维度拆分

Category 映射 (line 51):

```typescript
const CATEGORY_NAMES: Record<number, string> = {
  1: 'Single-hop',
  2: 'Temporal',
  3: 'Open-domain',
  4: 'Multi-hop',
  5: 'Adversarial',
};
```

聚合代码 (lines 630-639):

```typescript
for (const cat of [1, 2, 3, 4, 5]) {
  const catResults = allResults.filter(r => r.category === cat);
  if (catResults.length > 0) {
    const name = CATEGORY_NAMES[cat];
    f1ByCategory[name] = catResults.reduce((s, r) => s + r.f1, 0) / catResults.length;
    emByCategory[name] = catResults.reduce((s, r) => s + r.em, 0) / catResults.length;
  }
}
```

### Adversarial 评分

Category 5 的 ground truth 为空/null，模型应拒绝回答：

```typescript
const UNANSWERABLE_PATTERNS = /\b(unknown|unanswer|not (mentioned|provided|in the context|available|stated|found)|cannot (be determined|determine|answer)|no (information|context|evidence|data))\b/i;

export function scoreAdversarial(predicted: string): number {
  return UNANSWERABLE_PATTERNS.test(predicted) ? 1.0 : 0.0;
}
```

### "Before vs After Dreaming" 对比

ScallopBot **没有显式的 dreaming前后对比**。对比是 **结构性的**：对比 dreaming 禁用的 mode (OpenClaw/Mem0) vs dreaming 启用的 mode (ScallopBot)。

```typescript
// OpenClaw: no lifecycle at all
OPENCLAW_MODE = {
  enableDecay: false,
  enableFusion: false,
  enableDreams: false,     // <-- no dreaming
  enableReflection: false,
}

// ScallopBot: full cognitive pipeline
SCALLOPBOT_MODE = {
  enableDecay: true,
  enableFusion: true,
  enableDreams: true,      // <-- dreaming enabled
  enableReflection: true,
  enableProactive: true,
}
```

当 `enableDreams: true` 时，`gardener.sleepTick()` 在模拟的 2-3 AM 触发：
1. **NREM consolidation** (`runDreamCycle`) — 跨类别合并相关记忆为 derived "insight" memories
2. **REM exploration** — 发现非显而易见的连接，添加 EXTENDS 关系
3. **Reflection** (`runSelfReflection`) — 生成行为洞察，更新 SOUL.md 人格指南

另外还有一个 rerun test (`locomo-scallopbot-rerun.test.ts`)，显式对比 baseline vs fixed 硬编码值：

```typescript
console.log('  BEFORE vs AFTER — ScallopBot (conv-30, 105 QA)');
console.log(`  Baseline F1: 0.25  →  New F1: ${mode.overallF1.toFixed(3)}`);
console.log(`  Baseline EM: 0.22  →  New EM: ${mode.overallEM.toFixed(3)}`);
```

Baseline 值 (F1=0.25, EM=0.22) 来自之前的生产修复前运行，hardcoded。

### Mode 特定检索实现

| Mode | 检索公式 | 特殊处理 |
|---|---|---|
| OpenClaw | `score = 0.7 × cosine + 0.3 × (1 / (1 + bm25Rank))` | minScore=0.35, candidate multiplier=4 |
| Mem0 | 纯 cosine similarity | 无 BM25 或 prominence |
| ScallopBot | 带时序 date-range detection | 去掉 relatedMemories 防噪声 |
| ScallopBot-Tuned | 同 OpenClaw 公式，但操作在 ScallopBot 的 enriched store | 无 prominence penalty, 0.7 semantic weight |

### 30 天模拟评估（metrics.ts）

不同于 LoCoMo 的 QA accuracy，30 天模拟测检索质量 + 记忆健康：

```typescript
collectDayMetrics() → {
  memoryHealth: { active, dormant, archived },
  retrieval: { precisionAt5, recall, mrr },
  lifecycle: { fusionCount, remDiscoveries, relationsCount },
  cognitive: { soulWords, gapSignals, trustScore },
  affectAccuracy: classification accuracy
}
```

`scoreQuery()` 计算：
- **Precision@5**: relevant in top-5 / 5
- **Recall**: expected substrings found / total expected
- **MRR**: 1/(rank+1) of first relevant

---

## OpenDream 评估代码

> [!warning]
> GitHub 无法直接访问（连接超时），以下信息来自搜索结果和 agent 探索的部分抓取，可能不完整。

### 项目概况

- Repo: github.com/vincx2000/OpenDreams (4 stars)
- 核心pipeline: **Trace → Reflect → Consolidate → Memory**

### 两遍域匹配评估设计

OpenDream 的两遍评估是目前最接近 "dreaming前后对比" 的设计：

```
Pass-1 (Baseline):
  1. 在 15-task fixed suite 上跑 5 trials/task/condition
     → 收集 baseline transcripts (75 条)
  2. 记录每个 task 的 pass rate

Consolidation:
  3. OpenDream 处理 pass-1 transcripts
     → Trace: adapter normalize raw history
     → Reflect: Stage-1 LLM per-session structured reflection
     → Consolidate: Stage-2 LLM cross-session pattern extraction
     → 提出 add/modify/deprecate updates
  4. 输出 versioned AGENTS.md (consolidated memory)

Pass-2 (With Consolidated Memory):
  5. 在相同 15-task suite 上重新跑 5 trials/task/condition
     → 这次 AGENTS.md 作为上下文注入
  6. 记录每个 task 的新 pass rate

Comparison:
  7. 对比 Pass-1 vs Pass-2 的 pass rate
  8. 计算 aggregate: 92% → 96% (+4.0pp)
  9. 找出 discriminating tasks: 3 个 task 各 +20pp
  10. 验证 zero regressions: 所有 15 个 task 无退化
```

**15 task suite**: 固定的 15 个编程/任务场景，覆盖不同 domain。

**关键设计要素**:

1. **域匹配 (domain-matched)**: 同一个 task suite 同一个条件，确保比较公平——不是换题目，而是同一题目换了记忆上下文

2. **5 trials per task per condition**: 每个任务在每个条件下跑 5 次，减少偶然性

3. **150 总数据点**: 15 tasks × 5 trials × 2 conditions (baseline vs consolidated)

4. **AGENTS.md 作为 consolidated memory**: OpenDream 把反思后的经验写进 AGENTS.md，这是 Claude Code/Cursor 等工具可以读取的格式——consolidation 的产物是持久化的文本文件而非向量库

5. **Zero regressions 检查**: 不仅看 aggregate 提升，还逐 task 检查是否有任何 task 在 consolidation 后变差——这比简单看平均值更严格

### 结果数据

| 指标 | 值 |
|---|---|
| Aggregate pass rate | 92% → 96% (+4.0pp) |
| 3 个 discriminating tasks | 各 +20pp |
| Zero regressions | 所有 15 task 无退化 |

---

## 对 oGMemory DeepDreaming 验证的启示

### 可借鉴的 ScallopBot 设计

1. **独立 DB per evaluation**: 每个 mode×conversation 用独立 SQLite，避免交叉污染——ogmemory 也可以用独立 session 或 reset 机制

2. **Cognitive tick 在 day boundary**: 在注入 session 间触发 dreaming，模拟真实时间流逝——这是渐进式 dreaming 验证，而非一次性注入

3. **Token-level F1 + EM 双指标**: F1 允许部分匹配（更宽容），EM 要求精确——双指标比单一 accuracy 更有区分度

4. **Adversarial 维度**: 测模型是否知道什么时候应该拒绝回答——dreaming 后的记忆可能产生幻觉，这个维度很重要

5. **Mode 特定检索公式**: 不同记忆后端用不同检索策略，公平对比需要各自最优配置

6. **Structural ablation**: 通过 mode 配置开关（enableDecay/enableFusion/enableDreams）控制 dreaming 的不同子阶段，可以做细粒度消融

### 可借鉴的 OpenDream 设计

1. **两遍同题比较**: 同 task suite × baseline → consolidate → 再跑，直接测 dreaming 的增量效应

2. **5 trials/condition**: 减少偶然性，接近 DreamerV3 的多 seed 思想

3. **Zero regressions**: 最严格的检验——consolidation 不仅应提升，还应不退化任何维度

4. **AGENTS.md 作为产物**: consolidation 输出是人类可读的文本文件，便于 inspect dreaming 产出了什么

### 两者结合的理想设计

> [!tip] 建议的评估架构
> 1. **ScallopBot 的渐进注入 + cognitive tick**: 逐 session 注入，在 day boundary 触发 dreaming
> 2. **OpenDream 的两遍同题比较**: baseline→dreaming→再测，验证增量
> 3. **ScallopBot 的维度拆分**: 按类别 (single-hop/temporal/multi-hop/adversarial/knowledge-update) 报告
> 4. **OpenDream 的 zero regressions**: 逐维度检查无退化
> 5. **DreamerV3 的统计严谨性**: 多 run、IQM、CI
> 6. **ScallopBot 的 mode 开关**: enableDreams/enableFusion/enableReflection 粒度消融

### ScallopBot 的局限

1. **Hardcoded baseline**: rerun test 的 "before" 值是硬编码的旧结果，不是同一次运行中的动态对比
2. **QA accuracy 为主**: 不测 dreaming 产出的记忆本身的质量（抽象性、一致性、新颖性）
3. **LLM judge 不用**: F1/EM 是 token-level 自动计算，没有 LLM-as-Judge（对开放性问题可能不足）

### OpenDream 的局限

1. **15 task 太小**: 统计显著性可能不足（5 trials × 15 tasks = 75 data points per condition）
2. **编程任务偏重**: task suite 主要是编程场景，不测对话/推理/时序记忆
3. **AGENTS.md 静态文件**: consolidation 产物是文本文件而非结构化记忆，不测向量检索质量
4. **无维度拆分**: 只报 aggregate + 3 discriminating tasks，不按 LoCoMo 的 5 维度细分