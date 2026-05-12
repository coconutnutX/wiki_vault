---
type: concept
title: "OpenClaw Memory System"
status: mature
domain: architecture
tags: [memory, agent, obsidian, openclaw, pattern]
created: 2026-05-11
updated: 2026-05-11
related:
  - "[[OpenClaw MEMORY.md Implementation]]"
  - "[[OpenClaw Memory System Overview]]"
  - "[[OpenClaw Memory Architecture Analysis]]"
  - "[[OpenClaw Memory Research Corrections]]"
---

# OpenClaw Memory System

OpenClaw 项目的跨会话记忆系统实现机制，基于纯 Markdown 文件驱动，无隐藏状态。

## Core Pattern: File-Driven Memory

核心设计：**纯 Markdown 文件驱动，无隐藏状态**。模型只"记住"写入磁盘的内容。

```
用户对话上下文
       │
       ▼ (接近 compaction 阈值)
  Memory Flush ──写入──► memory/YYYY-MM-DD.md（每日笔记，append-only）
       │
       ▼ (dreaming 后台定期扫描)
  Dreaming Phases (Light → REM → Deep)
       │
       ▼ (通过分数/频率/多样性门槛)
  Promotion ──追加──► MEMORY.md（长期记忆，append-only）
       │
       ▼ (新会话启动)
  Bootstrap 加载 ◄──读取── MEMORY.md（注入系统 prompt，带截断）
  Startup Context ◄──读取── memory/YYYY-MM-DD.md（注入 session prelude）
```

## Three-Layer Memory Architecture

| 层 | 文件 | 用途 | 生命周期 |
|----|------|------|---------|
| 长期记忆 | `MEMORY.md` | 持久事实、偏好、决策 | 永久，append-only |
| 每日笔记 | `memory/YYYY-MM-DD.md` | 运行时上下文和观察 | 自动加载今天+昨天 |
| 梦境日记 | `DREAMS.md` | 后台整理摘要，供人工审查 | 可选 |

## When to Write

### 1. Memory Flush（预压缩刷写）
- 触发：上下文接近 compaction 阈值（默认 4000 tokens）或 transcript 达 2MB
- **只写入每日笔记**，不直接写 MEMORY.md
- 安全约束：`MEMORY.md` 被视为只读

### 2. Dreaming Promotion（短期 → 长期晋升）
- **唯一写入 MEMORY.md 的路径**
- 筛选条件：score ≥ 0.75, recallCount ≥ 3, uniqueQueries ≥ 2
- 纯追加，带 HTML 注释标记去重
- 使用文件锁防并发

### 3. CLI 手动触发
- `openclaw memory promote --apply`

## When to Read

| 场景 | 机制 | 截断策略 |
|------|------|---------|
| 会话启动 | Bootstrap 加载（8 个文件，MEMORY.md 排最后） | 75% head + 25% tail |
| 新会话 prelude | Startup Context 加载每日笔记 | 二分搜索裁剪 |
| 工具调用 | `memory_search`（语义搜索）/ `memory_get`（精确读取） | 可配置 |

## Truncation Strategy

```
单文件上限: 12,000 chars (bootstrap) / 1,200 chars (startup context)
总量上限:   60,000 chars (bootstrap) / 2,800 chars (startup context)
截断标记:   [...truncated, read MEMORY.md for full content...]
```

算法要点：迭代 3 次确定 head/tail 各保留多少字符，空间极度紧张时回退紧凑格式。

## Key Design Principles

1. **MEMORY.md 永远不被覆盖或删减** — 只有读取（带截断）和追加（经晋升）
2. **运行时写入先进每日笔记** — 经严格筛选后才能晋升到长期记忆
3. **大小写敏感的文件解析** — 优先 `MEMORY.md`，兼容 `memory.md`，排除符号链接
4. **缓存优化** — 以 inode + dev + size + mtimeMs 作为文件身份标识
5. **边界安全** — 通过 `openRootFile` 限制最大读取 2MB，防止路径穿越

## Source

- Repository: `github.com/openclaw/openclaw`
- Key files:
  - `src/memory/root-memory-files.ts` — 文件定位与解析
  - `src/agents/workspace.ts` — Bootstrap 加载
  - `src/agents/pi-embedded-helpers/bootstrap.ts` — 截断算法
  - `src/auto-reply/reply/startup-context.ts` — 每日笔记加载
  - `extensions/memory-core/src/flush-plan.ts` — 预压缩刷写
  - `extensions/memory-core/src/short-term-promotion.ts` — Dreaming 晋升
  - `extensions/memory-core/src/tools.ts` — 搜索与读取工具
