---
type: meta
title: "Hot Cache"
updated: 2026-05-12T12:00:00
---

# Recent Context

## Last Updated
2026-05-12. Migrated 5 OpenClaw memory analysis docs to wiki with corrections.

## Key Recent Facts
- OpenClaw 有两种并列 Memory 插件：memory-core（文件+SQLite）和 memory-lancedb（LanceDB），通过 plugins.slots.memory 互斥
- Dreaming 和 Flush 都是 memory-core 功能，替换 memory-lancedb 后均不可用
- Dreaming 阈值默认值：minScore=0.75, minRecallCount=3, minUniqueQueries=2（代码验证）
- Honcho 仅见于官方文档描述，代码中未找到实现
- memory_search 返回 snippet 文本摘要（不仅是引用），完整内容用 memory_get 从文件读取
- Wiki vault: /mnt/c/Data/wiki-vault，workspace: /data/Workspace2

## Recent Changes
- Created: [[OpenClaw Memory System Overview]], [[OpenClaw Memory Backend Comparison]], [[OpenClaw Memory Architecture Analysis]], [[OpenClaw Memory Data Flow]], [[OpenClaw Memory Research Corrections]] (all in wiki/repos/)
- Verified: 5 docs against official docs + local code, found 3 errors (threshold values, search result format, Honcho status)
- Updated: wiki/index.md, wiki/log.md

## Active Threads
- oG-Memory 主项目开发中
- OpenClaw 文档迁移完成，原始文件待清理
