---
type: index
folder: modules
updated: 2026-05-27
---

# Modules Index

One note per major module / package / service.

| Module | Path | Status | Language | Purpose |
|--------|------|--------|----------|---------|
| Lazy Mode Empty DB Fix | extraction/extraction_react_loop.py | fixed | Python | 修复 Lazy 模式空数据库场景无法提取的 bug |
| Lazy Mode Extraction Issues | extraction/extraction_react_loop.py | draft | Python | Lazy 模式类型判断、routing key、重复检测、冲突处理问题分析与优化方案 |
| @operational_tool Framework | core/operational_tool.py | approved | Python | 轻量装饰器框架：函数签名→自动schema、runtime参数过滤、dict dispatch、loop级工具集配置 |
| RecallLogger Bug 复盘 | dream/recall_logger.py | fixed | Python | content.md 过滤导致 DB 写入失败 + logging formatter 静默丢失，两个独立 bug |
| DeepDream LongMemEval 测试计划 | scripts/ingest_dream_data.py | draft | Python | 用 LongMemEval 数据验证 deepdream 流程（注入→dream→验证） |
| DeepDream LongMemEval 操作手册 | — | verified | — | 端到端操作手册：环境配置、启动、注入、dream、SQL 查看、踩坑总结 |
| Dream36 重测进度 0622 | — | in-progress | — | doubao提取+gpt-4o-mini compose/dream + GLM-5 reader/judge 重测，schema migration阻塞 |
| MABench LME-S 2026-06-24 运行 | — | in-progress | — | 新 API key 重跑：4 个新 blocker（embedder 连HF hang / RLS FORCE 每次重开 / 残留eval占worker / nltk无条件下载hang）+ embedder 死局 + 服务hang→死，含已验证修复与实用命令 |
<!-- Rows added during ingest -->
