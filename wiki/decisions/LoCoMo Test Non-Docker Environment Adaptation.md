---
type: decision
status: active
date: 2026-05-22
context: "LoCoMo 测试脚本原设计假设 ogmem 在 docker 容器中运行，通过 docker logs 检查后台提取进度"
decision: "修改 locomo_test 代码支持从日志文件读取，适配本地进程运行的 ogmem"
alternatives:
  - 在 docker 中运行 ogmem
  - 直接跳过等待逻辑
consequences:
  - 可在非 docker 环境运行 locomo 测试
  - 需额外配置 log_file 参数
tags: [decision, adr, locomo, testing, ogmem]
created: 2026-05-22
updated: 2026-05-22
---

# LoCoMo Test Non-Docker Environment Adaptation

## Context

同事的 locomo_test 脚本设计用于测试 OpenClaw + oG-Memory 的记忆功能。原设计假设：
- ogmem 在 docker 容器中运行
- 通过 `docker logs` 检查 `after_turn background extract done` 日志标记来等待后台提取完成

但我的环境：
- ogmem 作为本地 Python 进程直接运行
- 日志写入文件 `/data/Workspace2/oG-Memory/.ogmem_data/logs/context_engine.log`

## Decision

修改 locomo_test 代码，添加 `log_file` 配置支持非 docker 环境：

### 代码修改

**config.py** - 添加 `log_file` 字段：
```python
@dataclass
class OgmemEnv:
    ...
    log_file: str = ""  # optional: log file path for non-docker setups
```

**eval.py** - 修改日志读取逻辑：
```python
OGMEM_EXTRACT_LOG_MARKER = "after_turn background extract done"
OGMEM_EXTRACT_LOG_MARKER_ALT = "after_turn background snapshot commit done"

def count_ogmem_after_turn_extract_logs(..., log_file: str | None = None) -> int:
    if log_file:
        # Read from log file instead of docker logs
        with open(log_file, "r") as f:
            lines = f.readlines()
        ...
```

### 配置示例

```toml
[ogmem]
api_url = "http://127.0.0.1:8090"
docker_container = ""  # 空值表示不使用 docker
log_file = "/data/Workspace2/oG-Memory/.ogmem_data/logs/context_engine.log"
```

## Other Fixes in This Session

### OpenClaw 插件加载问题

**问题**：`og-memory-context-engine` 插件通过符号链接安装，但 openclaw 的 `discoverInDirectory` 使用 `Dirent.isDirectory()` 对符号链接返回 `false`，导致插件无法被发现。

**解决**：删除符号链接，直接复制插件文件到 `~/.openclaw/extensions/og-memory-context-engine/`

### package.json hooks 配置问题

**问题**：插件 `package.json` 包含 `"hooks": ["./index.js"]`，openclaw 将其当作 hook pack 处理，要求 HOOK.md 文件。

**解决**：移除 `hooks` 配置，只保留 `extensions`。

### Responses API 端点

**问题**：Gateway 默认不启用 `/v1/responses` 端点，返回 404。

**解决**：
```bash
openclaw config set gateway.http.endpoints.responses.enabled true
systemctl --user restart openclaw-gateway
```

## Alternatives Considered

1. **在 docker 中运行 ogmem** - 需要额外 docker 配置，与现有环境不符
2. **跳过等待逻辑** - 可能导致 QA 在 ingest 未完成时开始，测试结果不准确

## Consequences

- ✅ 可在非 docker 环境运行 locomo 测试
- ✅ 测试流程完整：health_check → ingest → qa → judge → stats
- ⚠️ 需要额外配置 `log_file` 参数
- ⚠️ 修改了 locomo_test 代码，与同事版本有差异

## Test Result

首次 small 测试成功：
- health_check: 1.2s
- ingest: 265.3s (4 sessions, LLM tokens=158,741)
- qa: 551.7s (35 questions)

## Links

- [[oG-Memory Operations Guide]] — ogmem 运行指南
- [[OpenClaw Memory System Overview]] — 系统架构