---
type: concept
title: "OpenClaw Memory for Developers"
status: active
tags: [openclaw, memory, guide, developer]
created: 2026-05-12
updated: 2026-05-12
---

# OpenClaw Memory for Developers

面向有 agent memory 开发背景的开发者，快速建立对 OpenClaw memory 实现的准确理解。

---

## 1. Overall Architecture

OpenClaw 的 memory 系统以插件槽位为核心。系统提供一个 `plugins.slots.memory` 槽位，通过互斥选择加载一个 memory 插件。默认选择是 `memory-core`（文件系统 + SQLite 索引），也可以换成 `memory-lancedb`（LanceDB 向量存储）。两者是完全独立的插件，存储模型、工具接口、返回格式都不同——memory-core 提供 `memory_search` / `memory_get`，返回文件片段引用；memory-lancedb 提供 `memory_recall` / `memory_store` / `memory_forget`，返回独立条目。架构图([[OpenClaw Memory Architecture Analysis#一、架构图]]。

在这个基础之上，还有一层 Enhancement Layer 可以自由叠加：Active Memory 在每次用户消息前运行一个阻塞式子 agent，主动检索相关记忆并注入 prompt；Commitments 从对话中推断未来的 follow-up 时机，通过 heartbeat 交付；Memory Wiki 提供结构化知识库。这三个模块与底层 memory 插件无关，可以与 memory-core 或 memory-lancedb 任意组合。

需要注意 memory-core 内部还有一个"搜索后端"的概念：builtin（默认 SQLite）、QMD（sidecar 进程）、Honcho（云端）。这些不是并列的 memory 插件，而是 memory-core 内部的可替换索引引擎。两层都叫 "backend" 但层级完全不同——这也是最容易混淆的地方。

### 1.1 Module Composition

各模块的组合关系有明确的允许和禁止规则。memory-core 与 memory-lancedb 互斥（通过槽位选择其一）；builtin 与 qmd 互斥（memory-core 内部的后端选择）；Dreaming 和 Flush 是 memory-core 独占功能，换成 memory-lancedb 后这两个功能都不可用。Active Memory 会自动适配当前 memory 插件的工具集——使用 memory-core 时调用 `memory_search/get`，使用 memory-lancedb 时调用 `memory_recall`。具体的组合可行性矩阵和数据流时间点([[OpenClaw Memory Data Flow#一、组合可行性]]。

### 1.2 Architecture-to-Code Mapping

| Layer | Concept | Code Location |
|-------|---------|---------------|
| Plugin core | `OpenClawPluginApi` | `src/plugins/types.ts:2527` |
| Slot selection | `plugins.slots.memory` | `src/plugins/slots.ts:12-20` |
| Memory plugins | memory-core, memory-lancedb | `extensions/*/` |
| Tool registration | `registerTool` | `src/plugins/types.ts:2556` |
| Hook registration | `api.on("hook_name", handler)` | `src/plugins/types.ts:2560` |
| Search backend | builtin / qmd | `extensions/memory-core/src/memory/` |
| Dreaming | memory-core feature | `extensions/memory-core/src/dreaming.ts` |
| Flush | memory-core feature | `extensions/memory-core/src/flush-plan.ts` |
| Enhancement | Active Memory, Commitments | `extensions/active-memory/`, `src/commitments/` |

完整的代码对应分析([[OpenClaw Memory Architecture Analysis#三、架构层次与代码对应总结]]。

---

## 2. MEMORY.md Lifecycle

MEMORY.md 是 memory-core 架构的核心文件，所有长期记忆的原文都存储在这里。SQLite 只是对它的分块索引（~400 token/块），不存储完整内容。完整分析见 [[OpenClaw MEMORY.md Lifecycle]]。

### Write Paths

MEMORY.md 有三条写入路径，性质完全不同。

**Agent 手动编辑**（正常对话中）是唯一能增删改已有内容的路径。Agent 通过通用 write 工具直接操作 MEMORY.md，可以添加新条目、编辑旧内容、删除过时记忆。官方文档的表述是"agent is expected to distill useful material from daily notes into MEMORY.md and remove stale entries"——精简和维护 MEMORY.md 是 Agent 的职责。([[OpenClaw MEMORY.md Lifecycle#1. Agent 手动编辑（正常对话）]]。

**Dreaming Deep Phase**（cron/heartbeat 触发）是自动追加路径。系统从 `.dreams/` 短期召回存储中筛选高分候选条目（minScore=0.75, recallCount≥3），先通过"重水化"从活文件验证候选内容是否仍然存在，然后追加到 MEMORY.md 末尾。每次追加时会同时写入一条 HTML 注释标记（如 `<!-- openclaw-memory-promotion:memory:2024-01-15.md:10-20:hash -->`），下次晋升前从 MEMORY.md 读回这些标记来跳过已晋升的条目——这就是去重机制。虽然逻辑上是追加，但实现是全文件覆写（读现有内容 → 追加新 section → 一次性写回），旧的 section 原封不动保留。([[OpenClaw MEMORY.md Lifecycle#2. Dreaming Deep Phase（自动写入）]]。

**Flush**（compaction 前触发）明确禁止写入 MEMORY.md。它的作用是在对话即将被压缩摘要之前，插入一个静默的 agent turn，提醒 agent 把重要内容保存到 `memory/YYYY-MM-DD.md`（每日笔记）。MEMORY.md 在此期间被视为只读——因为 flush 发生在对话中途，允许直接修改 MEMORY.md 可能导致不一致。

此外 `openclaw memory promote --apply` CLI 命令也走同一条写入路径，最终调用与 Dreaming Deep Phase 相同的 `applyShortTermPromotions` 函数。([[OpenClaw MEMORY.md Lifecycle#3. CLI 手动晋升]] 和 [[OpenClaw MEMORY.md Lifecycle#4. Flush 期间：明确禁止写入 MEMORY.md]]。

### Read Paths

MEMORY.md 有四条读取路径，用途和截断策略各不相同。

**Bootstrap 加载**是最重要的读取路径。每次新会话启动时，系统按固定顺序加载 8 个 bootstrap 文件（AGENTS.md → SOUL.md → ... → MEMORY.md），MEMORY.md 排在最后。这意味着它的预算分配最不利——如果前 7 个文件占用了大部分预算，MEMORY.md 可能被大幅截断甚至完全跳过。截断算法是机械的 75% head + 25% tail（单文件上限 12,000 chars，所有文件总计 60,000 chars），不选择"最重要的"内容。([[OpenClaw MEMORY.md Lifecycle#1. Bootstrap 加载（每次会话启动）]] 和 [[OpenClaw MEMORY.md Lifecycle#2. 注入时截断]]。

**搜索索引**通过文件监听自动维护。MEMORY.md 的变更触发分块和重索引，embedding 和 BM25 关键词索引都保留完整内容的分块，不受 prompt 预算限制。([[OpenClaw MEMORY.md Lifecycle#3. 搜索索引]]。

**memory_get** 工具直接从文件系统读取指定行范围，不经过 SQLite，返回完整文本。([[OpenClaw MEMORY.md Lifecycle#4. memory_get 精确读取]]。

**重水化**发生在 Dreaming 晋升前，从 `memory/*.md` 活文件重新读取候选条目的实际内容，验证文件是否存在、内容是否匹配。如果源文件已被删除，该候选条目会被跳过——这防止了"僵尸记忆"被写入 MEMORY.md。([[OpenClaw MEMORY.md Lifecycle#5. Dreaming 重水化]]。

---

## 3. Dreaming Mechanism

Dreaming 是 memory-core 的后台记忆巩固系统，通过三阶段（Light → REM → Deep）从短期召回中筛选高信号条目晋升到 MEMORY.md。完整分析见 [[OpenClaw Dreaming Mechanism]]。

### Signal Collection: `.dreams/` 短期召回存储

Dreaming 的输入来自 `memory/.dreams/short-term-recall.json`，这是一个持续积累的 JSON 文件。每当 `memory_search` 被调用时，搜索结果会被异步记录（best-effort，不阻塞搜索返回）：`recallCount` 递增，`totalScore` 累加本次分数，`queryHashes` 追加查询哈希用于去重，`recallDays` 追加当天日期。每个条目还有 `dailyCount`（Light Phase 从每日笔记发现的次数）和 `groundedCount`（Grounded Backfill 记录的次数），这三种信号源共同构成 Deep Phase 评分中的 frequency 信号。([[OpenClaw Dreaming Mechanism#二、短期召回存储（memory/.dreams/）]] 和 [[OpenClaw Dreaming Mechanism#二、信号如何进入 .dreams/]])。

### Light Phase: 收集 + 暂存

Light Phase 从近期 `memory/*.md` 每日笔记和 `session-corpus/` 会话 transcript 中提取片段，写入短期召回存储（`dailyCount` 递增）。它不做评分或晋升，只是把分散在各处的短期信号汇聚到统一的召回存储中。同时更新 `phase-signals.json` 中每个条目的 `lightHits` 计数。([[OpenClaw Dreaming Mechanism#三、Light Phase（收集 + 暂存）]])。

### REM Phase: 反思主题

REM Phase 分析召回条目中 `conceptTags` 的分布模式。如果一个概念标签在多个不同条目中重复出现，说明它是跨条目的主题。REM Phase 识别这些模式、选择"候选真相"（confidence >= 0.45），更新 `phase-signals.json` 的 `remHits`。它同样不写入 MEMORY.md，只为 Deep Phase 的评分提供额外的加成信号。([[OpenClaw Dreaming Mechanism#四、REM Phase（反思模式 + 主题）]])。

### Deep Phase: 评分 + 晋升

Deep Phase 是唯一写入 MEMORY.md 的阶段（写入细节见上文 Section 2 Write Paths）。它的核心是评分算法：六个基础信号（frequency 0.24、relevance 0.30、diversity 0.15、recency 0.15、consolidation 0.10、conceptual 0.06）加上 phase reinforcement 加成（Light 命中最多加 0.06，REM 命中最多加 0.09，都有 14 天半衰期衰减）。总分通过阈值门控后（score >= 0.75, signalCount >= 3, uniqueQueries >= 2），经过重水化验证，最终追加到 MEMORY.md。([[OpenClaw Dreaming Mechanism#五、Deep Phase（评分 + 晋升）]])。

### Phase Reinforcement 的设计意图

Phase reinforcement 是一个巧妙的设计：即使一个条目没有足够的 organic recall（被用户搜索命中），通过 Dreaming 自身的重复访问（Light/REM 阶段反复处理）也可以积累分数达到晋升门槛。这意味着 Dreaming 不只是被动等待用户触发 recall，而是主动从每日笔记和会话记录中挖掘值得长期保留的信息。但加成上限较低（0.06 + 0.09 = 0.15），不会让劣质条目通过重复刷分晋升。([[OpenClaw Dreaming Mechanism#五、Deep Phase（评分 + 晋升）]])。

### Grounded Backfill: 历史回溯

手动触发的 `openclaw memory rem-backfill` 可以从历史 `memory/*.md` 中重新提取可能被遗漏的长期记忆候选，写入短期召回存储（`groundedCount` 递增）。候选不直接晋升，而是等待下一次普通 Dreaming sweep 的 Deep Phase 评估。不需要时可以 `--rollback` 清除。([[OpenClaw Dreaming Mechanism#六、Grounded Backfill（历史回溯）]])。

---

## 4. Common Misconceptions

以下是调研过程中实际踩过的坑，每个都曾导致文档中出现错误描述。完整记录见 [[OpenClaw Memory Research Corrections]]。

### 4.1 memory-core 和 memory-lancedb 是并列插件，不是同一层的后端

最容易犯的错误是把 QMD、Honcho、LanceDB 都当成同一层级的"memory backend"。实际上 QMD/Honcho 是 memory-core **内部的搜索后端**（替换 SQLite 索引引擎），而 memory-lancedb 是与 memory-core **并列的 Memory 插件**（通过 `plugins.slots.memory` 槽位二选一）。配置层面也有区分：`plugins.slots.memory` 选择插件，`memory.backend` 选择 memory-core 内部的搜索后端。([[OpenClaw Memory Research Corrections#误解四：QMD 和 Honcho 与 memory-lancedb 同级]]。

### 4.2 MEMORY.md 只在 memory-core 中存在

使用 memory-lancedb 时完全没有 MEMORY.md。所有记忆存储在 LanceDB 的 `MemoryEntry` 表中（`{id, text, vector, category, importance}`），搜索返回完整条目而非文件片段引用。([[OpenClaw Memory Research Corrections#误解二：MEMORY.md 始终存在，是所有 Memory 插件的核心]]。

### 4.3 SQLite 是索引层，不是存储层

memory-core 中 SQLite 不存储记忆内容，只存 `{path, line_range, embedding}` 索引引用。实际内容始终在 MEMORY.md 和 `memory/*.md` 文件中。`memory_search` 返回的 `snippet` 字段包含文本摘要，但完整内容仍需 `memory_get` 从文件系统读取。([[OpenClaw Memory Research Corrections#误解三：SQLite 是 memory-core 的存储层]]。

### 4.4 Dreaming 的写入目标是 MEMORY.md，不只是 DREAMS.md

名称的相似性（DREAMS.md vs Dreaming）容易造成误解。实际上 Dreaming 有两个不同的输出目标：MEMORY.md 保存 Deep Phase 的长期记忆晋升结果（agent 启动时通过 bootstrap 加载），DREAMS.md 保存叙述性的 Dream Diary（仅供人类审查，不进入 agent prompt）。DREAMS.md 是副产品日志，MEMORY.md 才是核心产出。([[OpenClaw Memory Research Corrections#误解九（2026-05-12 新增）：Dreaming 只写入 DREAMS.md，不写入 MEMORY.md]]。

### 4.5 Dreaming 阈值默认值容易记错

正确的默认值是 `minScore=0.75, minRecallCount=3, minUniqueQueries=2`（代码 `short-term-promotion.ts:24-26`）。这些值比直觉预期的更保守——晋升门槛更高、需要更多次被 recall 才能晋升。([[OpenClaw Memory Research Corrections#误解五（2026-05-12 新增）：Dreaming 阈值默认值混淆]]。

### 4.6 Flush 和 Dreaming 都是 memory-core 独占功能

Flush 机制（`extensions/memory-core/src/flush-plan.ts`）和 Dreaming 都是 memory-core 的内部功能，不是框架核心提供的。替换为 memory-lancedb 后，compaction 前的自动保存和短期→长期记忆晋升都不可用。memory-lancedb 用自己的 `auto-capture` hook（agent_end 时触发）替代，但机制完全不同。([[OpenClaw Memory Research Corrections#误解七（2026-05-12 新增）：Flush 机制的归属]]。

### 4.7 MEMORY.md 的双重身份与设计局限

MEMORY.md 同时承担"完整存储"和"prompt 注入"两个职责，这造成了内在的设计张力。Dreaming 不断追加内容，但 bootstrap 注入有严格的 prompt 预算限制。文件增长到超出预算时，截断算法只机械地保留头部 75% 和尾部 25%，中间内容被丢弃——即使这些中间内容可能是重要的。SQLite 索引虽然保留了完整内容的分块，但 bootstrap 注入路径不走索引。

理论上更合理的设计是将存储和注入拆分为独立层：一个不受预算限制的完整存储文件，加一个始终控制在预算内的精选摘要文件，类似 CPU 缓存与主存的关系。OpenClaw 目前的方案是把这个精简职责交给 Agent 的判断力——让 Agent 手动维护 MEMORY.md 的紧凑度。([[OpenClaw MEMORY.md Lifecycle#五、MEMORY.md 的"双重身份"与设计张力]]。
