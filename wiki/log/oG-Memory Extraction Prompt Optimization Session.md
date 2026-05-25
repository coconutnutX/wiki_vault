---
type: session_log
title: "oG-Memory Extraction Prompt Optimization Session"
date: 2026-05-22
participants: aaa, Claude
related:
  - "[[oG-Memory]]"
  - "[[LoCoMo Test Non-Docker Environment Adaptation]]"
  - "[[LoCoMo Benchmark]]"
tags: [ogmem, extraction, prompt, lazy-mode, eager-mode, benchmark, optimization]
---

# oG-Memory Extraction Prompt Optimization Session

**Date**: 2026-05-22
**Goal**: 优化 oG-Memory extraction prompt，提升 LoCoMo benchmark 正确率

## Session Overview

1. 分析 lazy vs eager 模式在 LoCoMo sample 1 (conv-30) 上的测试结果差异
2. 根据错误模式，提出 extraction.yaml prompt 优化建议
3. 实施修改并重新测试
4. 发现 prompt 优化效果有限，深入分析根因
5. 确认 SUMMARY_PROMPT 是 lazy 模式下真正的瓶颈

## Benchmark Results

### 原始结果 (改进前)

| | Lazy v1 | Eager |
|---|---|---|
| **总体** | 63/81 (77.78%) | 62/81 (76.54%) |
| **Cat 1** | 10/11 (90.91%) | 9/11 (81.82%) |
| **Cat 2** | 19/26 (73.08%) | 17/26 (65.38%) |
| **Cat 4** | 34/44 (77.27%) | 36/44 (81.82%) |

### 改进 prompt 后 (lazy v2)

| | Lazy v1 | Lazy v2 |
|---|---|---|
| **总体** | 63/81 (77.78%) | 63/81 (77.78%) |
| **Cat 1** | 10/11 (90.91%) | 8/11 (72.73%) |
| **Cat 2** | 19/26 (73.08%) | 20/26 (76.92%) |
| **Cat 4** | 34/44 (77.27%) | 35/44 (79.55%) |

**变化**: Cat 2 时间类 +4%, Cat 4 细节类 +2%, Cat 1 跨会话 -18%。总体持平。

### 详细对比 v1 → v2

- **改进 11 题**: Q1,15,16,31,32,40,43,47,50,62,65 (时间类大幅改善)
- **退步 11 题**: Q4,17,24,26,37,39,54,56,67,68,74 (细节类泛化加剧)
- **仍错 7 题**: Q12,14,23,45,49,53,57

## Extraction Prompt 修改

文件: `/data/Workspace2/oG-Memory/extraction/prompts/templates/extraction.yaml`

5 处修改:
1. **TEMPORAL PRECISION**: 区分可解析 vs 不可解析时间 ("a few years ago" 不伪造绝对日期)
2. **OVERVIEW DETAIL PRESERVATION**: 新增区块，要求保留最具体细节而非泛化
3. **entity 规则**: 补充要求保留具体形容词和精确数字
4. **event 规则**: 补充要求保留精确名称和情感形容词，禁止合并不同事件
5. **Lazy mode UPDATE RULES**: 强调更新时保留旧记忆的具体细节

Git commit: `a834d7cc feat: improve extraction prompt for detail preservation and temporal precision`

## 根因分析

### 为什么 prompt 优化效果有限?

**extraction.yaml 只影响 eager 模式的 context_nodes 抽取。Lazy 模式下 QA 完全依赖 session_summary + session_archive。**

数据库验证:
- lazy 模式只有 `session_archive` 和 `session_summary` 两种 category, 没有预先抽取的 `context_nodes`
- session_summary 里实际上已包含关键细节如 "Finding Freedom"、"symbolizing freedom"、"teamed up with local artist"
- 但 QA 回答时检索没命中最相关的 summary

### 退步原因

1. **检索不返回最相关的 summary** — 向量检索基于 embedding 匹配，可能返回不太相关的 session
2. **QA LLM 有随机性** — gpt-4o-mini temperature=0.1
3. **关键细节分散在不同 summary 的不同 section** — "teamed up with artist" 在 Facts 里但 "video presentation" 在另一个 summary 里

### 真正的瓶颈: SUMMARY_PROMPT

文件: `/data/Workspace2/oG-Memory/extraction/summary_generator.py`

- SUMMARY_PROMPT 在两种模式下都会执行 (eager + lazy)
- 对 lazy 模式影响最大 — 是唯一结构化信息来源
- 对 eager 模式影响较小 — 有 context_nodes 补充
- 当前 prompt 说 "BE EXHAUSTIVE" 但没有强调保留原始形容词和引用语
- 导致 "magical"、"graceful" 等词被泛化成 "stress relief"、"supportive"

### 可优化方向

1. **优化 SUMMARY_PROMPT** — 让 summary 保留更多原始形容词和具体引用
2. **优化检索策略** — 提升向量检索命中率 (top-k 数量等)
3. **Summary 结构优化** — 减少细节分散，让因果关系/对比关系更显式

## Architecture Understanding

### oG-Memory Extraction Pipeline

```
after_turn() → _background_extract_write() → _chunk_and_extract()
  ├─ Step 0: SUMMARY_PROMPT → session_summary (always runs)
  ├─ Step 1 (eager): extraction.yaml → context_nodes
  └─ Step 1 (lazy): skip, defer to query-time extraction
```

### QA Retrieval Flow (lazy mode)

```
query → _search_working_set()
  ├─ Round 1: search session_archive + session_summary (top_k)
  ├─ Round 2: search session_summary specifically (top_k=1)
  └─ assemble() → inject summary into sessionContext → LLM answer
```

### 数据存储位置

- PostgreSQL (ogmemory): context_nodes, session_archives, vector_index
- OpenClaw: ~/.openclaw/agents/main/sessions/*.jsonl
- 测试输出: /data/Workspace2/locomo_test/output/

## Open Questions

- SUMMARY_PROMPT 优化能否显著改善 lazy 模式?
- 检索策略 (top-k, 混合检索) 的优化空间有多大?
- eager 模式下的 context_nodes 抽取质量是否也受 extraction.yaml 改进影响?