---
type: concept
status: active
created: 2026-05-14
updated: 2026-05-14
tags: [agentic-memory, react, memory-management, survey, research, 2026]
source: "/data/Workspace2/agentic_memory_react_research.md"
related:
  - "[[oG-Memory BookKeeper Design RFC]]"
  - "[[oG-Memory Extraction and Storage Analysis]]"
  - "[[OpenClaw Dreaming Mechanism]]"
  - "[[OpenClaw Memory System Overview]]"
---

# Agentic Memory 领域 ReAct Agent 调研

2026 年 Agentic Memory 领域中使用 ReAct Agent 进行深度记忆管理的研究调研。涵盖 10 篇论文 + 5 个开源项目，总结技术趋势与代表性架构模式。

> 对 oG-Memory [[oG-Memory BookKeeper Design RFC|BookKeeper]] 设计有直接参考价值。

---

## 核心论文

### 1. LongSeeker: Context-ReAct (2026.05)

**arXiv:2605.05191** — 将推理、上下文管理和工具使用整合在统一循环中。

五种原子操作：**Skip, Compress, Rollback, Snippet, Delete**，动态重塑工作上下文。Compress 被证明是"表达完备的"。基于 Qwen3-30B-A3B 微调（10k 合成轨迹）。

性能：BrowseComp 61.5%（vs Tongyi DeepResearch 43.2%）。

### 2. MemReader: 主动记忆提取 (2026.04)

**arXiv:2604.07877** — ReAct-style 长期记忆主动提取。

- MemReader-0.6B：紧凑被动提取器
- MemReader-4B：GRPO 优化的主动提取器

ReAct 决策机制：评估信息价值 → 检查引用歧义 → 验证完整性 → 选择性写入 / 延迟 / 检索历史 / 丢弃闲聊。

已集成到 MemOS 并部署。

### 3. MIA: Memory Intelligence Agent (2026.04)

**arXiv:2604.04503** — Manager-Planner-Executor 三层架构。

| 组件 | 功能 | 类型 |
|------|------|------|
| Memory Manager | 存储压缩的历史搜索轨迹 | 非参数化记忆系统 |
| Planner | 为问题生成搜索计划 | 参数化记忆 agent |
| Executor | 按计划搜索和分析信息 | 执行 agent |

创新点：交替强化学习增强 Planner-Executor 协作；Planner 在推理过程中持续演化（test-time learning）。

### 4. LightThinker++: 显式自适应记忆管理 (2026.04)

**arXiv:2604.03679** — 从推理压缩进化为 Explicit Adaptive Memory Management。

引入显式记忆原语（memory primitives），专门化的轨迹合成管道训练目的性记忆调度。

性能：峰值 token 使用减少 69.9%，准确率提升 +2.42%；长时序任务 80+ 轮次保持稳定 footprint（60%-70% 减少）。

### 5. DataClaw: 透明 ReAct Reasoning Engine (2026.04)

**arXiv:2604.24067** — 集成即时通讯平台的自主数据 agent。

透明 ReAct reasoning engine + 多层记忆系统（multi-tiered memory）实现跨会话上下文保持 + 可插拔技能架构。

### 6. MemSearch-o1: 推理对齐的记忆增长 (2026.04)

**arXiv:2604.17265** — 基于 reasoning-aligned memory growth 的 agentic 搜索。

从查询的 memory seed tokens 动态增长细粒度记忆片段 → 贡献函数回溯和深度细化 → 重组全局连接的记忆路径。从流式连接转向结构化 token 级增长 + 路径推理。

### 7. ContextWeaver: 依赖结构化记忆构建 (2026.04)

**arXiv:2604.23069** — 选择性和依赖结构化的记忆框架。

依赖性构建和遍历（每一步链接到依赖的前面步骤）→ 紧凑依赖总结（根到步骤的推理路径压缩为可重用单元）→ 轻量验证层（融入执行反馈）。

### 8. HiGMem: LLM 引导的分层记忆系统 (2026.04)

**arXiv:2604.18349** — 两层 event-turn 记忆系统，LLM 使用事件摘要作为语义锚点预测哪些相关轮次值得阅读。

LoCoMo10 benchmark：五个问题类别中四个达到最佳 F1，adversarial F1 从 0.54 提升到 0.78，检索轮次减少一个数量级。

### 9. HAGE: 加权图演化记忆 (2026.05)

**arXiv:2605.09942** — RL-driven 加权图演化利用 Agentic 记忆。

加权多关系记忆框架 → 查询条件化遍历 → LLM-based 分类器识别关系意图 → 路由网络动态调制边缘嵌入维度 → 强化学习联合优化路由行为和边缘表示。

### 10. Human-Inspired Memory Architecture (2026.05)

**arXiv:2605.08538** — 生物启发记忆架构，六种认知机制：

| 机制 | 功能 |
|------|------|
| sleep-phase consolidation | 睡眠期整合 |
| interference-based forgetting | 干扰性遗忘 |
| engram maturation | 记忆印记成熟 |
| reconsolidation upon retrieval | 检索时重新整合 |
| entity knowledge graphs | 实体知识图谱 |
| hybrid multi-cue retrieval | 混合多线索检索 |

---

## 开源项目

| 项目 | 描述 | 语言 |
|------|------|------|
| [hermes-workspace](https://github.com/outsourc-e/hermes-workspace) | Hermes Agent 原生 web workspace — chat、terminal、memory、skills | JavaScript |
| [ai-agents-from-scratch](https://github.com/pguso/ai-agents-from-scratch) | 从零构建 AI agents — function calling、memory、ReAct 模式 | JavaScript |
| [AgentPro](https://github.com/traversaal-ai/AgentPro) | ReAct-style agents with tools、knowledge base、memory | Python |
| [construct-os](https://github.com/KumihoIO/construct-os) | Memory-native AI agent runtime | Rust |
| [react-ai-agent](https://github.com/epigos/react-ai-agent) | React AI Agent with Long-Term Memory | Python |

---

## 技术趋势

1. **被动 → 主动**：从被动转录、一次性写入 → ReAct-style 推理驱动决策、选择性写入、动态演化
2. **统一推理-记忆-行动**：Context-ReAct 将三者整合，记忆管理成为 agent 决策的一部分
3. **多层记忆架构**：非参数化（外部存储）+ 参数化（模型内部）；分层 event-turn 结构；图结构记忆
4. **RL 优化**：GRPO 用于记忆提取决策；RL 用于路由和边缘表示优化
5. **生物启发**：睡眠整合、遗忘曲线、记忆印记；认知科学理论指导架构设计

### 代表性架构模式

```
┌─────────────────────────────────────────────────────┐
│                 ReAct-style Memory Agent            │
├─────────────────────────────────────────────────────┤
│  ┌─────────┐    ┌─────────┐    ┌─────────────────┐  │
│  │ Reason  │───▶│  Act    │───▶│ Memory Decision │  │
│  │ (思考)  │    │ (行动)  │    │ (写入/检索/丢弃)│  │
│  └────┬────┘    └────┬────┘    └────────┬────────┘  │
│       │              │                  │           │
│       └──────────────┴──────────────────┘           │
│                      ↺ Observe                      │
├─────────────────────────────────────────────────────┤
│  Memory Operations:                                 │
│  - Skip: 跳过无关信息                               │
│  - Compress: 压缩已解决信息                         │
│  - Rollback: 回退错误分支                           │
│  - Snippet: 提取关键片段                            │
│  - Delete: 删除无用记忆                             │
└─────────────────────────────────────────────────────┘
```

---

## 未来研究方向

1. **更精细的记忆调度**：当前仍较粗糙，需要更细粒度的原语
2. **跨 session 记忆一致性**：多会话间的记忆演化与一致性保证
3. **安全性**：记忆污染攻击防御（如 ShadowMerge 研究的问题）
4. **效率与准确性权衡**：Context bottleneck、memory dilution 问题
5. **多 agent 协作记忆**：共享记忆、冲突解决（如 EquiMem）

---

## 参考文献

1. LongSeeker: arXiv:2605.05191
2. MemReader: arXiv:2604.07877
3. MIA: arXiv:2604.04503
4. LightThinker++: arXiv:2604.03679
5. DataClaw: arXiv:2604.24067
6. MemSearch-o1: arXiv:2604.17265
7. ContextWeaver: arXiv:2604.23069
8. HiGMem: arXiv:2604.18349
9. HAGE: arXiv:2605.09942
10. Human-Inspired Memory: arXiv:2605.08538
11. FSFM (Selective Forgetting): arXiv:2604.20300
12. PRISM: arXiv:2605.12260
13. Goal-Mem: arXiv:2605.12213
14. R²-Mem: arXiv:2605.13486

*调研日期: 2026-05-14*
