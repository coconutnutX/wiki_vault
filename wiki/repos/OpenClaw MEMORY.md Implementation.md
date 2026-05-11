---
type: source
title: "OpenClaw MEMORY.md Implementation"
status: active
tags: [openclaw, memory, implementation, code-analysis]
created: 2026-05-11
updated: 2026-05-11
related:
  - "[[OpenClaw Memory System]]"
---

# OpenClaw MEMORY.md Implementation

OpenClaw 项目的 MEMORY.md 具体代码实现分析。本文档侧重源码级细节，与抽象设计模式 [[OpenClaw Memory System]] 互为补充。

## 仓库信息

- **Repo**: `github.com/openclaw/openclaw`
- **本地路径**: `/data/Workspace2/repos/openclaw`

## 一、文件定位与解析

**关键文件**: `src/memory/root-memory-files.ts`

- 常量 `CANONICAL_ROOT_MEMORY_FILENAME = "MEMORY.md"` (第 4 行)
- 解析时**大小写敏感**，优先 `MEMORY.md`，兼容旧版 `memory.md`
- **排除符号链接**（防止循环引用）
- 文件路径：`{workspaceDir}/MEMORY.md`，workspace 默认在 `~/.openclaw/workspace/`

## 二、写入机制（三个触发点）

### 1. Memory Flush（预压缩刷写）

**关键文件**: `extensions/memory-core/src/flush-plan.ts`

当上下文接近 compaction 阈值时，系统静默插入一轮 agent turn，提醒将重要上下文写入磁盘。

**关键约束**：flush **只写入 `memory/YYYY-MM-DD.md`（每日笔记）**，不直接写 MEMORY.md。flush prompt 明确要求：

> "Treat MEMORY.md, DREAMS.md, SOUL.md as read-only during this flush; never overwrite them."

触发条件（`flush-plan.ts:110-113`）：
- `softThresholdTokens`: 距 compaction 阈值 **4000 tokens**（默认）
- `forceFlushTranscriptBytes`: transcript 达到 **2MB**（默认）

### 2. Dreaming 晋升（短期 → 长期）

**关键文件**: `extensions/memory-core/src/short-term-promotion.ts:1551-1686`

这是**唯一直接写入 MEMORY.md 的路径**，通过 `applyShortTermPromotions` 函数实现。

筛选条件：
- `score >= 0.75`（默认）
- `recallCount >= 3`（被 recall 的次数）
- `uniqueQueries >= 2`（来自不同查询的数量）
- 不能已被晋升过
- 不能被标记为 contaminated dreaming snippet

写入逻辑（第 1641-1649 行）：

```typescript
const header = existingMemory.trim().length > 0 ? "" : "# Long-Term Memory\n\n";
const section = buildPromotionSection(toAppend, nowMs, timezone);
await fs.writeFile(memoryPath, `${header}${withTrailingNewline(existingMemory)}${section}`, "utf-8");
```

特点：
- **纯追加**，不覆盖已有内容
- 首次写入时添加 `# Long-Term Memory` 标题
- 每个晋升条目带 `<!-- openclaw-memory-promotion:xxx -->` HTML 注释标记用于去重
- 使用文件锁 `withShortTermLock` 防并发

### 3. CLI 手动晋升

通过 `openclaw memory promote --apply` 手动触发，最终也走 `applyShortTermPromotions`。

## 三、读取机制（两个路径）

### 1. Bootstrap 加载（每次会话启动）

**关键文件**: `src/agents/workspace.ts:615-680`

`loadWorkspaceBootstrapFiles()` 按固定顺序加载 8 个 bootstrap 文件，MEMORY.md 排第 8（最后）：

```
AGENTS.md → SOUL.md → TOOLS.md → IDENTITY.md → USER.md → HEARTBEAT.md → BOOTSTRAP.md → MEMORY.md
```

MEMORY.md **仅在文件存在时加载**（其他缺失文件标记为 `missing: true`，MEMORY.md 直接跳过）。

缓存优化：以 `inode + dev + size + mtimeMs` 作为文件身份标识，未变化则命中缓存。

### 2. memory_search / memory_get 工具

**关键文件**: `extensions/memory-core/src/tools.ts`

两个 agent 工具：
- `memory_search`：语义搜索 MEMORY.md + `memory/*.md` + 会话记录，支持混合模式（向量语义 + 关键词匹配）
- `memory_get`：精确读取 MEMORY.md 或 `memory/*.md` 的指定行范围

搜索需要配置 embedding provider。

## 四、截断机制

MEMORY.md 的截断发生在**注入系统 prompt 时**，有两层限制。

### 单文件截断

**关键文件**: `src/agents/pi-embedded-helpers/bootstrap.ts:132-228`

- 默认上限：**12,000 chars / 文件**
- 截断比例：**75% head + 25% tail**
- 截断标记：`[...truncated, read MEMORY.md for full content...]`
- 可选详细标记：`…(truncated MEMORY.md: kept 9000+3000 chars of 50000)…`
- 算法迭代 3 次确定 head/tail 各保留多少字符，确保 marker + 内容不超过 maxChars
- 空间极度紧张时回退到紧凑格式 `[…truncated 9000+3000/50000]`

### 总量截断

**同文件第 271-330 行**：
- 默认上限：**60,000 chars**（所有 bootstrap 文件合计）
- 按文件顺序依次分配预算，剩余预算不足 64 chars 时跳过后续文件

### 每日笔记截断

**关键文件**: `src/auto-reply/reply/startup-context.ts:7-15`：
- 单文件：16,384 bytes / **1,200 chars**
- 总预算：**2,800 chars**
- 回溯天数：2 天（今天 + 昨天）
- 超出时用二分搜索裁剪，标注 `[Untrusted daily memory: ...]`

## 五、数据流总览

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

## 六、核心设计原则

1. **MEMORY.md 永远不被覆盖或删减** — 只有读取（带截断）和追加（经晋升）
2. **运行时写入先进每日笔记** — 经严格筛选后才能晋升到长期记忆
3. **大小写敏感的文件解析** — 优先 `MEMORY.md`，兼容 `memory.md`，排除符号链接
4. **缓存优化** — 以 `inode + dev + size + mtimeMs` 作为文件身份标识
5. **边界安全** — 通过 `openRootFile` 限制最大读取 2MB，防止路径穿越

## 七、关键源文件索引

| 文件 | 行号范围 | 职责 |
|------|---------|------|
| `src/memory/root-memory-files.ts` | 全文 | 文件定位与解析 |
| `src/agents/workspace.ts` | 615-680 | Bootstrap 加载 |
| `src/agents/pi-embedded-helpers/bootstrap.ts` | 95-96, 132-228, 271-330 | 截断算法 |
| `src/auto-reply/reply/startup-context.ts` | 7-15 | 每日笔记加载 |
| `extensions/memory-core/src/flush-plan.ts` | 110-113 | 预压缩刷写触发条件 |
| `extensions/memory-core/src/short-term-promotion.ts` | 1551-1686 | Dreaming 晋升 |
| `extensions/memory-core/src/tools.ts` | — | 搜索与读取工具 |
