---
created: 2026-05-22
status: implemented
tags:
  - og-memory
  - cli
  - sql-backend
---

# oG-Memory SQL-only Headless Start

## Problem

`ogmem start headless` 强制检查 AGFS binary 存在，即使配置使用 SQL backend：

```python
# Original code in cli/commands/start.py
if not (agfs_bin.exists() and os.access(agfs_bin, os.X_OK)):
    print(fail(f"AGFS binary not found: {agfs_bin}"))
    return 1
```

这导致 SQL-only 用户无法使用 `ogmem start` 命令。

## Solution

修改 `start_headless()` 函数，根据 `storage_backend` 配置决定是否需要 AGFS：

- `storage_backend == "agfs"` → 启动 AGFS + ContextEngine
- `storage_backend == "sql"` → 只启动 ContextEngine，跳过 AGFS

## Implementation

**File**: [cli/commands/start.py](https://github.com/anthropics/oG-Memory/blob/main/cli/commands/start.py)

**Key changes**:

1. 在函数开始时加载配置获取 `storage_backend`
2. AGFS 启动逻辑包装在 `if storage_backend == "agfs":` 条件中
3. SQL mode 打印 "Storage backend: sql (AGFS skipped)" 提示
4. shutdown 函数也条件化处理 AGFS 停止

```python
# Load config to check storage_backend
try:
    from providers.unified_config import get_config
    cfg = get_config()
    storage_backend = getattr(cfg, 'storage_backend', 'agfs')
except Exception:
    storage_backend = env.get("STORAGE_BACKEND", "agfs")

agfs_started = False
if storage_backend == "agfs":
    # AGFS mode: require and start AGFS binary
    ...
else:
    # SQL mode: skip AGFS entirely
    print(f"  Storage backend: {storage_backend} (AGFS skipped)")
```

## Usage

```bash
# SQL-only mode (from ogmem.yaml with storage_backend: sql)
ogmem start headless -d

# Output:
#   Storage backend: sql (AGFS skipped)
#   ✓ ContextEngine already running on :8090
#   ContextEngine   http://127.0.0.1:8090  UP
```

## Related

- [[Promptfoo Extraction Testing Framework]]
- [[oG-Memory ReactLoop Abstract Base Class]]