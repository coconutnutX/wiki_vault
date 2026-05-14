---
type: concept
status: active
created: 2026-05-14
updated: 2026-05-14
tags: [memory, provenance, agentic-ai, implementation, design-pattern, agentmemory, neon-soul, memorai, aevs]
related:
  - "[[Memory Provenance in Agentic Systems]]"
  - "[[oG-Memory Provenance Design RFC]]"
---

# Agent Memory Provenance 追踪实现调研

> 基于 [[Memory Provenance in Agentic Systems]] 概念页面的深入调研，聚焦于 agentic memory 系统中 provenance 追踪的实际实现方案。

---

## 1. 问题定义

在 agentic memory 系统中，记忆经历多个处理阶段——原始观测 → 压缩/抽取 → 结构化存储 → 检索增强生成。**Provenance 追踪**的核心问题是：如何让每条结构化记忆都能回溯到产生它的原始数据？

这不仅是一个技术问题，还涉及：
- **可信度**：用户能否验证 agent 的记忆来源？
- **可调试性**：当 agent 基于错误记忆行动时，能否定位错误源头？
- **合规性**：记忆是否包含未授权信息？
- **一致性**：当源数据更新时，如何级联失效过时的衍生记忆？

---

## 2. Provenance 粒度分类

根据追踪精度的不同，实现方案可以分为四个层级：

| 粒度级别 | 描述 | 典型实现 |
|---------|------|---------|
| **文档级** (Document) | 记忆追溯到来源文档 | Mem0、LangFuse |
| **轮次级** (Turn/Session) | 记忆追溯到对话中的具体轮次 | MemORAI、agentmemory |
| **行号级** (Line/Span) | 记忆追溯到源文件的具体行号或字符位置 | Neon-Soul、AEVS |
| **句子级** (Sentence) | 生成的每个句子追溯到源句子 | TROVE |

---

## 3. 核心实现深度分析

### 3.1 agentmemory — Citation Chain 模式

**仓库**: [github.com/rohitg00/agentmemory](https://github.com/rohitg00/agentmemory)

agentmemory 通过三条互锁的链路实现 provenance 追踪：

#### 3.1.1 数据结构

**Memory 对象**的核心字段：

```typescript
interface Memory {
  id: string;
  version: number;
  parentId?: string;               // 指向上一版本
  supersedes?: string[];           // 被取代的记忆 ID 列表
  sourceObservationIds?: string[]; // 核心溯源链接
  sessionIds: string[];
  isLatest: boolean;
}
```

`sourceObservationIds` 是核心溯源机制——每条记忆都携带一组原始观测 ID，可以沿此链条回溯到最原始的 hook 事件。

**GraphNode / GraphEdge** 同样携带溯源信息：

```typescript
interface GraphNode {
  id: string;
  sourceObservationIds: string[];  // 溯源链接
  stale?: boolean;                 // 源记忆被取代时标记为过时
}

interface GraphEdge {
  id: string;
  sourceObservationIds: string[];
  version?: number;
  isLatest?: boolean;
  supersededBy?: string;
  tvalid?: string;      // 有效起始时间
  tvalidEnd?: string;   // 有效截止时间
  context?: EdgeContext; // 推理过程、情感、备选方案、置信度
}
```

#### 3.1.2 溯源链条的构建

溯源元数据在记忆生命周期的各个阶段被附加：

| 阶段 | 操作 | 溯源附加方式 |
|------|------|-------------|
| `mem::observe` | 捕获原始观测 | 分配唯一 ID + sessionId |
| `mem::compress` | 压缩观测 | 保留原 ID（就地覆盖） |
| `mem::remember` | 手动保存 | 填充 `sourceObservationIds` |
| `mem::consolidate` | 自动整合 | 收集各概念组 top 观测 ID |
| `mem::graph-extract` | 图谱抽取 | 每个节点/边记录源观测 ID |

#### 3.1.3 验证机制

`memory_verify` 工具实现双向溯源：

1. **Memory → Observation**：读取 `sourceObservationIds`，查找每个原始观测及其所属 Session
2. **Observation → Session**：加载父 Session 记录，返回完整上下文

查找策略使用 hint-based 加速：先在记忆的 `sessionIds` 范围内查找，未命中再全表扫描。

#### 3.1.4 版本与取代链

```
Memory v1 (id: mem_abc, isLatest: false)
  ← Memory v2 (id: mem_def, parentId: mem_abc, supersedes: [mem_abc], isLatest: false)
    ← Memory v3 (id: mem_ghi, parentId: mem_def, supersedes: [mem_def], isLatest: true)
```

**级联失效**：当记忆被取代时，`cascade-update` 扫描所有图谱节点和边，将共享相同 `sourceObservationIds` 的实体标记为 `stale = true`。

#### 3.1.5 审计日志

每次结构性操作都产生不可变 `AuditEntry`：

```typescript
interface AuditEntry {
  id: string;
  timestamp: string;
  operation: "observe" | "compress" | "remember" | "forget" | ...;
  functionId: string;
  targetIds: string[];
  details: Record<string, unknown>;
  qualityScore?: number;
}
```

#### 3.1.6 设计模式总结

| 模式 | 实现方式 |
|------|---------|
| 引用链 | `sourceObservationIds` 数组 |
| 版本链 | `parentId` + `supersedes` 链表 |
| 级联失效 | `stale` 标记 + cascade-update |
| 不可变审计 | `AuditEntry` 流 |
| 可观测性 | 委托给 iii 引擎的 OpenTelemetry 集成 |
| 时态溯源 | GraphEdge 的 `tvalid`/`tvalidEnd` |

---

### 3.2 Neon-Soul — 四层 Provenance 链

**仓库**: [github.com/geeks-accelerator/neon-soul](https://github.com/geeks-accelerator/neon-soul) (现已迁移至 live-neon/neon-soul)

Neon-Soul 实现了一条从 axiom 到源文件行号的四层溯源链：

#### 3.2.1 四层架构

```
Memory File (line N) → Signal → Principle → Axiom
     (源文件)          (抽取)    (蒸馏)     (收敛 N≥3)
```

#### 3.2.2 数据结构

**层 1: SignalSource**（叶级溯源记录）

```typescript
interface SignalSource {
  type: 'memory' | 'interview' | 'template' | 'session';
  file: string;        // 绝对文件路径
  section?: string;    // 文件内的标题/节
  line?: number;       // 源文件中的行号
  context: string;     // 周围文本（前100字符）
  extractedAt: Date;
}
```

`file:line` 组合提供精确的源定位。

**层 2: Signal**（信号层）

```typescript
interface Signal {
  id: string;           // "sig_<uuid>"
  text: string;
  confidence: number;
  source: SignalSource; // 携带 file/line 溯源
  stance?: 'assert' | 'deny' | 'question' | 'qualify' | 'tensioning';
  importance?: 'core' | 'supporting' | 'peripheral';
  provenance?: 'self' | 'curated' | 'external';
}
```

**层 3: PrincipleProvenance**（原则层）

```typescript
interface PrincipleProvenance {
  signals: Array<{
    id: string;
    similarity: number;
    source: SignalSource;  // 仍携带 file/line
    stance?: SignalStance;
  }>;
  merged_at: string;
  generalization?: GeneralizationProvenance; // LLM 模型、prompt 版本等
}
```

**层 4: AxiomProvenance**（公理层）

```typescript
interface AxiomProvenance {
  principles: Array<{
    id: string;
    text: string;
    n_count: number;    // 多少信号强化了此原则
  }>;
  promoted_at: string;
}
```

#### 3.2.3 行号追踪的实际运作

信号提取器逐行处理 memory 文件：

1. 将文件内容按行分割：`const lines = content.split('\n')`
2. 对第 `i` 行记录 `lineNum: i + 1`
3. 过滤结构性噪声（代码块、JSON、堆栈跟踪）
4. 候选行批量发送给 LLM 做身份信号检测
5. 检测到的信号包装为 `createSignalSource()` 记录

**关键取舍（CR-5）**：批处理模式为降低 ~40 倍 LLM 调用开销，将 `line` 设为 `0`。环境变量 `NEON_SOUL_LINE_TRACE=1` 可启用逐行精确追踪模式。

`trace` 命令输出溯源树：

```
誠 (honesty over performance)
└── "be honest about capabilities" (N=4)
    ├── memory/preferences/communication.md:23
    └── memory/diary/2024-03-15.md:45
```

#### 3.2.4 Artifact Provenance 与反回音室机制

除了行号追踪，Neon-Soul 实现了信号来源分类：

| 类型 | 含义 | 权重 |
|------|------|------|
| `self` | 个人反思 | 0.5 |
| `curated` | 采纳的内容 | 1.0 |
| `external` | 独立研究 | 2.0 |

Axiom 提升条件要求：
- 至少 3 个支撑原则
- 信号来自 ≥2 种不同 provenance 类型
- 至少一个外部来源或质疑立场

#### 3.2.5 持久化

```
.neon-soul/
  state.json         -- 合成状态、增量追踪
  signals.json       -- 所有信号及完整溯源
  principles.json    -- 原则及其 derived_from.signals
  axioms.json        -- 公理及其 derived_from.principles
  audit.jsonl        -- JSONL 审计轨迹
```

---

### 3.3 MemORAI — Turn-Level Provenance on Graph

**论文**: arXiv 2605.01386

MemORAI 在对话知识图谱上实现了轮次级 provenance，是最接近 transcript → structured memory 场景的学术方案。

#### 3.3.1 Provenance 在图中的表示

异构图 `G = (V, E)` 中有三类节点和三类边，每类都携带 provenance：

**节点类型**:
- **Entity 节点** (`e ∈ V_E`): 存储 `turn_ids` —— 实体被提及的对话轮次列表
- **Turn 节点** (`τ ∈ V_T`): 存储原文、`segment_id`、`turn_id`
- **Segment 节点** (`s ∈ V_S`): 存储段落摘要

**边类型**:
- **Entity→Entity 边**: `source_turns` 字段记录哪些对话轮次产生了该关系三元组
- **Entity↔Turn 边**: 实体到其被提及的具体轮次
- **Turn↔Segment 边**: 保留对话层级结构

#### 3.3.2 双层压缩中的 Provenance 保持

压缩分两层，但 provenance 不丢失：

**第一层 — 选择性过滤**：从每个话题段落中识别用户个人信息，保留对应消息并保留其 `turn_id` 和 `segment_id`。

**第二层 — 段落摘要**：被丢弃的消息生成摘要 `σ_i`，摘要本身成为 Segment 节点，通过 Turn-Segment 边保持层级结构。

消融实验证实其重要性：移除话题分割使 turn-level R@10 从 91.63 骤降至 23.86。

#### 3.3.3 检索中的 Provenance 利用

检索过程构建查询聚焦子图 `G_q`：

1. **多种子发现**：通过语义相似度找到 top-k segment 节点和 entity 节点
2. **一跳扩展**：从种子扩展到直接相连的 turn、entity、segment，保留完整 provenance 链
3. **动态加权 PageRank**：使用查询条件化的边权重排序

最终输出：排名靠前的 turn 节点 + 引用这些 turn 的所有三元组（通过 `source_turns` 成员关系查找），形成 provenance-aware prompt。

---

### 3.4 AEVS — 字符级锚定框架

**论文**: MDPI Computers 15(3):178, 2026

AEVS 实现了最细粒度的 provenance：将知识三元组的每个元素锚定到源文本的**字符级位置**。

#### 3.4.1 锚点五元组

```typescript
// 概念性表示
type Anchor = {
  text: string;           // 源文本中的表面字符串
  type: 'entity' | 'relation' | 'attribute';
  p_start: number;        // 字符起始位置（0-indexed）
  p_end: number;          // 字符结束位置（半开区间）
  mu: TypeMetadata;       // 类型特定的元数据
}
```

三种锚点类型：
- **Entity Anchors (A_E)**: 命名实体，作为主语或宾语
- **Relation Anchors (A_R)**: 关系短语，元数据存储 schema 映射函数 φ 的结果
- **Attribute Anchors (A_T)**: 字面值（日期、数字），作为属性三元组的宾语

**Faithfulness 定义**：三元组 τ = (s, r, o) 是 faithful 的，当且仅当 provenance 函数 π(τ, e) 对所有三个元素 e ∈ {s, r, o} 都非空。

#### 3.4.2 四阶段管道

```
Anchor Discovery → Grounded Extraction → Restoration Verification → Coverage Supplement
    (T → A)           (T,A → K_raw)        (K_raw,A,T → verified)    (补充覆盖)
```

**阶段 1 — 锚点发现**：
- LLM 识别所有有意义的 span，要求报告字符级位置
- 三阶段验证：位置边界检查 → 文本-位置一致性 → schema 映射有效性
- 不一致时自动修正（在全文中搜索锚点文本）

**阶段 2 — 锚定抽取**：
- LLM 必须从已发现的锚点中选择和组合，而非自由生成
- 每个 extracted triplet 携带 grounding annotations `(τ, γ_s, γ_r, γ_o)`

**阶段 3a — 恢复式验证**：
- "extract-then-restore" 范式：如果三元组是 faithful 的，每个元素应该能独立恢复到源锚点
- 四级匹配策略（按置信度递减）：精确匹配 → 模糊匹配 → schema 匹配 → 全文搜索
- 恢复状态向量 ρ = (ρ_s, ρ_r, ρ_o) ∈ {0,1}³，全恢复的直接接受

**阶段 3b — 覆盖感知补充**：
- 计算锚点覆盖率，对未使用的锚点触发补充抽取
- 语义去重使用 sentence embeddings（余弦相似度阈值 0.85）

#### 3.4.3 幻觉率评估

| 模型 | 幻觉率 |
|------|--------|
| Claude 4.5 Haiku | 0.23–0.44% |
| Gemini 2.5 Flash | 0.99–2.16% |
| GPT-5.1 | 4.52–14.71% |
| GPT-4o-mini | 4.99–20.23% |

97–100% 的检测到的幻觉是 "Unsupported Relation"（实体正确锚定但关系缺乏文本支撑），说明锚定机制有效防止了实体幻觉。

---

## 4. 更广泛生态中的 Provenance 方案

### 4.1 PROV-AGENT — W3C PROV 的 Agent 扩展

(arXiv 2508.02866, IEEE e-Science 2025)

将 W3C PROV 标准扩展到 agentic 工作流：
- Prompt → PROV Entity
- LLM 调用 → PROV Activity
- AI Agent → PROV Agent

使用 Model Context Protocol (MCP) 和数据可观测性，将近实时 agent 交互集成为端到端工作流 provenance。

### 4.2 MAIF — 密码学 Artifact 格式

(arXiv 2511.15097)

一种 AI 原生的容器格式，将 provenance 直接嵌入数据结构：
- **密码学哈希链** + DIDs 数字签名
- 每个动作签名，创建不可抵赖的监管链
- 性能：2,720 MB/s 流式传输，100 链路验证 179ms

### 4.3 LangFuse — RAG 可观测性

开源 LLM 可观测性平台，通过 `@observe()` 装饰器实现 RAG 管道的 span-by-span 追踪：
- 每次检索记录哪些文档/chunk 被获取
- 集成 RAGAS 评估忠实度和上下文相关性
- 适合 RAG 场景，但不追踪长期记忆演化

### 4.4 Mem0 — 隐式 Provenance

最流行的开源 agent 记忆层（25k+ stars）：
- 每条记忆携带来源上下文（哪个对话、哪个用户、哪个会话）
- 图层通过实体关系提供跨记忆的可追踪性
- 不提供审计级 provenance，侧重召回准确率

### 4.5 OpenTelemetry GenAI Semantic Conventions

正在标准化的 `gen_ai.*` 属性：
- 覆盖模型、token、prompt、工具使用、规划步骤
- 厂商中立，兼容 Datadog、Jaeger 等后端
- 实验性阶段，但代表跨平台追踪的方向

---

## 5. 设计模式对比

| 模式 | agentmemory | Neon-Soul | MemORAI | AEVS |
|------|------------|-----------|---------|------|
| **溯源粒度** | 轮次/Session | 文件行号 | 对话轮次 | 字符位置 |
| **核心数据结构** | `sourceObservationIds` 数组 | `SignalSource.file:line` | `source_turns` 字段 | Anchor 五元组 |
| **版本管理** | `parentId` + `supersedes` 链表 | 增量 SHA-256 哈希 | 图增量更新 | N/A (管道式) |
| **失效传播** | `cascade-update` 标记 stale | 增量重抽取 | N/A | 恢复式验证 |
| **审计日志** | `AuditEntry` 不可变流 | `audit.jsonl` | N/A | N/A |
| **信任分级** | qualityScore | Artifact Provenance 权重 | 检索置信度 | 恢复状态向量 |
| **可观测性** | OpenTelemetry (iii 引擎) | trace 命令 | DW-PageRank | 锚点覆盖率 |

---

## 6. 实践建议

### 6.1 最小可行 Provenance

对于大多数 agent memory 系统，最小可行方案是：

```
(source_id, span_start, span_end, timestamp, confidence)
```

例如从 transcript 中抽取记忆时，记录：来自哪个对话文件、起始位置、结束位置、时间戳、置信度。

### 6.2 四标签信任分类

OpenBrain 的分类法简单实用：

| 标签 | 含义 | 信任权重 |
|------|------|---------|
| **Observed** | 直接从环境/用户输入感知 | 高 |
| **Inferred** | 通过推理推导 | 中 |
| **Confirmed** | 通过多源或重复观测验证 | 最高 |
| **Imported** | 从外部系统引入 | 视来源而定 |

### 6.3 架构选择指南

```
你需要什么级别的 provenance？
│
├─ 只需要知道"来自哪个文档" → 文档级标签（如 Mem0）
│
├─ 需要回溯到"哪次对话" → 轮次级（如 agentmemory 的 sourceObservationIds）
│
├─ 需要精确定位到"哪一行" → 行号级（如 Neon-Soul 的 file:line）
│
├─ 需要防幻觉验证 → 锚定框架（如 AEVS 的 Anchor + Verification）
│
└─ 需要审计级可追溯 → 密码学链（如 MAIF）+ 不可变审计日志
```

### 6.4 性能与精度的取舍

| 方案 | Provenance 精度 | 性能开销 | 适用场景 |
|------|----------------|---------|---------|
| 批量处理 + 源文档标记 | 低 | 极低 | 快速原型 |
| 轮次级 provenance | 中 | 低 | 对话记忆系统 |
| 行号级追踪 | 高 | 中（逐行模式 40x 开销） | 需要审计的系统 |
| 字符级锚定 | 最高 | 高（多次 LLM 调用） | 防幻觉关键场景 |
| 密码学链 | 审计级 | 中（验证开销） | 合规要求高的场景 |

---

## 7. 结论

Agent memory provenance 追踪是一个快速演进的领域。当前没有主导标准，但几个清晰的设计模式正在浮现：

1. **引用链模式**（agentmemory）最适合需要记忆版本演化和级联失效的系统
2. **分层溯源模式**（Neon-Soul）最适合从非结构化文本中提取结构化知识的管道
3. **图谱 Provenance 模式**（MemORAI）最适合对话记忆系统，需要保持对话层级结构
4. **锚定验证模式**（AEVS）最适合对幻觉容忍度极低的场景

实际系统中，这些模式往往可以组合使用。例如，用 agentmemory 的引用链 + 版本管理做宏观溯源，在关键抽取点用 AEVS 的锚定机制做微观验证，同时用审计日志保障可追溯性。

---

## 参考来源

- agentmemory: https://github.com/rohitg00/agentmemory
- Neon-Soul: https://github.com/geeks-accelerator/neon-soul
- MemORAI: https://arxiv.org/html/2605.01386v1
- AEVS: https://www.mdpi.com/2073-431X/15/3/178
- TROVE: https://aclanthology.org/2025.acl-long.577/
- PROV-AGENT: https://arxiv.org/abs/2508.02866
- MAIF: https://arxiv.org/html/2511.15097v1
- Anatomy of Agentic Memory: https://arxiv.org/abs/2602.19320
- Mem0: https://github.com/mem0ai/mem0
- OpenTelemetry GenAI: https://opentelemetry.io/docs/specs/semconv/gen-ai/
- LangFuse: https://langfuse.com/docs/observability/overview
