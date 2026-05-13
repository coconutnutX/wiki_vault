---
type: meta
title: "Wiki Log"
updated: 2026-05-13
---

# Wiki Log

Chronological record of all wiki operations. Append-only — never edit past entries. New entries go at the TOP.

## 2026-05-13

- **[save]** oG-Memory Provenance Design Analysis + Schema-Driven Provenance Design → wiki/repos/ (溯源方案设计：初版三方案对比 + Schema-driven 深入设计含 Provenance ID/SourceRef/通用组件/四阶段规划)
- **[update]** repos/_index.md, index.md — 补充两篇溯源设计文档索引
- **[move]** ogmemory_provenance_design.md → wiki/repos/oG-Memory Provenance Design Analysis.md (初版精简保存)
- **[save]** oG-Memory Session Archive vs Extracted Memory → wiki/repos/ (Session Archive 与抽取记忆串行执行、数据独立、仅 extraction_summary 去重)
- **[update]** repos/_index.md, index.md — 补充 Session Archive vs Extracted Memory 索引
- **[save]** oG-Memory Extraction and Storage Analysis + Example → wiki/repos/ (抽取与存储模块架构分析+实例详解，代码位置已验证，双向链接)
- **[update]** repos/_index.md, index.md — 补充两篇 oG-Memory 文档索引
- **[save]** Memory Provenance in Agentic Systems → wiki/concepts/ (记忆溯源研究综述：provenance/source attribution 术语、5 篇核心论文、3 篇综述、2 个实践项目、实现建议)
- **[update]** concepts/_index.md, index.md — 补充 Memory Provenance 索引
- **[save]** OpenClaw Dreaming Mechanism Visualized → wiki/repos/ (Mermaid 可视化数据流：总览图 + 阶段零信号收集时序图 + Light/REM/Deep 读写流程图 + 文件读写总结表)
- **[update]** repos/_index.md, index.md — 补充 Visualized 文档索引

## 2026-05-12

- **[update]** OpenClaw Dreaming Mechanism → 补充存储增长分析：无自动清理机制、已晋升条目永久保留、信号字段窗口约束
- **[update]** OpenClaw MEMORY.md Lifecycle, OpenClaw Memory for Developers → 修正索引范围描述：SQLite 同时索引 MEMORY.md 和 memory/*.md（非仅 MEMORY.md）
- **[save]** OpenClaw Memory Research Corrections → wiki/repos/ (调研误解与纠正，含 8 个误解+方法论反思，解决 3 个待确认问题)
- **[save]** OpenClaw Memory Data Flow → wiki/repos/ (数据流时间点分析，补充 Flush 依赖说明)
- **[save]** OpenClaw Memory Architecture Analysis → wiki/repos/ (架构与代码对应，修正阈值错误 0.5→0.75, 2→3)
- **[save]** OpenClaw Memory Backend Comparison → wiki/repos/ (后端差异分析，补充 Honcho 未实现标注、Flush 归属)
- **[save]** OpenClaw Memory System Overview → wiki/repos/ (综合介绍，修正搜索返回格式描述，补充双向链接)
- **[save]** OpenClaw MEMORY.md Lifecycle → wiki/repos/ (MEMORY.md 完整生命周期：写入路径、读取路径、截断机制、去重)
- **[decision]** Wiki Linking Convention → wiki/decisions/ (内联双向链接+_index索引，不在文末集中列出)
- **[update]** repos/_index.md — 补充 6 个 OpenClaw 文档索引
- **[cleanup]** 删除 6 个 OpenClaw 文档末尾的"相关 Wiki 页面"段落（改用 _index.md 索引）
- **[verify]** 对照官方文档(6 URL)和本地代码验证 5 个 OpenClaw 分析文档的正确性

## 2026-05-11

- **[save]** Workspace Environment → wiki/entities/ (WSL2 工作环境、项目布局、conda py11 环境、wiki-vault 路径)
- **[save]** OpenClaw MEMORY.md Implementation → wiki/repos/ (源码级实现分析，含写入触发、读取路径、截断算法、关键文件行号)
- **[update]** claude-obsidian Plugin Setup Notes — 补充插件数据存储路径和跨仓库访问方案
- **[save]** claude-obsidian Plugin Setup Notes → wiki/meta/ (插件安装过程问题记录)
- **[save]** LLM Agent Memory System Design → wiki/concepts/ (从 OpenClaw 代码分析中提取的记忆系统设计模式)
- **[scaffold]** Created vault with Mode B (GitHub / Repository) structure for code architecture documentation
