---
type: source
title: "OpenClaw Memory Backend Comparison"
status: active
tags: [openclaw, memory, backend, comparison]
created: 2026-05-12
updated: 2026-05-12
related:
  - "[[OpenClaw Memory System Overview]]"
  - "[[OpenClaw Memory Architecture Analysis]]"
  - "[[OpenClaw Memory Data Flow]]"
  - "[[OpenClaw Memory Research Corrections]]"
  - "[[OpenClaw MEMORY.md Implementation]]"
---

# OpenClaw Memory Backend 实现差异分析（修订版）

## 核心结论

memory-core 和 memory-lancedb 是**并列的 Memory 插件**，不是同一层的不同后端：

| 插件 | 存储 | MEMORY.md | 工具 | 返回格式 |
|------|------|-----------|------|----------|
| memory-core | 文件系统 + SQLite 索引 | **存在** | search, get | 文件片段引用+摘要 |
| memory-lancedb | LanceDB 表 | **不存在** | recall, store, forget | 独立条目 |

---

## 一、memory-core：SQLite 是索引层，MEMORY.md 是存储

### 架构层次

```
MEMORY.md + memory/*.md  ← 实际内容存储（人类可编辑）
    ↓ 分块（~400 token）
SQLite 索引              ← 索引层，不存储完整内容
    ├── chunks_fts (BM25)
    └── chunks_vec (向量)
```

### 代码证据

**索引时写入 SQLite**：`manager-sync-ops.ts`

```typescript
// 监听 MEMORY.md 和 memory/ 目录变更
const watchPaths = new Set<string>([
  path.join(this.workspaceDir, "MEMORY.md"),
  path.join(this.workspaceDir, "memory"),
]);
this.watcher = resolveMemoryWatchFactory()(Array.from(watchPaths), {...});
```

**搜索返回引用+摘要**：`types.ts`

```typescript
type MemorySearchResult = {
  path: string;        // 文件路径，如 "MEMORY.md"
  startLine: number;   // 行号范围
  endLine: number;
  snippet: string;     // 文本片段摘要
  score: number;
  source: MemorySource;
};
```

> **修正说明**：原文称 "snippet: 仅用于显示，非完整内容"。实际上 snippet 是搜索返回的文本摘要，但完整内容仍需 `memory_get` 从文件读取。两种表述都有部分正确。— 2026-05-12

**memory_get 从文件读取**：`read-file.ts`

```typescript
export async function readMemoryFile(params: {
  workspaceDir: string;
  relPath: string;
  from?: number;
  lines?: number;
}): Promise<MemoryReadResult> {
  // 直接从文件系统读取，不从 SQLite
  const absPath = path.resolve(params.workspaceDir, params.relPath);
  const content = await readRegularFile({ filePath: absPath });
  return buildMemoryReadResult({ content, relPath, from, lines });
}
```

### Dreaming 写入 MEMORY.md

Deep Phase 是唯一写入 MEMORY.md 的阶段：

```typescript
// dreaming.ts:160
`Promote weighted short-term recalls into MEMORY.md (limit=${config.limit}...)`

// dreaming.ts:613
`Promoted ${applied.applied} candidate(s) into MEMORY.md.`
```

### Flush 也在 memory-core 中

```typescript
// extensions/memory-core/src/flush-plan.ts
// Flush 机制是 memory-core 的一部分
// 如果替换 memory-core 为 memory-lancedb，Flush 功能将丢失
```

> **注意**：Flush 机制（compaction 前自动保存到每日笔记）位于 `extensions/memory-core/src/flush-plan.ts`，是 memory-core 的功能。替换 memory-core 后此功能不可用。— 2026-05-12

---

## 二、memory-lancedb：LanceDB 是唯一存储

### 架构

```
LanceDB 表 MemoryEntry  ← 唯一存储
    ├── id (UUID)
    ├── text (原始文本)
    ├── vector
    ├── importance
    ├── category
    └── createdAt
```

**没有 MEMORY.md**：`memory-lancedb/` 目录中没有任何 MEMORY.md 相关代码。

### 搜索返回独立条目

```typescript
// index.ts:50
type MemorySearchResult = {
  entry: MemoryEntry;  // 完整条目
  score: number;
};

// index.ts:262
async search(vector: number[], limit = 5, minScore = 0.5): Promise<MemorySearchResult[]> {
  const results = await this.table!.vectorSearch(vector).limit(limit).toArray();
  return mapped.filter((r) => r.score >= minScore);
}
```

### 内置 Auto Hooks

```typescript
// before_prompt_build：自动注入
api.on("before_prompt_build", async (event) => {
  const vector = await embeddings.embed(event.prompt);
  const results = await db.search(vector, 3, 0.3);
  return { prependContext: formatRelevantMemoriesContext(results) };
});

// agent_end：自动捕获
api.on("agent_end", async (event) => {
  for (const text of extractUserTextContent(message)) {
    await db.store({ text, vector, importance: 0.7, category });
  }
});
```

---

## 三、插件声明对比

### memory-core

```json
{
  "id": "memory-core",
  "kind": "memory",
  "contracts": {
    "tools": ["memory_get", "memory_search"]
  }
}
```

### memory-lancedb

```json
{
  "id": "memory-lancedb",
  "kind": "memory",
  "contracts": {
    "tools": ["memory_forget", "memory_recall", "memory_store"]
  }
}
```

两者 `kind` 都是 `"memory"`，通过 `plugins.slots.memory` 槽位选择。

---

## 四、Active Memory 的适配逻辑

```json
// active-memory/uiHints.toolsAllow
"Defaults to memory_search and memory_get,
or memory_recall when plugins.slots.memory selects memory-lancedb"
```

代码实现：根据槽位自动选择工具集。

---

## 五、QMD/Honcho vs memory-lancedb

**关键区别**：

- QMD/Honcho 是 memory-core 的**搜索后端**（替换 SQLite）
- memory-lancedb 是**并列的 Memory 插件**（替换 memory-core）

```
memory-core 插件
    ↓ memory.backend 配置
    ├── builtin（SQLite）
    ├── qmd（sidecar）
    └── honcho（云端 + 额外工具）  ← 仅官方文档描述，代码中未见实现

memory-lancedb 插件（完全独立）
    ↓ 仅 LanceDB
```

> **代码验证**：QMD 在 `memory-host-sdk` 中有独立引擎实现。Honcho **仅见于官方文档描述，当前代码库中未找到实现代码**。— 2026-05-12

---

## 六、复刻要点

### memory-core

1. 文件系统存储（MEMORY.md + memory/*.md）
2. SQLite 索引（FTS5 + 可选向量）
3. 搜索返回 `(path, line_range, snippet)`，完整内容从文件读取
4. Dreaming 可选，写入 MEMORY.md
5. Flush 可选，写入 memory/*.md（compaction 前触发）

### memory-lancedb

1. LanceDB 向量表存储
2. 搜索返回完整条目（id + text + metadata）
3. 可选 auto-recall/auto-capture hooks
4. 无文件依赖

---

## 七、代码路径

| 功能 | 路径 |
|------|------|
| memory-core 工具定义 | `extensions/memory-core/src/tools.ts:238` (search), `:400` (get) |
| memory-core 索引管理 | `extensions/memory-core/src/memory/manager.ts` |
| memory-core 文件监听 | `extensions/memory-core/src/memory/manager-sync-ops.ts:424` |
| memory-core Flush | `extensions/memory-core/src/flush-plan.ts` |
| memory-lancedb 工具定义 | `extensions/memory-lancedb/index.ts:678` (recall), `:729` (store) |
| memory-lancedb 存储类 | `extensions/memory-lancedb/index.ts:200` (MemoryDatabase) |
| memory-lancedb auto hooks | `extensions/memory-lancedb/index.ts:988` (before_prompt), `:1038` (agent_end) |
| SDK 搜索结果类型 | `packages/memory-host-sdk/src/host/types.ts:3` |
| SDK 文件读取 | `packages/memory-host-sdk/src/host/read-file.ts:23` |

### 相关 Wiki 页面

- [[OpenClaw Memory System Overview]] — 系统综合介绍
- [[OpenClaw Memory Architecture Analysis]] — 架构与代码对应
- [[OpenClaw Memory Data Flow]] — 数据流时间点
- [[OpenClaw MEMORY.md Implementation]] — MEMORY.md 源码分析
- [[OpenClaw Memory Research Corrections]] — 调研误解与纠正
