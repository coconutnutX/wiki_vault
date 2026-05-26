---
type: meta
title: "Hot Cache"
updated: 2026-05-26T00:00:00
---

# Recent Context

## Last Updated
2026-05-26. oG-Memory ReactLoop 安全检查机制修复完成。

## Key Recent Facts
- **Safety check bypass 修复**: extract_* tool call 产出的 candidates 原来绕过 `_check_unread_existing_files()`，现在两条路径(content + tool-call)统一走 `_safety_check_candidates()`
- **Invalid response 处理**: extraction 场景下 disable_tools 继续迭代无意义，改为 `_on_invalid_response()` 返回 True 直接结束 loop
- **基类新增 2 个 virtual method**: `_post_tool_call_safety_check()` + `_on_invalid_response()`，extraction 子类覆盖
- **dev_0520 分支**: rebase 完成 + InternalToolUsageTracker 集成 + pipeline_name rename + safety check fix，共4个新 commit 未 push

## Recent Changes
- Modified: `core/react_loop.py` — 新增 `_post_tool_call_safety_check()` 和 `_on_invalid_response()` virtual methods
- Modified: `extraction/extraction_react_loop.py` — `_safety_check_candidates()` 统一方法 + `_on_invalid_response()` 覆盖
- Modified: `tests/unit/extraction/test_react_loop.py` — 测试更新为期望 early loop termination
- Committed: `1b5690c3` fix: unify safety check for extract_* tool calls and end loop on invalid response
- Committed: `23915b41` fix: add missing prompt_manager arg to integration test

## Active Threads
- SUMMARY_PROMPT 优化 (summary_generator.py) — 下一步最有提升空间的方向
- dev_0520 PR 描述待更新（新增 safety check fix commit）