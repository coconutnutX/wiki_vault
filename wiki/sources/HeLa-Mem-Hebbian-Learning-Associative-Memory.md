
**论文信息**
- arXiv: 2604.16839
- 会议: ACL 2026 (已接收)
- 作者: Jinchang Zhu, Jindong Li, Cheng Zhang, Jiahong Liu, Menglin Yang
- GitHub: https://github.com/ReinerBRO/HeLa-Mem

## 核心理念

借鉴认知神经科学的 Hebbian 学习原理（**"Neurons that fire together wire together"**），将对话历史建模为**动态关联图**而非静态向量库。

## 三大生物启发机制

1. **Association（关联）**：共激活的记忆自动加强连接
2. **Consolidation（整合）**：频繁访问的记忆集群固化为稳定语义知识
3. **Spreading activation（扩散激活）**：检索时通过已建立路径触发相关记忆

## 双层级架构

### Episodic Memory Graph（情景层）

- **节点** = 对话轮次（含文本、embedding、时间戳、关键词）
- **边权重** = Hebbian 连接强度，动态演化
- 边权重更新公式：
  ```
  w_ij(t+1) = (1-λ)·w_ij(t) + η·I(vi,vj∈Kt)
  ```
  - λ: decay rate
  - η: learning rate
  - I: 共激活指示函数

### Semantic Memory Store（语义层）

从情景图蒸馏出的结构化知识：
- **User Model**: 用户特征（如"喜欢户外活动"）
- **Factual Memory**: 提取的事实（带绝对时间戳）
- **Agent Knowledge**: Agent 的 persona 和行为模式

## 三模块协同

### 1. Online Encoding & Association

- 共激活的记忆加强连接，未使用的衰减
- 类似生物突触的可塑性

### 2. Reflective Memory Agent（Hebbian Distillation）

**核心功能**：
- 检测 **hub 节点**（高累计边权重）：`D(vi) = Σw_ij > δ_hub`
- 将 hub + 强连接邻居蒸馏为语义条目
- 类似睡眠整合，防止图爆炸

**Adaptive Forgetting**：
- 三条件同时满足才删除：
  1. 总边权重 < δ_prune（结构无关）
  2. 不活跃时长 > δ_age（时间休眠）
  3. 近期零访问

### 3. Dual-Path Retrieval

**Base Activation**:
```
S_base(vi) = (sim(q, ei) + α·keyword_match) · γ(vi)
```
- sim: cosine similarity
- γ(vi) = exp(-Δt/τ): 时间衰减因子

**Spreading Activation**:
```
S(vj) = S_base(vj) + β·Σ S_base(vi)·wij
```
- 高分节点通过 Hebbian 边传播激活

**Dual-Path Ranking**:
```
R_final = Top-k(S_base) ∪ Top-m(S|v∉Top-k)
```
- Base path: 语义相似性
- Flip path: 扩散激活召回的语义距离远但关联强的记忆

## 关键创新

**解决"语义陷阱"问题**：
- 传统 RAG 只看语义相似度，可能遗漏关键记忆
- HeLa-Mem 通过历史共激活建立的路径，能召回语义距离远但关联强的记忆
- 特别适合**多跳推理**场景

**案例（论文 Figure 5）**：
- Query: "Where did you first meet the person who influenced your career choice?"
- 历史共激活建立了 Turn 89（career advice）与 Turn 15（meeting location）的强关联（w=0.52）
- Baseline: 只召回 Turn 89，语义相似度 0.82，但漏掉 Turn 15（相似度仅 0.35）
- HeLa-Mem: 通过 Hebbian path 传播激活，Turn 15 最终得分 0.35 + 0.36 = 0.71，成功召回

## 实验结果

**LoCoMo benchmark**:
- Multi-hop reasoning: 40.14% F1（超越 MemoryOS 38.39%、A-Mem 27.02%）
- Temporal tasks: 47.29% F1
- Token 效率: ~1010 tokens（远低于 full context ~16K）

**平均排名**: 1.25（最佳，超越 MemoryOS 2.25）

## Ablation Study

| Variant | Avg F1 | 下降幅度 |
|---------|--------|----------|
| Full HeLa-Mem | 34.74% | - |
| w/o Reflective Agent | 29.87% | -4.87% (最大影响) |
| w/o Spreading Activation | 32.19% | -2.55% |
| w/o Forgetting | 34.28% | -0.46% |

**关键发现**：
- Reflective Agent（Hebbian Distillation）贡献最大
- Spreading Activation 对多跳推理至关重要
- Forgetting 在当前 benchmark（~300 turns）影响有限，但对长期可扩展性必要

## 与其他方法对比

| 方法 | 机制 | 局限 |
|------|------|------|
| A-Mem | 语义网络 | 缺乏动态关联演化 |
| MemoryBank | 向量库 + 遗忘曲线 | 静态存储，无关联学习 |
| MemGPT | 层级 memory + OS 式读写 | 显式管理，无自发关联 |
| HeLa-Mem | **动态 Hebbian 图 + 蒸馏 + 扩散激活** | 需要足够交互历史积累权重（cold start 问题） |

## 局限性

1. **Cold Start**: Hebbian 权重需要足够交互历史才能积累，早期对话阶段关联检索效果较弱
2. **依赖 LLM 质量**: Semantic Memory 和 Hub Detection 的质量受底层 LLM 能力影响，distillation 过程中的幻觉会传播到长期存储

## 相关概念

- [[Hebbian Learning]]
- [[Episodic-Semantic Memory]]
- [[Spreading Activation]]
- [[Memory Consolidation]]
- [[Multi-hop Reasoning]]
- [[LoCoMo Benchmark]]

---

**创建日期**: 2026-05-25
**标签**: #paper #agentic-memory #hebbian-learning #ACL2026 #bio-inspired