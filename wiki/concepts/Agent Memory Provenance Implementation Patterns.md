---
type: concept
status: active
created: 2026-05-13
updated: 2026-05-13
tags: [memory, provenance, agentic-ai, implementation, design-pattern, agentmemory, neon-soul, memorai, aevs]
related:
  - "[[Memory Provenance in Agentic Systems]]"
  - "[[oG-Memory Provenance Design RFC]]"
---

# Agent Memory Provenance Implementation Patterns

> 从 [[Memory Provenance in Agentic Systems]] 出发的深入调研，聚焦四大实现方案的 provenance 追踪机制、设计模式对比与实践建议。完整调研文档：`/data/Workspace2/Agent Memory Provenance 追踪实现调研.md`

## 四大实现方案

### 1. agentmemory — Citation Chain 模式

**仓库**: [github.com/rohitg00/agentmemory](https://github.com/rohitg00/agentmemory)

核心机制：`sourceObservationIds` 数组 + `parentId`/`supersedes` 版本链 + 不可变 `AuditEntry` 流。

关键设计：
- **引用链**: Memory、GraphNode、GraphEdge 都携带 `sourceObservationIds`，可回溯到原始 hook 事件
- **版本链**: `parentId` + `supersedes` 链表，支持记忆演化追踪
- **级联失效**: `cascade-update` 在记忆被取代时标记所有共享源观测的图谱实体为 `stale`
- **验证**: `memory_verify` 工具走双向溯源（Memory→Observation→Session）
- **时态溯源**: GraphEdge 携带 `tvalid`/`tvalidEnd` + `context`（推理、情感、备选方案）
- **可观测性**: 委托 iii 引擎的 OpenTelemetry 集成

### 2. Neon-Soul — 四层 Provenance 链

**仓库**: [github.com/live-neon/neon-soul](https://github.com/live-neon/neon-soul)

核心机制：Memory File (line N) → Signal → Principle → Axiom 四层溯源。

关键设计：
- **SignalSource**: `{ file, line, section, context }` 提供文件:行号级定位
- **Artifact Provenance**: `self`(0.5) / `curated`(1.0) / `external`(2.0) 三类信任权重
- **反回音室**: Axiom 提升要求 ≥3 支撑原则 + ≥2 种 provenance 类型 + 至少一个外部源
- **性能取舍**: 批处理模式行号设为 0（40x LLM 调用节省），`NEON_SOUL_LINE_TRACE=1` 启用精确追踪
- **trace 命令**: 输出 `file:line` 溯源树

### 3. MemORAI — Turn-Level Graph Provenance

**论文**: arXiv 2605.01386

核心机制：在异构对话知识图谱上实现轮次级 provenance。

关键设计：
- **Entity 节点**: 存储 `turn_ids`（实体被提及的轮次列表）
- **Entity→Entity 边**: `source_turns` 字段记录产生该关系三元组的对话轮次
- **双层压缩保持 provenance**: 选择性过滤保留 `turn_id`，摘要成为 Segment 节点保持层级
- **检索**: 动态加权 PageRank + provenance-aware prompt（top turns + 引用这些 turn 的三元组）
- **消融**: 移除话题分割使 turn-level R@10 从 91.63 降至 23.86

### 4. AEVS — 字符级锚定框架

**论文**: MDPI Computers 15(3):178, 2026

核心机制：将知识三元组的每个元素锚定到源文本的字符级位置。

关键设计：
- **Anchor 五元组**: `(text, type, p_start, p_end, mu)` — 字符级半开区间
- **Faithfulness**: 三元组 faithful ⟺ π(τ, e) 对 s/r/o 三元素均非空
- **四阶段管道**: Anchor Discovery → Grounded Extraction → Restoration Verification → Coverage Supplement
- **恢复式验证**: 四级匹配（精确→模糊→schema→全文），全恢复的直接接受
- **幻觉率**: Claude 4.5 Haiku 0.23–0.44%, GPT-4o-mini 4.99–20.23%

## 设计模式对比

| 模式 | agentmemory | Neon-Soul | MemORAI | AEVS |
|------|------------|-----------|---------|------|
| 溯源粒度 | 轮次/Session | 文件行号 | 对话轮次 | 字符位置 |
| 核心结构 | `sourceObservationIds` | `SignalSource.file:line` | `source_turns` | Anchor 五元组 |
| 版本管理 | parentId+supersedes | SHA-256 增量 | 图增量更新 | N/A (管道式) |
| 失效传播 | cascade-update stale | 增量重抽取 | N/A | 恢复式验证 |
| 审计日志 | AuditEntry 不可变流 | audit.jsonl | N/A | N/A |
| 信任分级 | qualityScore | Artifact Provenance 权重 | 检索置信度 | 恢复状态向量 |

## 更广泛生态

- **PROV-AGENT** (IEEE e-Science 2025): W3C PROV 标准的 Agent 扩展，prompt→Entity, LLM调用→Activity, Agent→Agent
- **MAIF** (arXiv 2511.15097): 密码学哈希链 + DID 签名的 AI 原生容器格式，100 链路验证 179ms
- **LangFuse**: 开源 RAG 可观测性，span-by-span 追踪检索来源，但不追踪长期记忆演化
- **Mem0**: 25k+ stars，隐式 provenance（来源上下文），侧重召回率非审计级
- **OpenTelemetry GenAI**: 标准化 `gen_ai.*` 语义约定，实验阶段

## 实践建议

**最小可行 provenance**: `(source_id, span_start, span_end, timestamp, confidence)`

**四标签信任分类**: Observed(高) / Inferred(中) / Confirmed(最高) / Imported(视来源)

**架构选择路径**:
- 只需知道"来自哪个文档" → 文档级标签
- 需要回溯到"哪次对话" → 轮次级（如 agentmemory）
- 需要精确定位到"哪一行" → 行号级（如 Neon-Soul）
- 需要防幻觉验证 → 锚定框架（如 AEVS）
- 需要审计级可追溯 → 密码学链（如 MAIF）+ 审计日志

**推荐组合**: agentmemory 的引用链 + 版本管理做宏观溯源，关键抽取点用 AEVS 锚定做微观验证，审计日志保障可追溯性。
