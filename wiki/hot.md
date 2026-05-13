---
type: meta
title: "Hot Cache"
updated: 2026-05-13T16:00:00
---

# Recent Context

## Last Updated
2026-05-13. Added Agent Memory Provenance Implementation Patterns research.

## Key Recent Facts
- **oG-Memory 抽取流程**: 两阶段 (Span Identification → Span Structuring)，支持 eager/lazy mode，dual-run 机制（默认 temp + temp=0）
- **oG-Memory 存储流程**: PolicyRouter 策略路由 → ArchiveBuilder → SQLContextFS 原子写入 → Outbox 异步索引 (pgvector L0/L1/L2)
- **oG-Memory Session Archive**: 原始消息完整存 session_archives.messages JSONB，不分块；抽取记忆存 context_nodes 三层分层
- **Memory Provenance** 学术术语：provenance、source attribution、grounding。MemORAI turn-level provenance 最接近实际场景
- **Provenance 四大实现模式**: agentmemory(citation chain) / Neon-Soul(4-layer file:line) / MemORAI(turn-level graph) / AEVS(char-level anchor)。推荐组合：引用链宏观溯源 + AEVS锚定微观验证 + 审计日志
- Wiki vault: /mnt/c/Data/wiki-vault，workspace: /data/Workspace2

## Recent Changes
- Created: [[Agent Memory Provenance Implementation Patterns]] (wiki/concepts/) — 四大实现方案深度对比
- Full research doc: `/data/Workspace2/Agent Memory Provenance 追踪实现调研.md`
- Created: [[Memory Provenance in Agentic Systems]] (wiki/concepts/)
- Deleted: /data/Workspace2/extraction_storage_analysis.md + extraction_storage_example.md (已迁入 wiki)

## Active Threads
- oG-Memory 主项目开发中，记忆溯源是关键设计问题
- 记忆溯源与 oG-Memory 的 evidence_quote 字段天然关联
