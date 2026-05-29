# HeLa-Mem 延伸相关工作（ACL/AAAI/NeurIPS 2026）

从 HeLa-Mem (arXiv:2604.16839, ACL 2026) 的生物启发记忆架构出发，近期顶会涌现大量相关工作。

## ACL 2026 Main Conference

### Auto-Dreamer: Learning Offline Memory Consolidation for Language Agents

- **arXiv**: 2605.20616
- **核心思想**: 明确区分 fast acquisition vs slow cross-session consolidation
- **机制**:
  - Fast per-session memory acquisition
  - Slow offline consolidation（跨 session 蒸馏）
  - GRPO RL 训练，end-to-end agent performance 作为 reward
- **实验**: ScienceWorld +7 pts，内存减少 12x；ALFWorld/WebArena zero-shot 迁移
- **与 HeLa-Mem 对比**:
  - HeLa-Mem: Reflective Agent 检测 hub 时触发 online 蒸馏
  - Auto-Dreamer: 独立 offline phase，学习如何 consolidate

### StructMem: Structured Memory for Long-Horizon Behavior in LLMs

- **arXiv**: 2604.21748
- **会议**: ACL 2026 Main
- **GitHub**: https://github.com/zjunlp/LightMem
- **核心思想**: Structure-enriched hierarchical memory
- **机制**:
  - Temporal anchoring dual perspectives
  - Periodic semantic consolidation
  - 事件级 bindings + 跨事件 connections
- **关键贡献**: temporal reasoning + multi-hop 性能提升，token/API/runtime 大幅减少
- **与 HeLa-Mem 对比**: 同样强调 episodic-semantic 分层，但 consolidation 机制为周期性而非 hub-triggered

### RecMem: Recurrence-based Memory Consolidation

- **arXiv**: 2605.16045
- **会议**: ACL 2026 Findings
- **核心思想**: Rethinking **when** consolidation should happen
- **机制**:
  - Subconscious memory layer（轻量 embedding 编码）
  - 仅在 sustained recurrence 出现时触发 LLM extraction
  - Semantic refinement 机制恢复遗漏 facts
- **实验**: Token 消耗减少 87%（对比 3 SOTA systems），精度超越 baseline
- **与 HeLa-Mem 对比**:
  - HeLa-Mem: 每次 retrieval 都更新 Hebbian 边权重（online）
  - RecMem: Lazy consolidation，只在语义 cluster 有足够信息时才调用 LLM

### HyperMem: Hypergraph Memory for Long-Term Conversations

- **arXiv**: 2604.08256
- **会议**: ACL 2026 Main
- **核心思想**: 用 hyperedge 捕捉 high-order associations
- **架构**:
  - 三层: Topics → Episodes → Facts
  - Hyperedge: groups related episodes + their facts
  - Hybrid lexical-semantic index + coarse-to-fine retrieval
- **实验**: LoCoMo 92.73% LLM-as-a-judge accuracy
- **与 HeLa-Mem 对比**:
  - HeLa-Mem: pairwise edges（节点-节点）
  - HyperMem: hyperedge（多元素联合依赖）→ 更适合捕捉 multi-hop reasoning 的多因素关联

---

## ACL 2026 Findings

### MARS: Agentic Recommender System with Hierarchical Belief-State Memory

- **arXiv**: 2605.14401
- **核心思想**: Recommendation as partially observable problem
- **架构**:
  - Event memory: buffers raw signals
  - Preference memory: mutable chunks + strength/evidence tracking
  - Profile memory: distilled natural language narrative
- **完整生命周期**: 6 operations
  - Extraction
  - Reinforcement
  - Weakening
  - Consolidation
  - Forgetting
  - Resynthesis
- **关键创新**: LLM-based planner 自适应调度（而非 fixed-interval heuristics）
- **实验**: InstructRec benchmark，HR@1 +26.4%，NDCG@10 +10.3%

### "Useful Memories Become Faulty When Continuously Updated by LLMs"

- **arXiv**: 2605.12978
- **重要发现**: LLM consolidation 反而可能退化记忆质量
- **现象**:
  - Memory utility 先上升，后下降
  - 可能跌至 no-memory baseline 以下
  - GPT-5.4 在 ARC-AGI 上，即使从 ground-truth consolidate，仍有 54% 失败率
- **原因分析**:
  - 同样 trajectories 在不同 update schedules 下产生不同记忆质量
  - Episodic-only control（保留原始轨迹）反而 competitive
- **建议**:
  - Raw episodes 应作为 first-class evidence
  - Gate consolidation explicitly（不要每次交互后自动触发）
- **启示**: 对 HeLa-Mem 的 Reflective Agent 设计提出质疑——过度蒸馏可能有害

### Memora: From Recall to Forgetting Benchmark

- **arXiv**: 2604.20006
- **会议**: ACL 2026 Findings
- **核心思想**: 长期记忆不只是 recall，还需处理 knowledge updates
- **benchmark**: Memora（weeks-to-months conversations）
- **新 metric**: FAMA (Forgetting-Aware Memory Accuracy)
  - Penalty: 使用 obsolete/invalidated memory
- **发现**: 现有 memory agents 频繁 reuse invalid memories，fail to reconcile evolving memories

---

## AAAI 2026 (Workshop/Bridge)

### SOLAR: Self-Optimizing Lifelong Autonomous Reasoner

- **arXiv**: 2605.20189
- **会议**: AAAI 2026 Streaming Continual Learning Bridge
- **核心思想**: Parameter-level meta-learning
- **机制**:
  - Treating model weights as environment to explore
  - Multi-level RL: Level I (single-edit) → Level II (chained strategies) → Level III (open-ended)
  - Evolving knowledge base = episodic memory buffer
- **五大策略家族**:
  - TTT (Test-Time Training)
  - LoRA Modifications
  - RL Frameworks (SQLM/R-Zero/SEAL)
  - TTS (Test-Time Scaling)
  - Latent Space approaches
- **实验**: Common-sense/math/medical/code/social/logical reasoning，超越 DnD/TTL/DOM

### DecentMem: Self-Evolving Multi-Agent Systems via Decentralized Memory

- **arXiv**: 2605.22721
- **核心思想**: 从 centralized → decentralized memory
- **架构**:
  - Dual-pool per agent:
    - Exploitation pool: consolidated past trajectories
    - Exploration pool: LLM-generated candidates
  - Online reweighting via LLM-as-a-judge
- **理论**: O(log T) cumulative regret，匹配 stochastic bandit lower bound
- **实验**: AutoGen/DyLAN/AgentNet + Qwen3/Gemma4，+23.8% over centralized baseline

---

## 生物启发/神经科学相关工作（Preprint）

### Human-Inspired Memory Architecture for LLM Agents

- **arXiv**: 2605.08538
- **六大认知机制**:
  1. Sleep-phase consolidation
  2. Interference-based forgetting
  3. Engram maturation
  4. Reconsolidation upon retrieval
  5. Entity knowledge graphs
  6. Hybrid multi-cue retrieval
- **实验**: VSCode issue-tracking (13K issues)，LongMemEval streaming M-tier
- **创新**: Synthetic calibration（无 benchmark data exposure）
- **与 HeLa-Mem 关系**: 更完整的生物机制集合，sleep-phase 是 offline consolidation 的生物基础

### FSFM: Biologically-Inspired Framework for Selective Forgetting

- **arXiv**: 2604.20300
- **理论基础**:
  - Hippocampal indexing/consolidation theory
  - Ebbinghaus forgetting curve
- **四大遗忘机制 taxonomy**:
  1. Passive decay-based
  2. Active deletion-based
  3. Safety-triggered
  4. Adaptive reinforcement-based
- **实验**: Access efficiency +8.49%，SNR +29.2%，security risks 100% elimination
- **与 HeLa-Mem 对比**: HeLa-Mem 的 Adaptive Forgetting 可参考此 taxonomy

### BrainMem: Brain-Inspired Evolving Memory for Embodied Agents

- **arXiv**: 2604.16331
- **领域**: Embodied task planning (EB-ALFRED/EB-Navigation/EB-Manipulation)
- **架构**:
  - Working memory
  - Episodic memory
  - Semantic memory → knowledge graphs + symbolic guidelines
- **特点**: Training-free, plug-and-play, 与任意 multimodal LLM 集成
- **实验**: Long-horizon + spatially complex tasks 性能显著提升

---

## 关键趋势分析（从 HeLa-Mem 延伸）

### 1. 从 Eager → Lazy Consolidation

| 方法 | 时机 | Token 效率 |
|------|------|-----------|
| HeLa-Mem | 每次 retrieval 更新边权重 | 中等 |
| RecMem | 仅 recurrence 时触发 LLM | 减少 87% |

### 2. 从 Online → Offline Consolidation

| 方法 | 模式 | 优势 |
|------|------|------|
| HeLa-Mem | Reflective Agent 检测 hub → online 蒸馏 | 即时性 |
| Auto-Dreamer | Fast acquisition + slow offline consolidation | 跨 session 发现 recurring patterns |

### 3. 从 Pairwise → High-Order Association

| 方法 | 结构 | 适用场景 |
|------|------|----------|
| HeLa-Mem | Hebbian graph（nodes + pairwise edges） | 双元素关联 |
| HyperMem | Hypergraph（hyperedge = 多元素联合） | Multi-hop reasoning（多因素依赖） |

### 4. 从盲目 Consolidation → 审慎 Consolidation

| 论文 | 发现 | 建议 |
|------|------|------|
| "Useful Memories Become Faulty" | 过度 consolidate 反有害 | Raw episodes = first-class evidence |
| HeLa-Mem | Hub detection 触发蒸馏 | 需验证是否过度蒸馏 |

### 5. 完整 Memory Lifecycle

| 方法 | 操作集 |
|------|--------|
| HeLa-Mem | Association + Consolidation + Forgetting + Retrieval |
| MARS | Extraction + Reinforcement + Weakening + Consolidation + Forgetting + Resynthesis |
| SOLAR | RL-based strategy discovery + KB enrichment |

---

## 相关概念

- [[HeLa-Mem-Hebbian-Learning-Associative-Memory]]
- [[Memory Consolidation]]
- [[Episodic-Semantic Memory]]
- [[Spreading Activation]]
- [[Hebbian Learning]]
- [[Lazy Consolidation]]
- [[Offline Memory Consolidation]]
- [[Hypergraph Memory]]
- [[Memory Lifecycle]]

---

**创建日期**: 2026-05-25
**标签**: #survey #agentic-memory #ACL2026 #AAAI2026 #bio-inspired #memory-consolidation