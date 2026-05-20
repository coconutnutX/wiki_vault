---
type: meta
title: "Hot Cache"
updated: 2026-05-20T16:45:00
---

# Recent Context

## Last Updated
2026-05-20. Deep Dream 框架设计文档完成，准备进入实现阶段。

## Key Recent Facts
- **Deep Dream Framework**: 三阶段流水线（Acquire → Process → Output），策略可插拔可组合，新增 `dream` category
- **dream category**: URI 格式 `ctx://{account}/users/{user}/memories/dream/{dream_id}/`，与 ProvenanceResolver.dream source_type 对应
- **溯源机制**: provenance_ids 存储来源记忆 URI + 继承的溯源链
- **oG-Memory 运行状态**: HTTP 服务运行在 8090 端口，storage_backend=sql，vector_db=opengauss (PostgreSQL+pgvector)
- **PostgreSQL 配置**: 数据库 ogmemory，用户 ogmem/ogmem123，pgvector 扩展已启用
- Wiki vault: /mnt/c/Data/wiki-vault，workspace: /data/Workspace2

## Recent Changes
- Created: [[ogmemory-deep-dream-framework]] (wiki/modules/) — Deep Dream 框架完整设计文档
- Updated: [[Wiki Index]] — 添加 Deep Dream 模块条目

## Active Threads
- Deep Dream 框架设计完成，下一步：实现基础版本（RecentAcquire + StanfordReflection）
- 需修改文件：core/validation.py、core/uri_resolver.py、commit/merge_policies.py
- LightDream 集成：后续由其他同事开发，Deep Dream 作为下游消费者
