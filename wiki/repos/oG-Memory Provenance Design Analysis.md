---
type: repo
status: active
created: 2026-05-13
updated: 2026-05-13
tags: [ogmemory, provenance, design, analysis]
---

# oG-Memory 记忆溯源方案设计分析（初版）

> 初版三方案对比分析。深入方案设计见 [[oG-Memory Schema-Driven Provenance Design]]。

## 现状

oG-Memory 溯源能力几乎为零。关键断裂点：`_Span(start, end)` 在 Phase 1 已携带消息范围，但构建 CandidateMemory 时被完全丢弃。

已有微弱关联：
- `evidence_quote`: 语义溯源（逐字引用），但无位置信息
- `_Span.start/end`: 有消息索引，但丢弃了
- `ctx.session_id`: 请求上下文有 session ID，但未写入 context_nodes

数据流断裂：`messages → _Span ✓ → CandidateMemory ✗ → ContextNode ✗ → context_nodes ✗`

## 三个方案

### A: Metadata 注入（~20 行，3 文件）
在 CandidateMemory 增加 source 字段，沿 pipeline 传到 metadata JSONB。最小改动但 merge 覆盖历史来源。

### B: Provenance 表（~60 行，新建表）
独立 provenance 表支持多来源追溯、merge 后不丢历史。完整但较重。

### C: Schema-driven（~50 行，改 schema 模型）
YAML 中声明 `auto_inject` 字段，框架自动注入。符合 oG-Memory 设计哲学。

## 初版推荐

A + B 渐进组合。但方案 C 更符合 schema-driven 设计理念——见深入分析。

## 学术对照

| 学术方案 | 对应 |
|----------|------|
| MemORAI turn-level provenance | source_session_id + source_span |
| TROVE 句子级溯源 | evidence_quote 部分替代 |
| AEVS anchor-to-source | source_span 锚定 |
