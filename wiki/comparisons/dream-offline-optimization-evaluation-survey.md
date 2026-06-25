---
type: comparison
title: "Dream / 离线优化 / 测评 — 论文总结"
created: 2026-06-23
updated: 2026-06-23
tags:
  - research
  - ai/memory
  - dreaming
  - consolidation
  - offline-optimization
  - evaluation
  - benchmark
related:
  - "[[agent-dream-consolidation-landscape-2025]]"
  - "[[dreaming-consolidation-validation-survey]]"
  - "[[letta-sleep-time-compute-dream-evaluation-analysis]]"
  - "[[LongMemEval Benchmark Deep Dive]]"
---

# Dream / 离线优化 / 测评 — 论文总结

> 围绕 **dream（记忆整合）、离线优化、测评** 三条线的论文清单，按类别整理，每篇一段概述。所有 arXiv 编号均已联网核实。带 ★ 为 2026 年新基准。

---

## 一、Dream / 记忆整合（Consolidation）机制

### [Generative Agents: Interactive Simulacra of Human Behavior](https://arxiv.org/abs/2304.03442)
Park et al., UIST 2023。提出记忆流 + 反思机制：累积重要性超过阈值（≈150）时触发 reflection，生成 3 个高层问题、检索相关记忆、合成抽象洞察存回记忆流，并可多层嵌套（观察→反思→更高层反思）。检索评分用 `recency × importance + relevance` 加权（recency = 0.995^hours）。验证用 25 人结构化访谈打 believability + 逐模块消融，reflection 移除导致 ≈32% 下降（仅次于 memory stream 移除）。是几乎所有 LLM agent reflection 设计的母本。

### [Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366)
Shinn et al., NeurIPS 2023。任务失败后让 LLM 生成自然语言 self-critique，存入 persistent memory buffer，下次尝试时检索注入 prompt——全程无参数更新，纯 in-context 学习。验证采用跨多轮 reflection 的迭代学习曲线（HumanEval / ALFWorld / WebShop），核心结果 ALFWorld 97% vs ReAct 77%（+20pp），reflection 贡献约 20–24% 绝对提升。是"自我批评→持久化→下次注入"这一闭环的代表。

### [Voyager: An Open-Ended Embodied Agent with Large Language Models](https://arxiv.org/abs/2305.16291)
Wang et al. (NVIDIA), NeurIPS 2023。把验证成功的 JavaScript 技能存入按语义描述索引的 skill library，失败时 LLM 反思错误→修正→重新验证，技能之间可组合复用（如 exploreCave 调用 combatZombie）。验证用 Minecraft 执行测试 + skill library 消融，核心结果 3.3× unique items、2.3× tech milestones，技能库移除致 tech tree mastery 下降约 27%。其"可组合复用的技能层级"启发了 dream 产出的层级引用链设计。

### [MemGPT: Towards LLMs as Operating Systems](https://arxiv.org/abs/2310.08560)
Packer et al., 2023（Letta 前身）。把 agent 记忆建模为 OS 式三层 core ↔ archival ↔ recall，LLM 通过 function call 自主管理 tier 切换与分页。验证覆盖超 context 窗口的文档分析，结论是自主 memory 管理优于外部 heuristic。后续 Letta 产品线把这条思路演化为 5-phase reflection 流水线（Investigate→Extract→Update→Review→Commit）。

### [Mastering Diverse Domains through World Models (DreamerV3)](https://arxiv.org/abs/2301.04104)
Hafner et al., 2023。用 RSSM 学习 world model，policy 完全在 latent 空间做 imagined rollouts 训练（1 步真实交互→100+ 步想象训练），首次从零在 Minecraft 获得钻石。它的更大价值是评估方法论范式——固定单一超参数跨 7 个 benchmark、多 seed（5–10）、IQM（25–75 分位）+ stratified bootstrap 95% CI + performance profiles，系统纠正了深度 RL 评估的单 seed / 报均值 / 无置信区间等陋习。这套统计严谨性被建议直接迁移到 dream 验证。

### [Continual Learning with Deep Generative Replay (DGR)](https://arxiv.org/abs/1705.10306)
Shin et al., NeurIPS 2017。用 GAN 生成伪数据回放旧任务经验 + self-replay 防止灾难性遗忘，无需存储真实数据。验证在 Permuted MNIST 的 10 任务序列上，报告平均准确率 + forgetting measure，对比 EWC/SI/真数据回放，结果约 82–87%（naive ≈60%，真数据回放 ≈85–90%）。它的"生成式回放闭环"（generator 自身也在回放中更新）启发了"被晋升的源记忆应降低检索权重"的设计。

### [ExpeL: LLM Agents Are Experiential Learners](https://arxiv.org/abs/2308.10144)
Zhao et al. (清华 LeapLab), AAAI 2024。双记忆架构（trajectory + insight），从多条轨迹中提取可泛化的 insights、验证后存入知识库。验证在 HotPotQA / ALFWorld 上做跨 episode 渐进曲线，结论是多轨迹 consolidation 显著优于单轨迹 reflection。是"从经验中归纳可复用知识"这一类 consolidation 的代表。

### [MemoryBank: Enhancing Large Language Models with Long-Term Memory](https://arxiv.org/abs/2305.10250)
Zhong et al., AAAI 2024。用 Ebbinghaus 遗忘曲线 $R = e^{-t/S}$ 管理记忆衰减，回忆时对记忆做 reinforcement（$S \times 1.6$），强度低于阈值则 prune。验证用多 session 对话一致性 + 个人化质量，效果呈现"类人遗忘"——记住重要的、忘掉不重要的。是显式衰减/遗忘机制的代表。

### [RecallM: An Adaptable Memory Mechanism with Temporal Understanding](https://arxiv.org/abs/2307.02738)
Kynoch & Latapie, 2023。提出基于图的 adaptable memory 机制，支持时间理解（temporal understanding）与记忆更新，能在答案随时间变化的场景下保持一致。验证用 temporal QA 基准，显著优于 naive RAG。是"时序感知记忆"方向的代表。（注：wiki 早期笔记把它误记为 "Zahid 2024 + 矛盾三分类"，实际作者与机制以本文为准。）

---

## 二、离线优化（Offline / Sleep-time Optimization）

### [Sleep-Time Compute: Leveraging Inactive Periods for Latency-Sensitive Inference](https://arxiv.org/abs/2504.13171)
Lin et al. (Letta + UC Berkeley), 2025。核心范式 $S(c) \to c'$：在用户离开的"睡眠时间"只看 context（不看 query）做预推理，把 c 重写为含中间推理步骤的 c'，用户来时用更小预算即可达到同等准确率。验证的是**数学推理的 compute efficiency**（Stateful GSM-Symbolic / AIME / SWE-Features），低 test-time budget 下省约 5× tokens、准确率提升 up to 13%（GSM）/18%（AIME），且仅在 query 可预测时有效。**重要澄清**：它验证的不是"对话记忆整理效果"——与本项目的 dream/consolidation 目标不同，其定量结果不可直接迁移，可复用的只是 pareto 对比 + before/after 框架结构。

### CLS Theory — McClelland, McNaughton & O'Reilly (1995)
*Why There Are Complementary Learning Systems in the Hippocampus and Neocortex*, Psychological Review 102(3):419–457, 1995。互补学习系统理论：海马体快速编码（pattern separation，对应即时抽取去重）、新皮层缓慢整合（pattern completion，对应离线 dream），两者靠 replay 渐进迁移。它给出三条工程约束——pattern separation（去重）、interleaved learning（新旧混合处理而非只处理新记忆）、slow consolidation（多轮渐进而非一次性）——是所有 dream/consolidation 机制的神经科学基础，也直接对应 oG-Memory Deep Dream 的改进方向（混合检索、多轮渐进）。

---

## 三、测评基准（Benchmarks）

### [LongMemEval: Benchmarking Chat Assistants on Long-Term Interactive Memory](https://arxiv.org/abs/2410.10813)
Wu et al., ICLR 2025。长期对话记忆基准，500 题、~40–500 个历史会话（S/M 两档，~115k–更大量 token）。测 5 类能力：single-session-user/assistant（单会话提取）、single-session-preference（隐含偏好）、multi-session（跨会话聚合推理）、temporal-reasoning（时间差/顺序）、knowledge-update（追踪当前值忽略旧值）、abstention（识别不可答）。流程为 Indexing→Retrieval→Reading，检索用 recall@k / ndcg，问答用 LLM-as-Judge（GPT-4o）。关键局限：单用户、无 user_id/agent_id、静态一次性注入、纯英文、只测"记忆问答"不测"基于记忆执行任务"。

### ★ [LongMemEval-V2: Evaluating Long-Term Agent Memory Toward Experienced Colleagues](https://arxiv.org/abs/2605.12493)
Wu et al. (UCLA), 2026。项目页 xiaowu0162.github.io/longmemeval-v2。V1 的升级版：场景从"聊天助手长期交互记忆"升级为**多模态 Web Agent 轨迹**——评估记忆系统能否把超长 agent 历史转化为可复用的"环境经验"，让 agent 成为定制环境里的"老练同事"。451 题，历史 haystack 达 **2500 万–1.15 亿 token**（V1 仅 11.5 万–150 万），支持截图+文本多模态。测 5 类维度：静态状态回忆、动态状态追踪、**工作流知识**、**环境陷阱**、前提意识。统一 Insert(h)/Query(q)→Reader 接口，指标为答案准确率 + 查询延迟，用 **LAFS Gain**（accuracy-latency 前沿）综合排序；最强基线 AgentRunbook-C（文件式编码记忆控制器）平均 72.5%，远超最强 RAG（48.5%），纯 reader 仅 1.3%。

### [Evaluating Very Long-Term Conversational Memory of LLM Agents (LoCoMo)](https://arxiv.org/abs/2402.17753)
Maharana et al. (Snap Research), ACL 2024。非常长程的对话记忆基准，每段对话约 300 轮、9K token、跨至多 35 个 session，提供问答等综合任务。是 ScallopBot、OpenDream、mem0、Letta 等项目的主流评测集；Letta 团队 2025 年 8 月报告：纯文件系统 + grep/search（74.0%）反超 mem0 graph（68.5%），得出"agent 能力 > 记忆工具类型"的结论。局限是只测 QA accuracy，不测 dreaming 过程本身。

### ★ [MemoryAgentBench: Evaluating Memory in LLM Agents via Incremental Multi-Turn Interactions](https://arxiv.org/abs/2507.05257)
Hu, Wang, McAuley (UC San Diego), ICLR 2026。GitHub: HUST-AI-HYZ/MemoryAgentBench。主张 memory agent 必须同时具备四项能力，而现有基准无一能全覆盖：**Accurate Retrieval（AR）、Test-Time Learning（TTL）、Long-Range Understanding（LRU）、Selective Forgetting / Conflict Resolution（CR）**。方法是把长上下文数据集**改造为多轮格式**（inject once, query multiple，逐 chunk 注入模拟增量累积），并新构造 EventQA 与 FactConsolidation 两个数据集。指标按任务映射（substring_exact_match / exact_match / Recall@5 / LLM-as-judge GPT-4o）。评测覆盖 context-based、RAG、cognee、letta 等方法，核心结论是**当前没有任何方法能同时掌握全部四项记忆能力**。我们已用 oG-Memory 对接其 LongMemEval S* 子集做 before/after dream 评测。

### ★ [Mem2ActBench: A Benchmark for Evaluating Long-Term Memory Utilization in Task-Oriented Autonomous Agents](https://arxiv.org/abs/2601.19935)
Shen et al. (中科院信息工程研究所), 2026。聚焦被忽视的能力——**主动利用长期记忆执行工具动作**：给定欠指定指令（关键参数被省略），agent 须推断该从历史检索什么约束并 ground 成可执行工具调用（选对工具 + 填对参数），模拟跨多次无关对话打断的"持久助手"场景。三阶段自动流水线（ToolACE+BFCL_v3 数据融合 + OASST1 噪声 → BERTopic+HDBSCAN 构建 Fact Evolution Chain → 反向生成记忆依赖型任务）产出 **2029 个长会话、400 个记忆依赖型工具调用任务**（人工验证 91.3% 强依赖记忆）。指标为参数级 F1、BLEU-1、Tool Accuracy（TA），并按 slot 类型（explicit/inferred/default）拆分。测 7 种记忆方法（LTMemory-RAG、Generative Agents、SCM、Langmem、MemTree、Mem0、A-Mem）×3 个 Qwen 规模；核心结论：TA 普遍高（~87–97%）但参数 grounding 差距大，瓶颈在**证据命中/检索质量而非推理**（oracle 检索比被动检索高 23 个 F1 点）。

---

## 引用核实说明

本次整理联网核实了所有 arXiv 编号。过程中发现 wiki 既有调研里存在几处需修正的引用：
- **ExPeL** 第一作者是 Andrew Zhao（清华 LeapLab）、arXiv:2308.10144、AAAI 2024（非 "Zhong"）。
- **MemoryBank**（Wanjun Zhong, AAAI 2024）与 ExPeL 不是同一作者组。
- **RecallM** 实际为 Kynoch & Latapie 2023、arXiv:2307.02738（非 "Zahid 2024"，矛盾三分类的描述未能证实，已改用论文实际机制）。
- **SCM（arXiv:2408.01322）** 该编号实际对应一篇计算机视觉 scanpath 论文，与 LLM agent 记忆无关——已从清单移除（仅作为 Mem2ActBench 的被测方法之一被提及）。
- **"Farooq & Bhatt 2024 Nature / Sleep Rewrites Rules"** 未能核实到对应论文，已从清单移除，避免引用存疑文献。
