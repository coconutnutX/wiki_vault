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
<!-- Rows added during ingest -->
