---
name: ogmemory-recalllogger-bug-retrospective
description: RecallLogger search_recall 表写入失败的 bug 复盘：content.md 过滤是根因，logging formatter 是独立问题
metadata:
  type: project
  created: 2026-06-09
---

# RecallLogger Bug 复盘：search_recall 表永远为空

> 日期：2026-06-09 | 相关 commits: `dedcf097`, `58051fba` | 相关 Issue: [#15](https://gitcode.com/akushonkamen/oG-Memory/issues/15) PR: [#28](https://gitcode.com/akushonkamen/oG-Memory/pull/28)

## 1. 问题现象

新增 RecallLogger 功能后，调用 `compose` / `search_memory` API 时，日志显示 `_recall_logger` 存在且 `hits=15`，但 PostgreSQL 的 `search_recall` 表始终为空（0 行）。没有任何异常抛出，也没有 warning 日志。

## 2. 调查过程

### 2.1 初步排除

| 检查项 | 结果 |
|--------|------|
| `_recall_logger` 是否注入成功 | ✅ `RecallLogger ready`，对象地址正常 |
| `result.hits` 是否有值 | ✅ hits=15 |
| `search_recall` 表是否存在 | ✅ 已创建，RLS 正常 |
| `conn.commit()` 是否执行 | ❓ 无法确认——日志中无 `dream.recall_logger` 的任何输出 |
| 异常是否被吞掉 | ❓ api.py 的 `except Exception` 未触发（无 warning 日志） |

### 2.2 日志消失之谜

关键现象：`service.api` 的 `logger.info("search_recall check: ...")` 正常输出，但 `dream.recall_logger` 内部所有 `logger.info()` / `logger.debug()` **完全没有出现**，也没有 `search_recall logging failed` 的 warning。

**排查思路 1：logging formatter bug（Issue #15）**

怀疑 `setup_logging()` 的 formatter 包含 `%(account_id)s` / `%(trace_id)s`，而 `dream.recall_logger` 未用 `with_context(logger, ctx)` → formatter 抛 `ValueError` → Python logging 静默吞掉。

**排除**：生产服务器 (`server/app.py`) 用 `logging.basicConfig()`，格式不含 context 字段，不会触发此 bug。在服务器日志中也**没有**找到 `--- Logging error ---` traceback。

**排查思路 2：Python stdout 缓冲**

怀疑 print/logger 输出被缓冲未 flush。用 `python -u`（unbuffered 模式）重启服务器后，`service.api` 的 print 正常出现，但 `dream.recall_logger` 的 print 仍未出现。

**排查思路 3：模块缓存**

怀疑 `.pyc` 缓存导致旧代码被加载。删除所有 `__pycache__` 后重启，问题依旧。

### 2.3 破局点：在 api.py 中加 BEFORE/AFTER print

```python
print(f"BEFORE log_recalls call, method={self._recall_logger.log_recalls}")
self._recall_logger.log_recalls(result.hits, query, ctx)
print(f"AFTER log_recalls call")
```

结果：**BEFORE 和 AFTER 都出现了**——`log_recalls` 被调用且正常返回（无异常），但内部代码的输出完全消失。

这证明 `log_recalls` **确实被调用了**，只是方法内部的逻辑没有执行到日志输出那一步。

### 2.4 最终定位：content.md 过滤

通过 debug endpoint 直接调用 `log_recalls` 并打印 hits 详情：

```json
{
  "hit_details": [
    {"uri": "ctx://acct-demo/.../events/xxx/content.md", "ends_with_content_md": true},
    {"uri": "ctx://acct-demo/.../events/yyy/content.md", "ends_with_content_md": true},
    {"uri": "ctx://acct-demo/.../session_summaries/zzz/content.md", "ends_with_content_md": true}
  ]
}
```

**所有 5 个 hits 的 URI 都以 `/content.md` 结尾！**

而 `log_recalls` 的过滤逻辑：

```python
rows = [
    (h.uri, ...)
    for h in hits
    if not h.uri.endswith("/content.md")  # ← 全部被过滤掉
]
if not rows:
    return  # ← 直接返回，不执行任何 INSERT 或日志
```

`rows` 为空 → `log_recalls` 在 `if not rows: return` 处**提前返回** → INSERT 代码根本没执行到 → DB 为空。

手动构造不带 `/content.md` 的 `RetrievedBlock` 直接调用 `log_recalls` → 成功写入 DB。**确认这就是根因。**

## 3. 根因分析

### 3.1 为什么所有 hits 都是 content.md 子节点？

向量索引存储的是 L2 node 的 `content.md` 子文件（实际文本 chunk），而非 L2 父节点本身。检索管线返回的 `RetrievedBlock` URI 全部指向这些 chunk 文件。

```
L2 父节点（实际记忆）：ctx://acct/users/alice/memories/events/launch
L2 子节点（向量 chunk）：ctx://acct/users/alice/memories/events/launch/content.md
```

### 3.2 过滤逻辑的设计意图与实际效果

| 设计意图 | 实际效果 |
|----------|----------|
| 过滤掉 content.md 子节点，只记录"可读的记忆节点" | 因为检索管线只返回 content.md 子节点，**所有 hits 被丢弃**，等于完全无效 |

过滤的意图是合理的（content.md 是索引 chunk，不是人类可读的记忆），但**假设了 hits 中包含 L2 父节点 URI**——这个假设不成立。

### 3.3 为什么日志也消失了？

committed 版本的 `log_recalls` 中，唯一的日志是 `logger.debug("wrote %d rows...")`，位于 INSERT **之后**。由于 `rows` 为空导致提前 `return`，这行 `logger.debug()` 永远不会执行。

再加上 `basicConfig(level=INFO)` 会过滤掉 `logger.debug()`——即使有非 content.md 的 hits，debug 日志也不会出现在生产输出中。

## 4. 两个独立 Bug 的澄清

本次 debug 涉及两个独立问题，容易混淆：

| Bug | 严重性 | 触发条件 | 影响 | 修复 |
|-----|--------|----------|------|------|
| **content.md 过滤 → DB 写入失败** | 🔴 严重 | 生产环境（所有 search 请求） | search_recall 表永远为空 | `removesuffix("/content.md")` 映射到父 URI |
| **formatter 缺字段 → log 静默丢失** | 🟡 中等 | 仅 `setup_logging()` 环境 | debug/warning 日志消失 | `DefaultContextFilter` 兜底 + `with_context` |

**formatter bug 不是 DB 写入失败的原因**。生产环境不会触发 formatter bug。两者只是恰好都导致"日志消失"这一相同表象，容易误判为同一问题。

## 5. 修复方案

### 5.1 核心修复：content.md URI 映射（recall_logger.py）

```python
# 修复前：丢弃 content.md hits（等于丢弃所有 hits）
rows = [
    (h.uri, ...)
    for h in hits
    if not h.uri.endswith("/content.md")  # 全被过滤
]

# 修复后：映射 content.md 到父 L2 URI
rows = [
    (
        h.uri.removesuffix("/content.md") if h.uri.endswith("/content.md") else h.uri,
        ctx.account_id,
        h.owner_space or ctx.user_space_name(),
        query,
        h.score,
        h.category or "",
    )
    for h in hits  # 不再过滤
]
```

### 5.2 辅助修复：日志可观测性

| 修改 | 原因 |
|------|------|
| `logger.debug` → `logger.info` | INFO 级别才能在生产日志中可见 |
| 新增 `logger.info("log_recalls: inserting %d rows...")` 在 INSERT 前 | 即使 INSERT 失败也能看到"走到这一步" |
| warning 用 `with_context(logger, ctx)` | 防止 `setup_logging()` 环境下 formatter 报错 |
| `DefaultContextFilter`（PR #28） | 兜底防止 formatter `ValueError` 导致日志静默丢失 |

### 5.3 测试更新

`test_skips_content_md_subnodes` → `test_maps_content_md_to_parent_uri`：验证 `/content.md` URI 被映射到父 URI 而非被丢弃。

## 6. 教训与反思

### 6.1 测试覆盖 vs 实际数据形状

单元测试用手工构造的 `RetrievedBlock`（URI 不带 `/content.md`），验证了 INSERT 逻辑正确。但**没有测试检索管线返回的真实 hits 数据形状**。测试通过 ≠ 生产可用，因为测试数据不代表实际数据。

**建议**：对涉及外部系统输出（向量索引、LLM 返回）的过滤逻辑，应增加"真实数据形状"的集成测试或 mock 测试——至少模拟 hits 全部是 `/content.md` 子节点的场景。

### 6.2 隐式假设要显式验证

`log_recalls` 隐式假设了 "hits 中包含 L2 父节点 URI"，但从未验证。当假设不成立时，代码不会报错——只是静默地什么都不做。

**建议**：对过滤后结果为空的情况，至少输出一条 warning 或 info 日志（如 `"log_recalls: 0 rows after filtering, %d hits were content.md sub-nodes"`），而非静默 `return`。静默跳过是 debug 最难的故障模式。

### 6.3 logging 系统的两套配置风险

项目存在两套 logging 配置（设计的 `setup_logging()` vs 实际的 `basicConfig()`），导致：

- 在测试环境遇到的 formatter bug 在生产中不出现
- 以为"修了 logging 就修了 DB 写入"，实际是两个独立问题

**建议**：统一 logging 配置入口，或在 `setup_logging()` 中加入 `DefaultContextFilter` 兜底（PR #28 已做），消除 formatter 静默失败的风险。

### 6.4 Debug 策略反思

| 步骤 | 耗时 | 是否有效 |
|------|------|----------|
| 检查 RLS、连接池、SQL 语法 | 中 | ❌ 排除了这些方向但非根因 |
| 怀疑 logging formatter bug | 长 | ❌ 以为是根因，实际是独立问题 |
| 在 api.py 加 BEFORE/AFTER print | 短 | ✅ **破局点**——证明 log_recalls 被调用但内部不执行 |
| 打印 hits 的 URI 详情 | 短 | ✅ **一击命中**——所有 URI 都是 content.md |

**关键教训**：当函数"被调用了但没效果"时，最快的方法是在调用前后加 print，然后**检查传入参数的实际值**，而非先怀疑基础设施（连接池、RLS、logging）。数据驱动的 debug 优先于机制驱动的 debug。

---

**相关页面**：[[ogmemory-deep-dream-framework]] | [[oG-Memory Operations Guide]] | [[PostgreSQL RLS Tenant Isolation]]