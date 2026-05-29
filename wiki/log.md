---
type: meta
title: "Wiki Log"
updated: 2026-05-27
---

# Wiki Log

Chronological record of all wiki operations. Append-only — never edit past entries. New entries go at the TOP.

## 2026-05-27

- **[save]** oG-Memory @operational_tool Framework Design → wiki/modules/ (轻量装饰器框架设计：operational tools vs extraction tools 分类、函数签名自动生成schema、runtime参数过滤、dict dispatch替代if/elif、loop级工具集配置、Deep Dream扩展示例。关联 [[ogmemory-deep-dream-framework]] 方案B重新审视、[[ogmemory-tool-category-analysis]]、[[ogmemory-operation-tool-optimization]]、[[ogmemory-tool-definition-refactor-analysis]]）
- **[update]** modules/_index.md — 补充 @operational_tool Framework 条目

## 2026-05-22

- **[save]** LoCoMo Test Non-Docker Environment Adaptation → wiki/decisions/ (修改 locomo_test 代码支持日志文件读取：log_file 参数、docker_container 空值处理、OGMEM_EXTRACT_LOG_MARKER_ALT 备用标记 + OpenClaw 插件符号链接问题修复 + responses API 端点启用)
- **[update]** decisions/_index.md, index.md — 补充 LoCoMo Test Non-Docker Environment Adaptation 索引

## 2026-05-21

- **[save]** Lazy Mode Extraction Issues and Optimization Plan → wiki/modules/ (Lazy 模式问题分析：类型判断缺少指导、routing key 规范化不足、重复检测缺失、冲突分析缺失 + P0-P3 优化方案 + 测试验证策略)
- **[update]** modules/_index.md — 补充 Lazy Mode Extraction Issues 条目
- **[update]** hot.md — 更新最近上下文：Lazy 模式优化、类型判断、routing key 规范化
- **[commit]** oG-Memory `de991dc5` — refactor: replace hard-coded early termination with prompt-driven decision (删除硬编码 early termination，改用 prompt instruction 让 LLM 自主决策何时终止)

## 2026-05-20

- **[update]** Lazy Mode Empty Database Bug Fix → 补充 `_to_candidate` 统一解析重构：简化 `_execute_tool` 返回结构、Eager/Lazy 共用解析方法、数据结构一致化
- **[save]** Promptfoo Extraction Testing Framework → wiki/decisions/ (使用 promptfoo + Python script provider 为 extraction 模块搭建测试框架：provider 类型、数据传递方式、串行运行避免死锁)
- **[save]** Lazy Mode Empty Database Bug Fix → wiki/modules/ (Lazy 模式空数据库 bug 修复：prompt 补充、tool_choice 修复、格式匹配、candidates 提取)
- **[update]** decisions/_index.md, modules/_index.md — 补充新页面索引
- **[save]** oG-Memory Operations Guide → wiki/repos/ (运行指南：PostgreSQL 配置、启动/停止命令、API 端点、数据库管理、健康检查、初始化步骤)
- **[update]** index.md — 补充 oG-Memory Operations Guide 索引
- **[save]** PostgreSQL Installation Decision → wiki/decisions/ (放弃 OpenGauss Docker，改用 apt 安装 PostgreSQL 16：安装命令、数据目录、常用操作、清理记录、AGFS 状态)
- **[update]** index.md — 补充 PostgreSQL Installation Decision 索引

## 2026-05-19

- **[save]** oG-Memory ReactLoop Abstract Base Class → wiki/concepts/ (ReAct 模式抽象基类设计：Template Method 模式、4 抽象方法、2 可选 override、子类继承扩展、Factory Method 追踪、重构历程)
- **[update]** concepts/_index.md, index.md — 补充 ReactLoop 页面索引

## 2026-05-14

- **[save]** Agent Memory Provenance 追踪实现调研 → wiki/concepts/ (Provenance 实现深度调研：agentmemory/Neon-Soul/MemORAI/AEVS 四方案详解 + 生态调研 + 设计模式对比 + 实践建议)
- **[delete]** Agent Memory Provenance Implementation Patterns → 被[[Agent Memory Provenance 追踪实现调研]]完全覆盖，删除旧版
- **[update]** index.md, concepts/_index.md — 替换 Implementation Patterns 为新文档索引
- **[update]** oG-Memory Provenance Design RFC — 更新头部引用链接

## 2026-05-13

- **[delete]] oG-Memory Provenance Design Analysis + oG-Memory Schema-Driven Provenance Design — 已被 [[oG-Memory Provenance Design RFC]] 整合替代，删除旧版避免冗余；同步更新 index.md、repos/_index.md、RFC 引用说明、Agent Memory Provenance Implementation Patterns 相关链接
- **[save]** Agent Memory Provenance Implementation Patterns → wiki/concepts/ (四大实现方案深度对比：agentmemory 引用链、Neon-Soul 四层溯源、MemORAI 图谱轮次级、AEVS 字符级锚定 + 生态调研 + 设计模式对比 + 实践建议)
- **[update]** index.md, hot.md — 补充新页面索引和热缓存
- **[save]** oG-Memory Provenance Design RFC → wiki/repos/ (RFC 讨论稿：现状分析、ID 方案 prov:1:{type}:{id}:{detail}、待确认项、P0/P1/P2 实现路径)
- **[update]** oG-Memory Schema-Driven Provenance Design → 发现 pipeline 断裂（extraction 丢弃 message ID、span/archive 索引不对齐、archive 可覆写），新增 §0 前置修复；Provenance ID 改为基于 archive_id + message_id；阶段规划调整为 P0/P1/P2/P3
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
