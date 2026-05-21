---
type: meta
title: "Hot Cache"
updated: 2026-05-21T15:00:00
---

# Recent Context

## Last Updated
2026-05-21. Lazy 模式提取问题分析与优化方案完成。

## Key Recent Facts
- **Lazy Mode 核心问题**: 类型判断缺少指导、routing key 规范化不足、重复检测缺失、冲突分析缺失
- **类型判断边界模糊**: case/pattern/skill、entity/profile/preference、event/case 易混淆
- **Routing Key 问题**: LLM 主导生成缺少规范、系统兜底生成不稳定（依赖 abstract）、缺少已有检查
- **Upsert vs Add_only**: routing_key 决定唯一性，upsert 可合并更新，add_only 只追加不修改
- Wiki vault: /mnt/c/Data/wiki-vault，workspace: /data/Workspace2

## Recent Changes
- Created: [[Lazy Mode Extraction Issues and Optimization Plan]] (wiki/modules/) — Lazy 模式问题分析 + P0-P3 优化方案
- Updated: [[Modules Index]] — 添加 Lazy Mode Extraction Issues 条目
- Committed: `de991dc5` (oG-Memory) — refactor: replace hard-coded early termination with prompt-driven decision

## Active Threads
- Lazy 模式优化实施：先 P0/P1（prompt + schema），观察效果后决定 P2/P3
- 类型判断决策树添加到 Phase 1 prompt + Phase 2 prompt
- Routing key 规范化指导添加到 schema description
- 重复检测机制依赖 VectorIndex（需确认是否已有）
