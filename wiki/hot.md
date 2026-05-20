---
type: meta
title: "Hot Cache"
updated: 2026-05-20T09:40:00
---

# Recent Context

## Last Updated
2026-05-20. oG-Memory 服务已启动，使用 PostgreSQL 16 作为 storage 和 vector_db。

## Key Recent Facts
- **oG-Memory 运行状态**: HTTP 服务运行在 8090 端口，storage_backend=sql，vector_db=opengauss (PostgreSQL+pgvector)
- **PostgreSQL 配置**: 数据库 ogmemory，用户 ogmem/ogmem123，pgvector 扩展已启用
- **启动命令**: `ogmem start local --daemon` 或 `python server/app.py`
- **健康检查**: `curl http://127.0.0.1:8090/api/v1/health` → {"status":"ok","sql":"connected"}
- **PostgreSQL 16**: apt 安装 v16.14，数据目录 `/var/lib/postgresql/16/main`，systemd 自动管理
- **ReactLoop 抽象基类**: Template Method 模式，子类实现 4 抽象方法 + 2 可选 override
- Wiki vault: /mnt/c/Data/wiki-vault，workspace: /data/Workspace2

## Recent Changes
- Created: [[oG-Memory Operations Guide]] (wiki/repos/) — 运行指南：启动/停止/验证命令
- Created: [[PostgreSQL Installation Decision]] (wiki/decisions/) — 从 OpenGauss Docker 切换到 PostgreSQL

## Active Threads
- oG-Memory ReactLoop 泛化完成，ToolDef 注册模式延后讨论
- 记忆溯源与 oG-Memory evidence_quote 字段关联
