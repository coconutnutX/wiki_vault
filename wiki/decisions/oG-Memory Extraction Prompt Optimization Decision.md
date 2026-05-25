---
type: decision
title: "oG-Memory Extraction Prompt Optimization Decision"
date: 2026-05-22
status: implemented
related:
  - "[[oG-Memory Extraction Prompt Optimization Session]]"
  - "[[oG-Memory Extraction Pipeline Modes]]"
tags: [ogmem, extraction, prompt, optimization]
---

# oG-Memory Extraction Prompt Optimization Decision

**Date**: 2026-05-22
**Status**: Implemented (commit a834d7cc)

## Decision

修改 `/data/Workspace2/oG-Memory/extraction/prompts/templates/extraction.yaml`，5 处改进：

1. **TEMPORAL PRECISION**: 区分可解析 vs 不可解析时间 — "a few years ago" 不伪造绝对日期，保留原文并附加估算上下文
2. **OVERVIEW DETAIL PRESERVATION**: 新增区块 — 要求保留最具体细节，附正反例 (Marley flooring vs "suitable flooring")
3. **entity 规则**: 补充保留具体形容词/描述词和精确数字
4. **event 规则**: 补充保留精确名称和情感形容词，禁止合并不同事件为单一节点
5. **Lazy mode UPDATE RULES**: 强调更新时保留旧记忆具体细节，只追加新信息

## Rationale

基于 LoCoMo benchmark (sample 1, conv-30) 错误分析：
- 40% 错误是时间日期错误 (Cat 2)
- 50% 错误是细节被泛化 (Cat 4)
- 10% 是信息完全丢失

## Outcome

总体持平 (77.78% → 77.78%)。Cat 2 时间类 +4%，Cat 4 +2%，但 Cat 1 跨会话类 -18%。

## Key Insight

**extraction.yaml 优化对 lazy 模式效果有限** — lazy 模式下 QA 完全依赖 session_summary + session_archive 的检索，而非预先抽取的 context_nodes。真正的瓶颈在 SUMMARY_PROMPT 的质量和向量检索命中率，不在 extraction prompt。

## Next Steps

- 考虑优化 SUMMARY_PROMPT (summary_generator.py) — 保留原始形容词和引用语
- 考虑优化检索策略 — 提升向量检索命中率