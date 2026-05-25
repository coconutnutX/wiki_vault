---
type: meta
title: "Hot Cache"
updated: 2026-05-25T00:00:00
---

# Recent Context

## Last Updated
2026-05-25. oG-Memory extraction prompt 优化完成，根因分析发现 SUMMARY_PROMPT 是 lazy 模式真正瓶颈。

## Key Recent Facts
- **Extraction prompt 优化**: extraction.yaml 5 处修改，Cat 2 时间类+4%但总体持平77.78%
- **根因发现**: lazy 模式下 QA 只依赖 session_summary+session_archive，extraction.yaml 只影响 eager 的 context_nodes 抽取
- **SUMMARY_PROMPT 是关键**: 对 lazy 模式影响最大，当前 "BE EXHAUSTIVE" 但没有强调保留原始形容词
- **LoCoMo benchmark 数据**: Lazy 77.78%, Eager 76.54%, Lazy v2(改进prompt后) 77.78%
- Wiki vault: /mnt/c/Data/wiki-vault，workspace: /data/Workspace2

## Recent Changes
- Modified: oG-Memory/extraction/prompts/templates/extraction.yaml — 5 处 prompt 改进
- Committed: a834d7cc feat: improve extraction prompt for detail preservation and temporal precision
- Ran: lazy mode benchmark v2 (conv-30, 19 sessions, 81 QA)

## Active Threads
- SUMMARY_PROMPT 优化 (summary_generator.py) — 下一步最有提升空间的方向
- 检索策略优化 — 提升向量检索命中率
- eager 模式下 extraction.yaml 改进的实际效果尚未单独验证