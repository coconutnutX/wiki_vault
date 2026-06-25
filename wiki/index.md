---
type: meta
title: "Wiki Index"
updated: 2026-06-11
---

# Wiki Index

Master catalog of all wiki pages. Updated on every ingest.

## Modules
- [[memoryagentbench-lme-s-run-20260624]] — 2026-06-24 新 API key 重跑 MABench LongMemEval S*：4 个新 blocker（embedder连HF hang / RLS FORCE每次重开→BYPASSRLS / 残留eval占worker / nltk无条件下载hang）+ embedder死局（无API embedding）+ 服务hang→死，含已验证修复与实用命令
- [[memoryagentbench-lme-s-test-standardization]] — MemoryAgentBench LongMemEval S* 测试流程规范：固定 user_id owner_space、chunk 级进度恢复、--contexts 选择运行、原子写入、踩坑清单
- [[ogmemory-recalllogger-bug-retrospective]] — RecallLogger search_recall 表写入失败的 bug 复盘：content.md 过滤是根因，logging formatter 是独立问题
- [[ogmemory-deep-dream-framework]] — Deep Dream 记忆整合框架：两层 schema 设计、prompt-driven tool、Bug 修复、字段统一
- [[ogmemory-tool-definition-refactor-analysis]] — 工具定义改造分析：YAML vs @tool 模式对比、推荐方案、影响评估
- [[ogmemory-tool-category-analysis]] — 工具分类分析：Extraction vs 操作工具本质区别、整合利弊、分层建议
- [[ogmemory-operation-tool-optimization]] — 操作工具优化设计：Registry 模式、自动注册、name 推导、分层控制
- [[ogmemory-react-loop-safety-check]] — ReactLoop 安全检查机制：extract_* tool call 绕过修复、_safety_check_candidates 统一、invalid response 直接结束 loop
- [[ogmemory-deep-dream-e2e-test-guide]] — DeepDream 端到端测试指南：清空DB→注入LoCoMo对话→调用Dream→查看产出SQL、日志追踪、provenance解析
- [[longmemeval-dream36-test-report]] — LongMemEval Dream_36 测试报告：owner_space 跨用户泄漏 bug 发现与修复、before/after-dream 61.1%对比、DeepDream dilution 效果分析
- [[longmemeval-test-standardization]] — LongMemEval 测试流程规范：实验命名、目录结构、数据库隔离、日志管理、服务启停、run_config 完整配置、踩坑清单
- [[longmemeval-result-comparison-xlsx-guide]] — 测试结果对比 XLSX 生成流程：evidence_diff 对比逻辑、7个踩坑修正、完整生成脚本、SQL/文件查看方法

## Components
<!-- Reusable UI or functional components -->

## Decisions
- [[Wiki Linking Convention]] — 内联双向链接 + _index.md 索引，不在文末集中列出相关页面
- [[PostgreSQL Installation Decision]] — 放弃 OpenGauss Docker，改用 apt 安装 PostgreSQL 16：安装命令、清理记录、AGFS 状态
- [[LoCoMo Test Non-Docker Environment Adaptation]] — 修改 locomo_test 代码支持日志文件读取，适配非 docker 环境
- [[oG-Memory Extraction Prompt Optimization Decision]] — extraction.yaml 5 处修改：时间精度、细节保留、event/entity 规则、lazy update rules；效果：Cat2+4%但Cat1-18%，总持平

## Meta
- [[claude-obsidian Plugin Setup Notes]] — 插件安装、vault 搭建过程中的问题和解决方案

## Dependencies
<!-- External deps, versions, risk assessment -->

## Flows
<!-- Data flows, request paths, auth flows -->

## Repos
- [[Letta Code Memory System Deep Dive]] — Letta Code 记忆系统深度调研：三层记忆体系、MemFS git-backed 存储、reflection 5-phase 流水线、检索注入策略、冲突解决
- [[Letta Code Reflection Dream Prompt Deep Dive]] — Reflection/Dream prompt 全拆解：子代理 system prompt 逐段分析、user prompt 拼装逻辑、transcript 格式、budget 截断降级、触发调度、完成后 system prompt 重编译、与 oG-Memory Deep Dream 对比
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
- [[oG-Memory BookKeeper RFC v3]] — BookKeeper RFC v3：基于团队讨论结论，ReActLoop 泛化 + Deep Dream + 演进路径
- [[oG-Memory Operations Guide]] — 运行指南：PostgreSQL 配置、启动/停止命令、API 端点、数据库管理、健康检查

## Entities
- [[Workspace Environment]] — WSL2 工作环境：项目布局、Python conda 环境、wiki-vault 路径

## Concepts
- [[OpenClaw Memory for Developers]] — 面向开发者的快速导览：架构、MEMORY.md 生命周期、常见误解
- [[Memory Provenance in Agentic Systems]] — 记忆溯源：抽取/压缩后如何对应回原始文本，相关论文与实践方案
- [[Agent Memory Provenance 追踪实现调研]] — Provenance 实现深度调研：四大方案 + 生态 + 设计模式对比 + 实践建议
- [[Agentic Memory ReAct Research]] — 2026 ReAct Agent 记忆管理调研：10 篇论文 + 5 个开源项目，技术趋势与架构模式
- [[oG-Memory ReactLoop Abstract Base Class]] — ReAct 模式抽象基类：Template Method 设计、子类继承扩展、Factory Method 追踪
- [[oG-Memory Extraction Pipeline Modes]] — lazy/eager 模式架构：SUMMARY_PROMPT 总执行、eager 额外抽取 context_nodes、lazy 查询时才抽取
- [[dreaming-consolidation-validation-survey]] — Dreaming/Consolidation 验证调研：5类验证方法、3梯队代表工作、关键Gap、LongMemEval详解、oGMemory适配脚本清单
- [[scallopbot-opendream-eval-code-deep-dive]] — ScallopBot 与 OpenDream 评估代码详解：F1/EM算法、cognitive tick流程、两遍域匹配设计、可借鉴要点与局限
- [[agent-dream-consolidation-landscape-2025]] — Agent Dream/Consolidation 研究全景 2025：Sleep-Time Compute、Sleep Rewrites Rules 规则抽象、KG 分层压缩、Self-Play+Experience Replay、Deep Dream 改进路线图
- [[letta-sleep-time-compute-dream-evaluation-analysis]] — Letta Sleep-time Compute 与 Dream 评估分析（⚠️原文修正版）：论文验证的是数学推理compute efficiency而非记忆整理效果，Leaderboard/Recovery-Bench不适合测Dream，可复用pareto对比框架但task须改为对话场景，截至目前无公开数据直接验证Dream效果
- [[dream-modifying-existing-memories-storage-analysis]] — Deep Dream 修改已有记忆可行性：ContextFS 协议已预留 upsert/archive/delete，版本管理有乐观锁但无历史版本，Dream 当前只走 add_only，4种修改路径对比（D短期→C中期→A+C长期）

## Log
- [[oG-Memory Extraction Prompt Optimization Session]] — 2026-05-22：lazy/eager benchmark 对比、prompt 优化、根因分析发现 SUMMARY_PROMPT 是 lazy 模式真正瓶颈

## Questions
<!-- Filed answers to user queries -->

## Comparisons
<!-- Side-by-side analyses -->
- [[dream-offline-optimization-evaluation-survey]] — Dream/离线优化/测评 论文清单（仅论文，标题带 arXiv 地址）：consolidation 机制 9 篇 + 离线优化 2 篇 + 测评基准 5 篇（含 LongMemEval-V2、UCSD MemoryAgentBench、Mem2ActBench 三篇 2026 新基准），所有编号已联网核实

## Sources
<!-- One summary page per raw source -->
