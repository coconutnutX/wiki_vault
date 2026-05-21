---
type: meta
title: "Hot Cache"
updated: 2026-05-21T10:30:00
---

# Recent Context

## Last Updated
2026-05-21. 工具定义改造分析完成：不推荐完全迁移 @tool，推荐改良现有架构。

## Key Recent Facts
- **Tool Definition Refactor**: 不推荐完全迁移 @tool，因为 YAML 数据驱动架构优势明显；推荐改良现有 tool_builder.py
- **ClassVar 问题**: tool_name 硬编码可能不匹配实际调用，需改为从 schema 自动推导
- **Deep Dream @tool**: Deep Dream 可直接使用 @tool（独立模块，无历史包袱）
- **ContextWriter 能力**: CREATE/MERGE/ARCHIVE/DELETE 全部支持，Deep Dream 删除/更新可直接依赖
- Wiki vault: /mnt/c/Data/wiki-vault，workspace: /data/Workspace2

## Recent Changes
- Created: [[ogmemory-tool-definition-refactor-analysis]] (wiki/modules/) — 工具定义改造分析文档
- Updated: [[ogmemory-deep-dream-framework]] (wiki/modules/) — 补充 @tool 装饰器推荐方案
- Updated: [[Wiki Index]] — 添加工具定义分析条目

## Active Threads
- 工具定义改造分析完成，下一步：Deep Dream 实现基础版本
- 方案 C 推荐：改良 tool_builder.py，tool_name 从 schema 推导（~20 行改动）
- Deep Dream 新模块直接使用 @tool 装饰器
