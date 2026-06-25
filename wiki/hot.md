---
type: meta
title: "Hot Cache"
updated: 2026-06-11
---

# Recent Context

## Last Updated
2026-06-24. 新 API key（`sk--hAux...` @ `113.46.219.251:8080`）重跑 MABench LongMemEval S*，踩 4 个新 blocker + embedder 死局 + 服务 hang→死。详见 [[memoryagentbench-lme-s-run-20260624]]。
2026-06-23. 整理 Dream/离线优化/测评 文献总结 + 补充三篇 2026 新基准（LongMemEval-V2、AgentMemoryBench、Mem2ActBench）。

## Key Recent Facts
- MABench LME-S 重跑环境：API key `sk--hAux...` @ `113.46.219.251:8080/v1`，仅 GLM-5/5.1/5.2/Qwen3.7-Plus（**无 gpt-4o、无 embedding 模型**），reader/oG-Memory/judge 全用 GLM-5
- 服务必须 `HF_HUB_OFFLINE=1` 启动（bge-m3 加载会连不通的 HF）；eval 也加（跳过 HF/GitHub 重试）
- **`ALTER ROLE ogmem WITH BYPASSRLS`**（一次性）—— 否则 every after_turn 被 `ensure_schema` 重新打开的 FORCE RLS 挡住 500。`NO FORCE RLS` 无效（每次被重开）
- eval reader 从 env 读 `OPENAI_API_KEY`/`OPENAI_BASE_URL`（`agent.py:111`），跑前必须 export
- 杀服务用 `fuser -k 8090/tcp`，别用 `pkill -f "server/app.py"`（会自杀 wrapper，exit 144）
- DeepDream E2E 验证完成：9/9 题成功，21 条 dream URI，全部 process_consolidation
- 发现并修复 provenance_ids 字符级拆分 bug（LLM 返回逗号分隔字符串而非数组），commit eedd9cd8
- 服务 cwd 必须是 oG-Memory 目录，否则 dream prompt-driven tool 不加载
- account_id = `longmemeval`，psql 查询前必须 SET app.account_id

## Recent Changes
- Created: `wiki/modules/ogmemory-deepdream-longmemeval-e2e-runbook.md` — 操作手册
- Created: `longmemeval/scripts/run_deepdream.py` — 按题维度调 deepdream 脚本
- Created: `longmemeval/configs/dream-smoke9-ingest.toml` — 注入专用配置
- Committed: fix provenance_ids string-split bug (eedd9cd8)
- Committed: feat: process_correction + process_consolidation (15269c63)

## Active Threads
- DeepDream LongMemEval QA 评测（before/after 对比，待跑）
- Dream 修改已有记忆路径选择（待决策）
- SchemaField vs field dict 统一重构（待决策）

## Key Architecture Notes
- 服务启动：`cd /data/Workspace2/oG-Memory && python -u server/app.py`（cwd 必须正确）
- dream 脚本：`run_deepdream.py --runner-output <dir> --data-file <json>`（读 run_config.json 获取 run_id）
- provenance_ids：LLM 可能返回逗号分隔字符串，已修复自动 split
- 按题维度隔离 user_id 是正确评测语义（不应共享 user_id）