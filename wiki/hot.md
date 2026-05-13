---
type: meta
title: "Hot Cache"
updated: 2026-05-13T12:00:00
---

# Recent Context

## Last Updated
2026-05-13. Added Memory Provenance research survey and OpenClaw Dreaming visualization.

## Key Recent Facts
- **Memory Provenance** 是 agentic memory 中追踪记忆来源的核心问题，学术界主要术语：provenance、source attribution、grounding
- MemORAI 的 turn-level provenance 最接近 transcript → structured memory 场景，TROVE 做句子级溯源（ACL 2025）
- 实用方案：抽取时为每条记忆附加 `(source_id, span_start, span_end)` 元数据
- OpenClaw 有两种并列 Memory 插件：memory-core（文件+SQLite）和 memory-lancedb（LanceDB），通过 plugins.slots.memory 互斥
- Dreaming 和 Flush 都是 memory-core 功能，替换 memory-lancedb 后均不可用
- Wiki vault: /mnt/c/Data/wiki-vault，workspace: /data/Workspace2

## Recent Changes
- Created: [[Memory Provenance in Agentic Systems]] (wiki/concepts/) — 5 篇核心论文 + 3 篇综述 + 2 个实践项目
- Created: [[OpenClaw Dreaming Mechanism Visualized]] (wiki/repos/) — Mermaid 可视化数据流
- Updated: concepts/_index.md, wiki/index.md, wiki/log.md

## Active Threads
- oG-Memory 主项目开发中，记忆溯源是关键设计问题
- OpenClaw 文档迁移完成
