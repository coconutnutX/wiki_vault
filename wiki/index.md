---
type: meta
title: "Wiki Index"
updated: 2026-05-13
---

# Wiki Index

Master catalog of all wiki pages. Updated on every ingest.

## Modules
<!-- One [[Page]] per module, added during ingest -->

## Components
<!-- Reusable UI or functional components -->

## Decisions
- [[Wiki Linking Convention]] — 内联双向链接 + _index.md 索引，不在文末集中列出相关页面

## Meta
- [[claude-obsidian Plugin Setup Notes]] — 插件安装、vault 搭建过程中的问题和解决方案

## Dependencies
<!-- External deps, versions, risk assessment -->

## Flows
<!-- Data flows, request paths, auth flows -->

## Repos
- [[OpenClaw Memory System Overview]] — 系统综合介绍：双插件架构、槽位系统、复刻要点
- [[OpenClaw Memory Backend Comparison]] — 后端差异分析：memory-core vs memory-lancedb、SQLite 索引层
- [[OpenClaw Memory Architecture Analysis]] — 架构与代码对应：插件系统、Dreaming、批判性分析
- [[OpenClaw Memory Data Flow]] — 数据流时间点：组合可行性、检索/存储时机、配置示例
- [[OpenClaw MEMORY.md Lifecycle]] — MEMORY.md 完整生命周期：写入/读取/截断的统一视图
- [[OpenClaw Dreaming Mechanism]] — Dreaming 三阶段数据流：recall store 变化、评分算法、phase reinforcement
- [[OpenClaw Dreaming Mechanism Visualized]] — Dreaming 机制 Mermaid 可视化：逐阶段读写数据流图
- [[OpenClaw Memory Research Corrections]] — 调研误解与纠正记录（8 个误解+方法论反思）
- [[oG-Memory Extraction and Storage Analysis]] — 抽取与存储模块架构：两阶段抽取、DB-first 存储、异步索引
- [[oG-Memory Extraction and Storage Example]] — 抽取与存储实例详解：从原始对话到持久化的完整数据流
- [[oG-Memory Session Archive vs Extracted Memory]] — Session Archive 与抽取记忆的关系：串行执行、数据独立、仅去重关联
- [[oG-Memory Provenance Design RFC]] — Provenance RFC：现状分析 + ID 方案 + 实现路径（团队讨论稿）

## Entities
- [[Workspace Environment]] — WSL2 工作环境：项目布局、Python conda 环境、wiki-vault 路径

## Concepts
- [[OpenClaw Memory for Developers]] — 面向开发者的快速导览：架构、MEMORY.md 生命周期、常见误解
- [[Memory Provenance in Agentic Systems]] — 记忆溯源：抽取/压缩后如何对应回原始文本，相关论文与实践方案
- [[Agent Memory Provenance Implementation Patterns]] — 四大实现方案深度对比：agentmemory 引用链、Neon-Soul 四层溯源、MemORAI 图谱轮次级、AEVS 字符级锚定

## Questions
<!-- Filed answers to user queries -->

## Comparisons
<!-- Side-by-side analyses -->

## Sources
<!-- One summary page per raw source -->
