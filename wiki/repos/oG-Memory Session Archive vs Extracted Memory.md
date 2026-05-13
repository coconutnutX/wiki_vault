---
type: repo
status: active
created: 2026-05-13
updated: 2026-05-13
tags: [ogmemory, session-archive, extraction, dedup, data-relationship]
verified: false
---

# oG-Memory Session Archive 与抽取记忆的关系

> 结论：串行执行，数据独立，仅通过 extraction_summary 做去重标记。
>
> 相关文档：[[oG-Memory Extraction and Storage Analysis]]、[[oG-Memory Extraction and Storage Example]]

## 1. 执行顺序（串行，不是并行）

`after_turn` (`memory_service.py:1666-1742`) 调用 `_background_extract_write()`，依次执行：

```
① write_api.commit_session()  → 抽取记忆 → context_nodes 表
② buf.extraction_summary      = _update_extraction_summary() → 记录已抽取内容
③ self._async_drain()          → 嵌入向量
④ mgr.commit_snapshot()        → Session Archive → session_archives 表
```

关键代码 (`memory_service.py:1707-1718`):

```python
# 抽取完成后，更新 extraction_summary
buf.extraction_summary = _update_extraction_summary(buf.extraction_summary, write_result)
# 嵌入向量
self._async_drain()
# 最后才做 Session Archive
commit_result = mgr.commit_snapshot(session_id, archive_snapshot, ctx, wait=True)
```

## 2. 数据关联（弱关联：仅用于去重）

`extraction_summary` (`memory_service.py:163-198`):

```python
def _update_extraction_summary(existing_summary, write_result, max_chars=2000):
    plans = write_result.get("plans", [])
    for plan in plans:
        if plan.get("action") != "skip":
            uri = plan.get("target_uri", "")
            # 例如: ctx://acme/profile/alice-tech → "[profile] alice-tech"
            category = parts[-2]  # profile
            slug = parts[-1]      # alice-tech
            new_lines.append(f"[{category}] {slug}")

    combined = existing_summary + "\n" + new_lines
    return combined[-max_chars:]  # 保留最新的 2000 chars
```

下轮抽取时 (`extraction/tools.py:305-310`):

```python
summary_block = ""
if session_summary:
    summary_block = (
        "Previously Extracted Context "
        "(DO NOT re-identify these — they are already saved):\n"
        f"{session_summary}\n\n"
    )
```

注入到 Phase 1 Prompt:

```
Previously Extracted Context (DO NOT re-identify these — they are already saved):
[profile] alice-tech
[entity] london-uk
[event] alice-intro-20240115
```

## 3. 两个系统对比

| 维度 | 抽取记忆 (context_nodes) | Session Archive (session_archives) |
|------|--------------------------|-------------------------------------|
| 提取内容 | 结构化知识（profile/entity/event） | 对话历史压缩 |
| 存储位置 | context_nodes 表 | session_archives 表 |
| 检索方式 | 向量检索 + URI 定位 | session_id 索引 |
| 注入位置 | working set → user message | _collect_archives → overview/abstract |
| Prompt | YAML Schema + 抽取 Prompt | SessionCompressor 压缩 Prompt |
| 关联机制 | extraction_summary → 去重标记 | 无反向引用 |

## 4. 核心发现

二者确实没有数据关联：

1. Session Archive 的 overview/abstract **不引用**抽取记忆中的实体/事件
2. 抽取记忆的 content **不引用** Session Archive 的 overview
3. 唯一关联：`extraction_summary` 用于去重（告诉下一轮抽取不要重复识别已抽取内容）

这是两个独立系统：
- **抽取记忆**：结构化知识图谱（向量检索）
- **Session Archive**：对话历史压缩（token 预算控制）
