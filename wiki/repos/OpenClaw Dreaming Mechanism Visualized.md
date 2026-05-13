---
type: source
title: "OpenClaw Dreaming Mechanism Visualized"
status: active
tags: [openclaw, memory, dreaming, visualization, mermaid]
created: 2026-05-13
updated: 2026-05-13
---

# Dreaming 机制可视化数据流

本文是对 [[OpenClaw Dreaming Mechanism]] 的可视化补充，用 Mermaid 图逐阶段展示每个过程的读写操作。核心思路：Dreaming 就是一个 **"短期信号 → 评分筛选 → 长期存储"** 的管道，每个阶段只做一两件事，读写的文件都不同。

---

## 总体数据流

```mermaid
graph TD
    subgraph 触发源
        A["memory_search() 被调用"]
        B["cron / heartbeat 触发 Dreaming sweep"]
    end

    subgraph 存储
        ST["short-term-recall.json<br/>短期召回存储"]
        PS["phase-signals.json<br/>阶段命中计数"]
        DAILY["memory/*.md<br/>每日笔记"]
        CORPUS["session-corpus/<br/>会话 transcript"]
        MEM["MEMORY.md<br/>长期记忆"]
        DREAMS["DREAMS.md<br/>Dream Diary（人类可读日志）"]
    end

    A -->|"异步写入<br/>recallCount++<br/>totalScore累加<br/>queryHashes追加"| ST

    B --> SWEEP["Dreaming Sweep"]

    subgraph SWEEP["Dreaming Sweep（一次完整运行）"]
        direction TB
        L["Light Phase"]
        R["REM Phase"]
        D["Deep Phase"]
        L --> R --> D
    end

    DAILY -->|"读取"| L
    CORPUS -->|"读取"| L
    ST -->|"读取已有条目"| L
    L -->|"dailyCount++<br/>lightHits++"| ST
    L -->|"lightHits++"| PS
    L -->|"写入叙述"| DREAMS

    ST -->|"读取 conceptTags"| R
    R -->|"remHits++"| PS
    R -->|"写入叙述"| DREAMS

    ST -->|"读取全部条目"| D
    PS -->|"读取加成信号"| D
    D -->|"评分 ≥ 0.75?<br/>重水化验证"| D
    D -->|"追加晋升 section"| MEM
    D -->|"promotedAt = timestamp"| ST
    D -->|"写入叙述"| DREAMS
```

---

## 阶段零：信号收集（随时发生，不属于 Sweep）

```mermaid
sequenceDiagram
    participant User as 用户/Agent
    participant MS as memory_search
    participant record as recordShortTermRecalls
    participant STR as short-term-recall.json

    User->>MS: 搜索 "project layout"
    MS-->>User: 返回搜索结果（不等待写入）
    MS->>record: 异步投递搜索结果
    Note over record: best-effort，不阻塞
    record->>STR: 找到已有条目？
    alt 已有条目（claimHash 匹配）
        STR->>STR: recallCount++<br/>totalScore += score<br/>queryHashes 追加（≤32）<br/>recallDays 追加（≤16）
    else 新条目
        STR->>STR: 创建新 entry<br/>recallCount=1<br/>dailyCount=0<br/>groundedCount=0
    end
```

**读**：无（搜索本身读的是 SQLite 索引，这里只关心搜索结果的后处理）

**写**：`short-term-recall.json`
- `recallCount++`
- `totalScore` 累加本次分数
- `queryHashes` 追加查询哈希（FIFO，上限 32）
- `recallDays` 追加当天日期（FIFO，上限 16）

---

## 阶段一：Light Phase（收集 + 暂存）

```mermaid
flowchart LR
    subgraph 读取
        D1["memory/*.md<br/>（近期每日笔记）"]
        D2["session-corpus/<br/>（近期 transcript）"]
        D3["short-term-recall.json<br/>（已有条目）"]
    end

    subgraph 处理
        P["提取片段<br/>去重<br/>限制数量"]
    end

    subgraph 写入
        W1["short-term-recall.json<br/>dailyCount++"]
        W2["phase-signals.json<br/>lightHits++"]
        W3["DREAMS.md<br/>Light 叙述"]
    end

    D1 --> P
    D2 --> P
    D3 --> P
    P --> W1
    P --> W2
    P --> W3
```

**目的**：把分散在每日笔记和会话记录里的短期信号汇聚到统一的召回存储中。

**读**：
- `memory/*.md`（近期每日笔记文件）
- `session-corpus/*.txt`（近期会话 transcript）
- `short-term-recall.json`（检查已有条目，避免重复创建）

**写**：
- `short-term-recall.json`：匹配到的条目 `dailyCount++`；新发现的条目创建并设置 `dailyCount=1`
- `phase-signals.json`：每个处理过的条目 `lightHits++`
- `DREAMS.md`：追加 Light Phase 的叙述性日记

**不写**：MEMORY.md、recallCount

---

## 阶段二：REM Phase（反思主题）

```mermaid
flowchart LR
    subgraph 读取
        D1["short-term-recall.json<br/>（近期条目的 conceptTags）"]
    end

    subgraph 处理
        P["统计每个 conceptTag<br/>在多少条目中出现<br/>→ 识别重复主题"]
        P2["选择候选真相<br/>（confidence ≥ 0.45）"]
    end

    subgraph 写入
        W1["phase-signals.json<br/>remHits++"]
        W2["DREAMS.md<br/>REM 叙述"]
    end

    D1 --> P --> P2
    P2 --> W1
    P2 --> W2
```

**目的**：识别跨条目反复出现的概念主题，为 Deep Phase 评分提供额外信号。

**读**：`short-term-recall.json`（只看近期条目的 `conceptTags` 字段）

**写**：
- `phase-signals.json`：匹配到的条目 `remHits++`
- `DREAMS.md`：追加 REM Phase 的叙述性日记

**不写**：MEMORY.md、short-term-recall.json、recallCount、dailyCount

---

## 阶段三：Deep Phase（评分 + 晋升）

唯一写入 MEMORY.md 的阶段。

```mermaid
flowchart TD
    subgraph 读取
        R1["short-term-recall.json<br/>（全部条目）"]
        R2["phase-signals.json<br/>（lightHits, remHits）"]
    end

    subgraph 评分
        S1["6 基础信号<br/>frequency×0.24<br/>relevance×0.30<br/>diversity×0.15<br/>recency×0.15<br/>consolidation×0.10<br/>conceptual×0.06"]
        S2["+ Phase Reinforcement<br/>lightBoost ≤ 0.06<br/>remBoost ≤ 0.09"]
        S3["score ≥ 0.75?<br/>signalCount ≥ 3?<br/>uniqueQueries ≥ 2?"]
    end

    subgraph 验证
        V["重水化<br/>从源文件重新读取内容<br/>验证文件存在且内容匹配"]
    end

    subgraph 写入
        W1["MEMORY.md<br/>追加晋升 section<br/>（## Promoted From ...）"]
        W2["short-term-recall.json<br/>promotedAt = timestamp"]
        W3["DREAMS.md<br/>Deep 叙述"]
    end

    R1 --> S1 --> S2 --> S3
    R2 --> S2
    S3 -->|"通过阈值"| V
    V -->|"验证通过"| W1
    V -->|"验证通过"| W2
    V --> W3
    S3 -->|"未通过"| SKIP["跳过，不晋升"]
```

**读**：
- `short-term-recall.json`（全部条目的信号数据）
- `phase-signals.json`（lightHits 和 remHits 作为加成）
- `memory/*.md`（重水化：从源文件验证候选内容是否仍然存在）

**写**：
- `MEMORY.md`：追加 `## Promoted From Short-Term Memory (date)` section，每条晋升内容带 HTML 注释标记用于去重
- `short-term-recall.json`：已晋升条目设置 `promotedAt = timestamp`（不删除，只是标记）
- `DREAMS.md`：追加 Deep Phase 的叙述性日记

---

## 文件读写总结

| 文件 | 谁读 | 谁写 | 写入内容 |
|------|------|------|----------|
| `short-term-recall.json` | Light/REM/Deep 全阶段 | memory_search 后处理、Light、Deep | recallCount/dailyCount/groundedCount、promotedAt |
| `phase-signals.json` | Deep（读加成） | Light（lightHits++）、REM（remHits++） | 各阶段命中计数 |
| `MEMORY.md` | Bootstrap、搜索索引、memory_get | Deep（追加）、Agent 手动编辑 | 晋升的长期记忆 |
| `DREAMS.md` | 无人（纯人类审查） | Light/REM/Deep | 各阶段叙述性日记 |
| `memory/*.md` | Light（收集信号）、Deep（重水化验证） | Agent（正常编辑）、Flush（保存重要内容） | 每日笔记 |
| `session-corpus/` | Light（收集信号） | 会话系统 | 会话 transcript |

一句话总结：**`.dreams/` 里的两个 JSON 文件是 Dreaming 的"工作台"——所有阶段都往里面记数，只有 Deep Phase 的最后一步才会把得分最高的条目"毕业"到 MEMORY.md。** DREAMS.md 只是副产品日志，给人类看的，不影响任何自动化流程。
