---
type: meta
title: "Hot Cache"
updated: 2026-05-26
---

# Recent Context

## Last Updated
2026-05-26. Dreaming/Consolidation 验证调研完成，已存入 wiki。

## Key Recent Facts
- **验证方法只有5类**：消融、迭代曲线、持续学习基准、问答基准、执行测试——没有任何工作专门设计基准测 consolidation 本身效果
- **最接近 dreaming前后对比**：OpenDream 两遍域匹配（+4pp aggregate, +20pp 3个任务, 0 regression）、ScallopBot LoCoMo（+26% F1, +25% EM）
- **LongMemEval 关键局限**：假设记忆一次性注入，不测逐步经历→逐步记忆→逐步 dreaming 的过程
- **测 LongMemEval on ogmemory 需要写6个脚本**：数据格式转换(P0)、注入(P0)、QA+compose(P0)、dreaming验证(P0)、检索命中(P1)、判题(复用)

## Recent Changes
- Created: `wiki/concepts/dreaming-consolidation-validation-survey.md`
- Updated: `wiki/index.md` — 新增条目

## Active Threads
- oGMemory deep dreaming 验证方案设计（下一步）
- LongMemEval 数据格式适配脚本（待写）
- ScallopBot 与 OpenDream 评估代码详解已完成