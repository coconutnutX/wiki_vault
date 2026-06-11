---
type: repo
url: "https://github.com/xiaowu0162/LongMemEval"
language: Python
status: active
key_takeaway: "长期对话记忆评测基准，测试5种核心能力，模拟真实多会话场景，强调在线解析+检索+问答流程"
relevant_to: [oG-Memory, agent-memory, benchmark]
tags: [repo, benchmark, memory-eval, ICLR-2025]
created: 2026-06-10
updated: 2026-06-10
---

# LongMemEval Benchmark Deep Dive

## Overview

**LongMemEval** 是 ICLR 2025 发表的长期对话记忆评测基准，专门测试 Chat Assistant 在持续交互中的长期记忆能力。

- **论文**: [LongMemEval: Benchmarking Chat Assistants on Long-Term Interactive Memory](https://arxiv.org/abs/2410.10813)
- **代码**: https://github.com/xiaowu0162/LongMemEval
- **数据**: https://huggingface.co/datasets/xiaowu0162/longmemeval-cleaned

核心设计理念：模拟真实用户与 Assistant 的多会话长期交互，测试系统在「在线解析动态交互历史」后回答问题的能力。

## 测试的五种核心能力

| 类型 | 测试目标 | 示例问题 |
|------|----------|----------|
| **single-session-user** | 信息提取（单会话用户侧） | "What degree did I graduate with?" |
| **single-session-assistant** | 信息提取（单会话助手侧） | "What was the rotation for Admon on a Sunday?" |
| **single-session-preference** | 用户偏好理解 | 推荐视频编辑资源时，用户偏好 Adobe Premiere Pro 相关内容 |
| **multi-session** | 跨会话推理 | "How many items of clothing do I need to pick up or return?" (需聚合多个会话信息) |
| **temporal-reasoning** | 时间推理 | "How many days passed between my visit to MoMA and the Met exhibit?" |
| **knowledge-update** | 知识更新追踪 | "What was my personal best time in the 5K run?" (需追踪最新值，忽略旧值) |
| **abstention** | 识别不可回答问题 | "What is the name of my hamster?" (用户从未提及，应回答「不知道」) |

**数据统计**:
- 500 个评测问题
- ~40-50 个历史会话 (LongMemEval-S)，可扩展到 500+ (LongMemEval-M)
- 每个会话含多个 turn（user/assistant 对话）

## 针对的场景

**真实场景模拟**:
1. 用户与 Assistant 进行多会话持续交互（跨度可能数周/数月）
2. 用户在历史会话中提及各类个人信息、偏好、事件
3. 系统需要在线解析这些交互（而非预先存储结构化知识）
4. 系统需要从海量历史中检索相关信息回答新问题
5. 系统需要处理知识更新（用户改变偏好/状态）
6. 系统需要正确识别无法回答的问题（避免幻觉）

**核心挑战**:
- 商业 Chat Assistant 和长上下文 LLM 在此基准上准确率下降 ~30%
- 需要在 ~115k token 的历史中找到特定信息片段
- 多会话推理需要聚合分散在不同会话的信息
- 知识更新需要正确追踪「当前值」（而非返回过时信息）

## 限制与局限性

1. **单用户场景**: 每个评测实例假设单一用户与单一 Assistant 的交互，无多用户区分
2. **无 user_id/agent_id**: 数据结构中不包含用户/助手标识符，复现时需自行假设
3. **静态评测**: 评测是一次性的（给定历史+问题→答案），不测试持续交互中的实时记忆更新
4. **人工构造**: 历史会话由 ShareGPT/UltraChat + 模拟生成，非真实用户对话
5. **英文为主**: 数据主要为英文，多语言能力未覆盖
6. **无 agent 行为测试**: 只测试「记忆问答」，不测试「基于记忆执行任务」
7. **oracle 检索易**: `longmemeval_oracle.json` 只包含 evidence sessions，实际场景需处理海量噪音

## 运行流程详解

### 数据结构

```json
{
  "question_id": "e47becba",
  "question_type": "single-session-user",
  "question": "What degree did I graduate with?",
  "question_date": "2023/05/30 (Tue) 23:40",
  "answer": "Business Administration",
  "answer_session_ids": ["answer_280352e9"],
  "haystack_dates": ["2023/05/20 (Sat) 02:21", ...],  // 所有历史会话时间戳
  "haystack_session_ids": ["sharegpt_yywfIrx_0", ...],  // 所有历史会话ID
  "haystack_sessions": [
    [
      {"role": "user", "content": "..."},
      {"role": "assistant", "content": "..."},
      {"role": "user", "content": "...", "has_answer": true}  // evidence turn 标记
    ],
    ...
  ]
}
```

### 评测流程

**Step 1: 准备历史数据**
```bash
# 下载数据
mkdir -p data/
cd data/
wget https://huggingface.co/datasets/xiaowu0162/longmemeval-cleaned/resolve/main/longmemeval_oracle.json
wget https://huggingface.co/datasets/xiaowu0162/longmemeval-cleaned/resolve/main/longmemeval_s_cleaned.json
```

**Step 2: 选择评测模式**

| 模式 | 描述 | 历史长度 |
|------|------|----------|
| `longmemeval_oracle` | 只含 evidence sessions | ~几k token |
| `longmemeval_s` | ~40 sessions, ~115k token | 中等难度 |
| `longmemeval_m` | ~500 sessions | 极高难度 |

**Step 3: 运行评测系统**

系统需实现三阶段流程：

1. **Indexing**: 将历史会话建立索引
   - 粒度可选：`session`（整个会话作为检索单元）或 `turn`（单轮对话作为单元）
   - 支持扩展：session summary / keyphrase / user fact
   
2. **Retrieval**: 从历史中检索相关记忆
   ```bash
   cd src/retrieval
   bash run_retrieval.sh ../../data/longmemeval_s_cleaned.json flat-stella session
   ```
   
3. **Reading**: 用检索结果回答问题
   ```bash
   cd src/generation
   bash run_generation.sh retrieval_log_file gpt-4o flat-stella-session 30 json false con
   ```

**Step 4: 评估结果**
```bash
cd src/evaluation
python evaluate_qa.py gpt-4o hypothesis_file ../../data/longmemeval_oracle.json
```

### 关键流程细节

**注入方式**: 
- 全量注入（一次性提供所有历史）
- 或检索后注入（先检索相关 session/turn，再构建 prompt）

**检索评估指标**:
- `recall_any@k`: k 内是否命中任意一个正确 session/turn
- `recall_all@k`: k 内是否命中所有正确 session/turn
- `ndcg@k`: 排序质量

**问答评估**:
- 使用 LLM-as-Judge（GPT-4o/Llama-3.1-70B）判断答案正确性
- 不同问题类型有不同判断 prompt（如 knowledge-update 允许部分正确）
- abstention 问题单独评估

## 复现注意事项

### 1. 无 user_id/agent_id 区分

数据假设单一用户场景。如需支持多用户：
- 需自行添加 user_id 字段
- 检索时需按 user_id 过滤
- 问答 prompt 需明确「这是关于哪个用户的问题」

### 2. 历史会话格式

历史以 JSON 格式提供，prompt 构建时可选择：
- `json`: 直接 JSON string（推荐）
- `nl`: 自然语言格式化

### 3. 时间戳处理

- `haystack_dates` 和 `question_date` 提供时间信息
- temporal-reasoning 类型问题需要利用时间戳计算时间差
- 检索时可做时间范围过滤

### 4. Evidence 标记

- `has_answer: true` 标记哪些 turn 包含答案
- 仅用于评测（计算 recall），实际系统无法看到此标记
- `answer_session_ids` 标记哪些 session 包含答案

### 5. Knowledge Update 处理

- 同一事实可能在不同 session 有不同值
- 系统需返回「最新值」
- 检索时需考虑时间顺序或 max(timestamp)

### 6. 检索粒度选择

- `session` 粒度：检索单元是整个会话，可能包含大量无关 turn
- `turn` 粒度：检索单元是单轮对话，更精准但索引更大
- 论文推荐 session 粒度 + index expansion

### 7. Index Expansion 策略

论文提出多种索引扩展方法：
- `session-summ`: 用 LLM 生成会话摘要作为额外索引键
- `session-keyphrase`: 提取关键词作为索引键
- `session-userfact`: 提取用户事实作为索引键
- `turn-keyphrase`: turn 级别的关键词提取

### 8. CoN (Conversational Notes) 模式

可选的阅读增强：
- 先用 LLM 从检索结果提取「阅读笔记」
- 再用笔记回答问题
- 类似 ReAct 的先观察后推理

## 与其他 Benchmark 对比

| Benchmark | 特点 | 与 LongMemEval 区别 |
|-----------|------|---------------------|
| LoCoMo | 多会话记忆 QA | 更偏工业场景，会话更少 |
| MemoryBench | 单轮记忆测试 | 无多会话推理 |
|needle-in-a-haystack | 单事实检索 | 无对话语境，无推理 |
| RULER | 长上下文压力测试 | 无对话结构，无知识更新 |

## 对 oG-Memory 的启示

1. **评测指标**: 可借鉴 session/turn 级 recall + QA accuracy 双指标体系
2. **时间处理**: temporal-reasoning 测试提醒系统需记录并利用时间信息
3. **知识更新**: 需支持「当前值」追踪，而非只存储历史事实
4. **多粒度索引**: session vs turn 粒度的选择与扩展策略
5. **在线解析**: 强调从原始对话实时提取记忆，而非依赖预处理
6. **abstention**: 需支持「我不知道」响应，而非强答导致幻觉

## Open Questions

1. 如何将 LongMemEval 的评测流程适配到多用户场景？
2. oG-Memory 的 Dreaming 机制能否改善 knowledge-update 类问题的表现？
3. LongMemEval 的 index expansion 策略如何与 oG-Memory 的 extraction pipeline 结合？
4. 如何构建类似 LongMemEval 的中文评测数据？
5. 是否需要增加「基于记忆执行任务」而非只做 QA 的评测类型？

## References

- 论文: https://arxiv.org/abs/2410.10813
- GitHub: https://github.com/xiaowu0162/LongMemEval
- HuggingFace: https://huggingface.co/datasets/xiaowu0162/longmemeval-cleaned
- LongMemEval-V2 (2026): https://github.com/xiaowu0162/LongMemEval-V2 (agentic context)