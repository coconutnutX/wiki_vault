---
type: repo
title: "oG-Memory Operations Guide"
status: active
created: 2026-05-20
updated: 2026-05-20
tags: [ogmemory, operations, postgresql, startup, guide]
related:
  - "[[oG-Memory Extraction and Storage Analysis]]"
  - "[[PostgreSQL Installation Decision]]"
---

# oG-Memory Operations Guide

使用 PostgreSQL 16 作为 storage 和 vector_db 的运行指南。

---

## 配置概览

| 配置项 | 值 |
|--------|-----|
| HTTP 端口 | 8090 |
| Storage Backend | sql (PostgreSQL) |
| Vector DB | opengauss (PostgreSQL + pgvector) |
| 数据库 | ogmemory |
| 用户 | ogmem / ogmem123 |
| 配置文件 | `config/ogmem.yaml` |

### 配置示例

```yaml
storage:
  backend: sql
  connection_string: "host=127.0.0.1 port=5432 dbname=ogmemory user=ogmem password=ogmem123"
  pool_size: 5

vector_db:
  type: opengauss
  connection_string: "host=127.0.0.1 port=5432 dbname=ogmemory user=ogmem password=ogmem123"
  dimension: 1536
  table_name: vector_index
  pool_size: 5
```

---

## 启动服务

### 方式一：ogmem CLI（推荐）

```bash
cd /data/Workspace2/oG-Memory
ogmem start local --daemon
```

### 方式二：直接启动 Python

```bash
cd /data/Workspace2/oG-Memory
source ~/miniconda3/etc/profile.d/conda.sh && conda activate py11

# 前台运行
python server/app.py

# 后台运行
nohup python server/app.py > .ogmem_data/logs/context_engine.log 2>&1 &
echo $! > .ogmem_data/ce.pid
```

---

## 停止服务

```bash
# 方式一：ogmem CLI
cd /data/Workspace2/oG-Memory
ogmem stop local

# 方式二：手动停止
kill $(cat .ogmem_data/ce.pid) 2>/dev/null
rm .ogmem_data/ce.pid
```

---

## 验证服务

### 健康检查

```bash
curl http://127.0.0.1:8090/api/v1/health

# 预期返回：
# {"sql":"connected","status":"ok","storage_backend":"sql","llm":"gpt-4o-mini"}
```

### 测试 compose API

```bash
curl -X POST http://127.0.0.1:8090/api/v1/compose \
  -H "Content-Type: application/json" \
  -d '{"userId":"u-alice","sessionId":"test-session"}'
```

---

## 数据库管理

### 连接 PostgreSQL

```bash
PGPASSWORD=ogmem123 psql -h 127.0.0.1 -U ogmem -d ogmemory
```

### 常用查询

```sql
-- 查看表
\dt

-- 查看记忆节点
SELECT uri, category, context_type, created_at FROM context_nodes LIMIT 5;

-- 查看向量索引
SELECT * FROM vector_index LIMIT 5;

-- 查看会话归档
SELECT archive_id, session_id, created_at FROM session_archives LIMIT 5;
```

---

## 日志查看

```bash
tail -f /data/Workspace2/oG-Memory/.ogmem_data/logs/context_engine.log
```

---

## 数据库初始化

首次运行需要初始化表结构：

```bash
# 创建数据库和用户
sudo -u postgres psql -c "CREATE DATABASE ogmemory;"
sudo -u postgres psql -c "CREATE USER ogmem WITH PASSWORD 'ogmem123';"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE ogmemory TO ogmem;"

# 安装 pgvector 扩展
sudo apt install -y postgresql-16-pgvector
sudo -u postgres psql -d ogmemory -c "CREATE EXTENSION vector;"

# 初始化表结构
PGPASSWORD=ogmem123 psql -h 127.0.0.1 -U ogmem -d ogmemory \
  -f fs/sql_adapter/db_schema.sql
```

---

## API 端点一览

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/v1/health` | GET | 健康检查 |
| `/api/v1/compose` | POST | 组装记忆上下文 |
| `/api/v1/after_turn` | POST | 对话后处理（抽取记忆） |
| `/api/v1/ingest` | POST | 单条消息摄入 |
| `/api/v1/ingest_batch` | POST | 批量消息摄入 |
| `/api/v1/compact` | POST | 压缩对话历史 |
| `/api/v1/bootstrap` | POST | 初始化记忆 |
| `/api/v1/dispose` | POST | 清理记忆 |
| `/api/v1/sessions/<id>` | GET | 获取会话信息 |
| `/api/v1/sessions/<id>/messages` | POST | 添加消息 |
| `/api/v1/sessions/<id>/commit` | POST | 提交会话（抽取） |

---

## 注意事项

- **AGFS Go 服务端未编译**：当前使用 SQL storage backend，AGFS 未编译不影响运行
- **vector_db.type = opengauss**：虽然名称是 opengauss，但实际使用 psycopg2 连接 PostgreSQL，兼容原生 PostgreSQL
- **embedding dimension**：配置为 1536，需与 LLM embedding 模型匹配（text-embedding-ada-002）
- **RLS 已启用**：context_nodes、relation_edges、session_archives 表启用了 Row-Level Security，需设置 `app.account_id`