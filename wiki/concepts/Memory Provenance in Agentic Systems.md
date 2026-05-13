---
type: concept
status: active
created: 2026-05-13
updated: 2026-05-13
tags: [memory, provenance, agentic-ai, source-attribution, research]
---

# Memory Provenance in Agentic Systems

## Problem Statement

在开发 agentic memory system 时，记忆经过抽取、压缩或整理后，如何对应回原始文本？例如原始 transcript 被处理为结构化记忆后，如何知道每条记忆来自哪段原始对话。

## Key Terminology

| Term | Meaning | Granularity |
|------|---------|-------------|
| **Provenance** | 数据的血统/来源追踪 | 最通用 |
| **Source Attribution** | 将输出归因到源 | 文档级或句子级 |
| **Grounding** | 将抽取结果锚定到原文 | span 级 |
| **Traceability** | 可追溯性 | 工程视角 |

## Directly Relevant Papers

### TROVE — Text pROVEnance (ACL 2025)

将溯源问题形式化为：将 LLM 生成的**每个句子**追溯到参考文档中具体的源句子。区别于传统的文档级归因，做的是句子级（fine-grained）溯源。

- Link: [ACL Anthology](https://aclanthology.org/2025.acl-long.577/)
- arXiv: [2503.15289](https://arxiv.org/html/2503.15289v1)

### MemORAI (arXiv 2605.01386)

提出 **Provenance-Enriched Graph Construction**：LLM 抽取实体-关系三元组时，在每个三元组上附带 **turn-level provenance**（追溯到对话中的具体轮次）。双层压缩同时保留溯源信息。最接近 transcript → structured memory 的场景。

- Link: [arXiv HTML](https://arxiv.org/html/2605.01386v1)
- HuggingFace: [papers/2605.01386](https://huggingface.co/papers/2605.01386)
- Core innovations: selective memory filtering, adaptive graph-based memory organization, context-aware retrieval

### AEVS Framework (MDPI Computers)

**Anchor-Extraction-Verification-Supplement** 框架，强制将每个知识三元组的每个元素（头实体、关系、尾实体）锚定到原文的具体位置。

- Link: [MDPI Computers](https://www.mdpi.com/2073-431X/15/3/178)

### SymGen — Symbolic References (OpenReview)

让 LLM 在生成文本时交替插入对源数据字段的**符号引用**，使每段输出都能验证对应到哪个源字段。

- Link: [OpenReview](https://openreview.net/forum?id=fib9qidCpY)

### Verifiable Generative Provenance for Multimodal Tool-Using Agents (arXiv 2605.09934)

提供更细粒度的方法来追踪生成文本如何从源材料中产生，引用 TROVE 等工作。

- Link: [arXiv HTML](https://arxiv.org/html/2605.09934v1)

## Surveys

### Anatomy of Agentic Memory (arXiv 2602.19320)

对 agentic memory 系统做系统性的分类学分析，从架构和系统两个维度分类，分析实证局限性。

- Link: [arXiv](https://arxiv.org/abs/2602.19320)
- GitHub: [FredJiang0324/Anatomy-of-Agentic-Memory](https://github.com/FredJiang0324/Anatomy-of-Agentic-Memory) — 按分类学组织的论文列表

### Tracing the Data Trail: A Survey of Data Provenance in LLMs

综述 LLM 全链路中的数据溯源、透明度和可追溯性。

- Link: [ResearchGate](https://www.researchgate.net/publication/399965419_Tracing_the_Data_Trail_A_Survey_of_Data_Provenance_Transparency_and_Traceability_in_LLMs)

### Attribution, Citation, and Quotation: A Survey of Evidence-Based Text Generation

134 篇论文的大规模综述，覆盖 attribution、citation 和 quotation。

- Link: [TheMoonlight.io](https://www.themoonlight.io/en/review/attribution-citation-and-quotation-a-survey-of-evidence-based-text-generation-with-large-language-models)

## Practical Implementations

- **[agentmemory](https://github.com/rohitg00/agentmemory)** — 每个 memory 操作发出 trace span，内置 provenance 追踪
- **[Neon-Soul](https://github.com/geeks-accelerator/neon-soul)** — 每个 axiom 追溯到 memory 文件中的确切行号

## Practical Recommendation

在 agentic memory 系统中实现溯源，最实用的方案是在抽取时为每条记忆附加元数据：

```
(source_id, span_start, span_end)
```

例如从 transcript 中抽取记忆时，记录：来自哪个对话文件、哪一行到哪一行。

## Other Notable Mentions

- **AgenTracer** ([OpenReview](https://openreview.net/forum?id=l05DseqvuD)) — multi-agent 系统中的故障归因框架
- **PRISM** ([arXiv](https://arxiv.org/html/2604.16909v1)) — 探测 LLM 的推理、指令和源记忆能力
- **CAMS** ([ScienceDirect](https://www.sciencedirect.com/science/article/pii/S1110866526001003)) — 保护 LLM agent 长期记忆免受注入攻击
