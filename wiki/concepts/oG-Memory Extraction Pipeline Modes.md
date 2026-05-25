---
type: concept
title: "oG-Memory Extraction Pipeline Modes"
related:
  - "[[oG-Memory Extraction and Storage Analysis]]"
  - "[[oG-Memory Extraction Prompt Optimization Session]]"
  - "[[oG-Memory Operations Guide]]"
tags: [ogmem, extraction, lazy, eager, pipeline, architecture]
---

# oG-Memory Extraction Pipeline Modes

## Core Architecture

`_chunk_and_extract()` 在两种模式下执行流程不同：

```
after_turn() → _background_extract_write() → _chunk_and_extract()
  ├─ Step 0: SUMMARY_PROMPT → session_summary (always, both modes)
  ├─ Step 1 (eager): extraction.yaml → context_nodes (profile/entity/event/etc)
  └─ Step 1 (lazy): skip structured extraction
```

**关键点**: SUMMARY_PROMPT 生成的 session_summary 在两种模式下都执行，是独立于 extraction_mode 的第一步。

## Mode Differences

### Eager Mode
- ingest 时即完成结构化抽取
- 存储内容: `session_archive` + `session_summary` + `context_nodes`
- QA 回答同时依赖 session_summary + 预先抽取的 context_nodes
- 更大的 token 消耗 (llm_calls=183, embedding_calls=547)

### Lazy Mode
- ingest 时只生成 summary + archive，不做结构化抽取
- 存储内容: `session_archive` + `session_summary` (没有 context_nodes)
- QA 回答只依赖 session_summary + session_archive
- 查询时可能按需抽取 (但当前 LoCoMo 测试流程中未触发)
- 更小的 token 消耗 (llm_calls=78, embedding_calls=116)

## SUMMARY_PROMPT 的关键地位

文件: `/data/Workspace2/oG-Memory/extraction/summary_generator.py`

**对 lazy 模式影响最大** — 是唯一结构化信息来源。
**对 eager 模式影响较小** — 有 context_nodes 补充。

SUMMARY_PROMPT 当前指令 "BE EXHAUSTIVE" 但没有强调保留原始形容词和引用语，导致:
- "magical" → "stress relief"
- "graceful" → "supportive"
- "Finding Freedom" → sometimes preserved, sometimes lost

## Retrieval Flow (Lazy Mode)

```
query → _search_working_set()
  ├─ Round 1: search session_archive + session_summary (top_k)
  ├─ Round 2: search session_summary specifically (top_k=1)
  └─ assemble() → inject into sessionContext → LLM answer
```

## Configuration

extraction_mode 在 `/data/Workspace2/oG-Memory/config/ogmem.yaml`:
```yaml
memory:
  extraction:
    mode: lazy  # or eager
```

修改后需重启 ogmem 服务才生效。

## Benchmark Impact (LoCoMo sample 1, conv-30)

| | Lazy | Eager |
|---|---|---|
| 总体 | 77.78% | 76.54% |
| Cat 2 (时间) | 73.08% | 65.38% |
| Cat 4 (细节) | 77.27% | 81.82% |