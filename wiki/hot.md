---
type: meta
title: "Hot Cache"
updated: 2026-06-01
---

# Recent Context

## Last Updated
2026-06-01. Deep Dream wiki 页面更新为 v3.0-implemented，反映实际代码状态。

## Key Recent Facts
- Deep Dream 两层 schema 设计: schemas/ (存储层) + tool_specs/ (LLM 工具层)
- Bug 1 修复: check_dream_duplicate 改用 get_directory_uri 跨 registry 查找
- Bug 2 修复: _on_invalid_response 覆盖，防止 tools 被禁用
- 字段统一: overview 移除(=abstract)，importance→confidence(0.0-1.0)
- dev_0530 commit: d1829b75 (bug fixes), 4ec24d25 (refactoring)

## Recent Changes
- Updated: `wiki/modules/ogmemory-deep-dream-framework.md` — v3.0-implemented
- Updated: `wiki/index.md` — 更新描述

## Active Threads
- oGMemory deep dreaming 验证方案设计（下一步）
- SchemaField vs field dict 统一重构（待决策）