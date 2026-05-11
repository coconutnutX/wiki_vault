---
type: meta
title: "Hot Cache"
updated: 2026-05-11T17:30:00
---

# Recent Context

## Last Updated
2026-05-11. Saved detailed OpenClaw MEMORY.md code analysis.

## Key Recent Facts
- OpenClaw MEMORY.md 写入有三个触发点：flush（→ daily notes）、dreaming promotion（→ MEMORY.md）、CLI 手动
- 截断算法在 bootstrap.ts 中实现，75% head + 25% tail，单文件 12,000 chars 上限
- claude-obsidian plugin skill 在非 vault CWD 下需要手动调用 `claude-obsidian:save`

## Recent Changes
- Created: [[OpenClaw MEMORY.md Implementation]] (repos) — 源码级分析，关联 [[OpenClaw Memory System]] (concepts)
- Updated: wiki/repos/_index.md, wiki/index.md, wiki/log.md

## Active Threads
- Vault 已就绪，可开始 ingest 其他仓库进行分析
- 待研究：如何让 skill 在 wiki-vault 外也能自动工作
