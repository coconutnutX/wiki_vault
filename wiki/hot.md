---
type: meta
title: "Hot Cache"
updated: 2026-06-09
---

# Recent Context

## Last Updated
2026-06-09. E2E test guide 补充 OutboxWorker/AGFS 说明。

## Key Recent Facts
- 2025 新范式 Sleep-Time Compute：Anthropic/OpenAI 主推 agent 在会话间隙做离线推理
- Sleep Rewrites Rules (Farooq 2024, Nature)：dream 不仅回放记忆，还抽象通用规则
- KG Hierarchical Abstraction：4层图谱渐进压缩（raw→episodic→semantic→conceptual）
- Deep Dream 改进路线图：P0 interleaved acquire + conflict resolution, P1 rule_discovery + deprecation + semantic_merge
- 新增 4 个 prompt-driven tool 建议：process_rule_discovery, process_conflict_resolution, process_deprecation, process_semantic_merge

## Recent Changes
- Updated: `wiki/modules/ogmemory-deep-dream-e2e-test-guide.md` — 补充 OutboxWorker 自动启动机制、AGFS 已弃用说明、compose account_id 常见问题、search_recall 验证方法
- Created: `wiki/modules/ogmemory-recalllogger-bug-retrospective.md`
- Created: `wiki/concepts/agent-dream-consolidation-landscape-2025.md`
- Created: `wiki/concepts/dream-modifying-existing-memories-storage-analysis.md`
- Updated: `wiki/index.md` — 新增两个概念页链接
- Updated: `wiki/concepts/_index.md` — 新增两行

## Active Threads
- RecallLogger content.md 修复已验证（compose 写入 16 行 → 32 行），logging fix (PR #28) 已回退不用 with_context
- oGMemory deep dreaming 验证方案设计（下一步）
- Deep Dream 多工具扩展设计（rule_discovery, conflict_resolution, deprecation, semantic_merge）
- Dream 修改已有记忆路径选择：短期 RelationEdge 降权 vs 中期 DEPRECATED 状态 vs 长期 MergePolicy 混合
- SchemaField vs field dict 统一重构（待决策）

## Key Architecture Notes
- OutboxWorker 在 server/app.py 进程内自动运行（SQL backend 用 NOTIFY/LISTEN，无需单独进程）
- AGFS 已弃用，当前仅使用 PostgreSQL (SQL backend)，`ogmem start --mode index` 不需要运行
- 启动命令需 `python -u` 禁用 stdout buffering + `conda activate py11`