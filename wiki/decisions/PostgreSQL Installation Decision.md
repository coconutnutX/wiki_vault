---
type: decision
title: "PostgreSQL Installation Decision"
status: implemented
created: 2026-05-20
updated: 2026-05-20
tags: [decision, database, postgresql, opengauss, installation]
related:
  - "[[Workspace Environment]]"
---

# PostgreSQL Installation Decision

## Decision
放弃 OpenGauss Docker 方案，改用 apt 直接安装 PostgreSQL 16 作为本地数据库。

## Why
OpenGauss Docker 方案存在多个问题：
- **容器管理麻烦**：需要手动启动/停止 Docker 容器
- **表查看不便**：需进入容器或使用特殊工具
- **权限混乱**：容器内用户映射导致数据目录权限问题
- **数据目录散落**：多次挂载产生冗余目录（已清理）
- **生态工具受限**：兼容性不如原生 PostgreSQL

## How to Apply
使用 PostgreSQL 16 作为 oG-Memory 项目的主要数据库：
- 数据存储、向量索引、关系存储均可使用
- 直接 `psql` 连接，无需 Docker
- 服务自动启动，systemd 管理

---

## Installation Details

### 安装命令
```bash
# 添加 PostgreSQL APT 源
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

# 导入签名密钥（避免 apt-key deprecated）
wget -qO /tmp/postgresql_key.asc https://www.postgresql.org/media/keys/ACCC4CF8.asc
gpg --dearmor -o /tmp/postgresql.gpg /tmp/postgresql_key.asc
sudo mv /tmp/postgresql.gpg /etc/apt/trusted.gpg.d/postgresql.gpg

# 安装
sudo apt update
sudo apt install -y postgresql-16
```

### 安装结果
- **版本**: PostgreSQL 16.14
- **数据目录**: `/var/lib/postgresql/16/main`
- **配置文件**: `/etc/postgresql/16/main/postgresql.conf`
- **自动启动**: systemd 管理，开机自启

### 常用操作
```bash
# 连接数据库
sudo -u postgres psql

# 创建用户和数据库
sudo -u postgres createuser aaa
sudo -u postgres createdb ogmemory

# 服务管理
sudo service postgresql start/stop/restart/status
```

---

## Cleanup History

### AGFS 数据目录（已清理）
- `/data/Workspace2/.agfs_data` ✅ 已删除
- `/data/Workspace2/.ogmem_data/agfs` ✅ 已删除
- `/data/Workspace2/oG-Memory/.ogmem_data/agfs` ✅ 已删除

### OpenGauss 数据目录（待 sudo 删除）
- `/data/opengauss` ⏳ 需要 sudo
- `/data/opengauss_data` ⏳ 需要 sudo

运行命令：
```bash
sudo rm -rf /data/opengauss /data/opengauss_data
```

---

## AGFS Status (Separate from Database)
- **Python SDK (pyagfs)**: 已安装在 conda py11 环境 (`/home/aaa/miniconda3/envs/py11`)
- **Go 服务端**: 源码在 `/data/Workspace2/oG-Memory/agfs/`，**未编译**
- **配置文件**: `/home/aaa/.agfs-config.yaml` (端口 1833)

AGFS 是文件系统服务，与数据库选择无关，仍需编译 Go 服务端才能使用。