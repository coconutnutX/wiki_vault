---
type: meta
title: "Hot Cache"
updated: 2026-05-13T14:00:00
---

# Recent Context

## Last Updated
2026-05-13. Added oG-Memory extraction/storage analysis + Memory Provenance research.

## Key Recent Facts
- **oG-Memory 抽取流程**: 两阶段 (Span Identification → Span Structuring)，支持 eager/lazy mode，dual-run 机制（默认 temp + temp=0）
- **oG-Memory 存储流程**: PolicyRouter 策略路由 → ArchiveBuilder → SQLContextFS 原子写入 → Outbox 异步索引 (pgvector L0/L1/L2)
- **oG-Memory Session Archive**: 原始消息完整存 session_archives.messages JSONB，不分块；抽取记忆存 context_nodes 三层分层
- **Memory Provenance** 学术术语：provenance、source attribution、grounding。MemORAI turn-level provenance 最接近实际场景
- Wiki vault: /mnt/c/Data/wiki-vault，workspace: /data/Workspace2

## Recent Changes
- Created: [[oG-Memory Extraction and Storage Analysis]] + [[oG-Memory Extraction and Storage Example]] (wiki/repos/) — 代码位置已验证，双向链接
- Created: [[Memory Provenance in Agentic Systems]] (wiki/concepts/)
- Deleted: /data/Workspace2/extraction_storage_analysis.md + extraction_storage_example.md (已迁入 wiki)

## Active Threads
- oG-Memory 主项目开发中，记忆溯源是关键设计问题
- 记忆溯源与 oG-Memory 的 evidence_quote 字段天然关联
