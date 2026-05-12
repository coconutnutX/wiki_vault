---
type: source
title: OpenClaw Memory System Overview
status: active
tags:
  - openclaw
  - memory
  - plugin
  - architecture
created: 2026-05-12
updated: 2026-05-12
related:
  - "[[OpenClaw Memory Backend Comparison]]"
  - "[[OpenClaw Memory Architecture Analysis]]"
  - "[[OpenClaw Memory Data Flow]]"
  - "[[OpenClaw MEMORY.md Implementation]]"
  - "[[OpenClaw Memory System]]"
---

# OpenClaw Memory 系统综合介绍（修订版）

本文从开发者视角分析 OpenClaw 的内存系统架构，帮助理解各模块的关系及复刻要点。

## 一、核心发现：两种不同的 Memory 插件

OpenClaw 有**两种并列的 Memory 插件**，它们是**完全不同的架构**，不是同一层的不同后端：

| 插件 | 存储 | 工具 | MEMORY.md |
|------|------|------|-----------|
| **memory-core** | 文件系统（MEMORY.md + memory/*.md）+ SQLite 索引 | `memory_search`, `memory_get` | **存在**，是核心 |
| **memory-lancedb** | LanceDB 向量表（独立条目） | `memory_recall`, `memory_store`, `memory_forget` | **不存在** |

**关键点**：
- MEMORY.md 只在使用 **memory-core** 时存在
- memory-lancedb 使用完全不同的存储模型，没有 MEMORY.md
- 通过 `plugins.slots.memory` 槽位选择使用哪个插件

---

## 二、Memory Core：文件系统 + SQLite 索引

### 架构

```
┌─────────────────────────────────────────────────────────┐
│                    存储层（人类可编辑）                    │
│  MEMORY.md        ← 长期记忆（启动时加载到 prompt）       │
│  memory/*.md      ← 每日笔记                              │
│  DREAMS.md        ← Dream Diary（可选）                   │
└─────────────────────────────────────────────────────────┘
                          ↓ 分块索引
┌─────────────────────────────────────────────────────────┐
│                    SQLite 索引层                          │
│  chunks_fts       ← BM25 关键词索引                       │
│  chunks_vec       ← 向量索引（sqlite-vec，可选）          │
│  ~/.openclaw/memory/<agentId>.sqlite                     │
└─────────────────────────────────────────────────────────┘
```

### SQLite 和 MEMORY.md 的关系

**SQLite 只是索引层，不存储完整内容**：

1. **索引时**：读取 MEMORY.md + memory/*.md → 分块（~400 token）→ 存入 SQLite（path + line range + embedding）
2. **搜索时**：query → SQLite 返回 `(path, startLine, endLine, snippet, score)` → 返回包含文本片段摘要
3. **读取时**：`memory_get` 用搜索返回的路径 → **从文件读取实际完整内容**

```typescript
// 搜索返回的是文件片段引用 + snippet 摘要
type MemorySearchResult = {
  path: string;        // "MEMORY.md" 或 "memory/2024-01-15.md"
  startLine: number;   // 行号范围
  endLine: number;
  snippet: string;     // 文本片段摘要
  score: number;
};

// memory_get 从文件系统读取完整内容
async function memory_get(path, from, lines) {
  return await readMemoryFile({ workspaceDir, relPath: path, from, lines });
}
```

> **修正说明**：原文称"搜索只返回引用 (path, line_range)"，实际搜索也返回 `snippet` 文本摘要，但完整内容仍需 `memory_get` 从文件读取。— 2026-05-12

### Dreaming：写入 MEMORY.md

Dreaming 是 memory-core 的可选功能，用于**自动提升短期记忆到长期**：

```
memory/*.md（短期）→ Light Phase → 暂存 → REM Phase → Deep Phase → 写入 MEMORY.md（长期）
```

Deep Phase 是**唯一写入 MEMORY.md** 的阶段：
- 从短期召回记录中评分候选条目
- 通过阈值门控（minScore=**0.75**, minRecallCount=**3**, minUniqueQueries=**2**）
- **追加到 MEMORY.md**（人类可审查）

---

## 三、Memory LanceDB：纯向量数据库存储

### 架构

```
┌─────────────────────────────────────────────────────────┐
│                    LanceDB 存储层                         │
│  MemoryEntry 表：                                        │
│  ├── id (UUID)                                          │
│  ├── text (原始文本，独立条目)                            │
│  ├── vector (embedding)                                 │
│  ├── importance (0-1)                                   │
│  ├── category (preference/fact/decision/other)          │
│  └── createdAt                                          │
│  ~/.openclaw/memory/lancedb/                             │
└─────────────────────────────────────────────────────────┘
```

**没有 MEMORY.md**：所有记忆都是 LanceDB 中的独立条目。

> **注意**：memory-lancedb 的 `openclaw.plugin.json` schema 接受 `dreaming` 配置，但**代码中未实际实现** dreaming 功能。参见 [[OpenClaw Memory Research Corrections]]。

### 工具

```typescript
// 搜索返回独立条目
memory_recall(query, limit) → {
  entry: { id, text, category, importance },
  score: number
}

// 存储新条目
memory_store(text, importance?, category?) → { id, text, ... }

// 删除条目
memory_forget(id) → void
```

### Auto Hooks

memory-lancedb 内置自动功能：

```typescript
// before_prompt_build：自动注入相关记忆
api.on("before_prompt_build", async (event) => {
  const vector = await embeddings.embed(event.prompt);
  const results = await db.search(vector, 3, 0.3);
  return { prependContext: formatMemories(results) };
});

// agent_end：自动捕获用户消息
api.on("agent_end", async (event) => {
  for (const text of extractUserText(event.messages)) {
    await db.store({ text, vector, importance: 0.7, category });
  }
});
```

---

## 四、插件槽位系统

通过 `plugins.slots.memory` 选择使用哪个 memory 插件：

```json5
{
  plugins: {
    slots: {
      memory: "memory-core"  // 或 "memory-lancedb"，或 "none"（禁用）
    }
  }
}
```

### 插件声明

```json
// memory-core/openclaw.plugin.json
{
  "id": "memory-core",
  "kind": "memory",
  "contracts": {
    "tools": ["memory_get", "memory_search"]
  }
}

// memory-lancedb/openclaw.plugin.json
{
  "id": "memory-lancedb",
  "kind": "memory",
  "contracts": {
    "tools": ["memory_forget", "memory_recall", "memory_store"]
  }
}
```

### Active Memory 的适配

active-memory 插件根据槽位自动选择工具：

```json
// active-memory/openclaw.plugin.json uiHints.toolsAllow
"Defaults to memory_search and memory_get, or memory_recall
when plugins.slots.memory selects memory-lancedb"
```

---

## 五、QMD 和 Honcho：memory-core 的搜索后端

**注意**：QMD 和 Honcho 不是并列的 memory 插件，而是 memory-core 的**搜索后端选择**。

```
memory-core 插件
    ↓ 搜索请求
    ↓ 后端选择（通过 memory.backend 配置）
    ├── builtin（SQLite，默认）
    ├── qmd（sidecar 进程）
    └── honcho（云端服务，额外工具）
```

> **代码验证说明**：QMD 在代码中存在独立引擎实现（`memory-host-sdk` 中的 QMD 引擎）。Honcho 仅见于官方文档描述，**当前代码库中未找到 Honcho 的实现代码**，可能尚未发布或已移除。— 2026-05-12

### 后端对比

| 后端 | 配置 | 索引位置 | 特点 |
|------|------|----------|------|
| builtin | `memory.backend = "builtin"`（默认） | SQLite | 无额外依赖 |
| qmd | `memory.backend = "qmd"` | QMD sidecar | reranking、索引外部目录 |
| honcho | 安装插件 | 云端 | 跨会话、用户画像、多 agent（仅文档描述） |

**Honcho 特殊**：安装后提供额外工具（`honcho_context`, `honcho_ask` 等），可与 memory-core 共存。

---

## 六、正确的架构图

```
┌─────────────────────────────────────────────────────────────┐
│                Memory Plugin Slot (plugins.slots.memory)     │
│                                                              │
│  选择一：memory-core          选择二：memory-lancedb         │
│  ┌───────────────────────┐   ┌───────────────────────┐      │
│  │ 文件存储层             │   │ LanceDB 存储层        │      │
│  │ MEMORY.md + memory/*.md│   │ MemoryEntry 表       │      │
│  │ DREAMS.md（可选）      │   │ （无 MEMORY.md）      │      │
│  └───────────────────────┘   └───────────────────────┘      │
│           ↓ 索引                        ↓ 向量               │
│  ┌───────────────────────┐   ┌───────────────────────┐      │
│  │ SQLite / QMD / Honcho │   │ LanceDB vector search │      │
│  │ （搜索后端选择）        │   │ （唯一后端）          │      │
│  └───────────────────────┘   └───────────────────────┘      │
│           ↓                              ↓                   │
│  tools: memory_search,         tools: memory_recall,        │
│        memory_get                     memory_store,         │
│                                       memory_forget         │
│           ↓                              ↓                   │
│  返回：文件片段引用+摘要        返回：独立条目                │
│  (path, line_range, snippet)   (id, text, category)          │
│                                                              │
│  ┌───────────────────────┐                                  │
│  │ Dreaming（可选）       │   ← 仅 memory-core 有此功能      │
│  │ 写入 MEMORY.md         │                                  │
│  │ Flush（memory-core）   │   ← 替换 memory-core 后会丢失    │
│  └───────────────────────┘                                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Enhancement Layer                          │
│                                                              │
│  Active Memory：阻塞式记忆检索                                │
│  - 使用 memory-core 时调用 memory_search/get                 │
│  - 使用 memory-lancedb 时调用 memory_recall                  │
│                                                              │
│  Commitments：推断式跟进（独立功能）                          │
│  Memory Wiki：结构化知识库（与 memory-core 并存）             │
└─────────────────────────────────────────────────────────────┘
```

---

## 七、模块关系总结

| 模块 | 类型 | 存储 | 工具 | 能否共存 |
|------|------|------|------|----------|
| memory-core | Memory 插件 | 文件 + SQLite | search/get | 与 memory-lancedb **互斥** |
| memory-lancedb | Memory 插件 | LanceDB 表 | recall/store/forget | 与 memory-core **互斥** |
| QMD | 搜索后端 | Sidecar | — | memory-core 内，与 builtin/honcho 互斥 |
| Honcho | 搜索后端+额外工具 | 云端 | honcho_* | 可与 memory-core 共存（仅文档描述） |
| Active Memory | 增强插件 | — | — | 与任何 memory 插件共存 |
| Dreaming | memory-core 功能 | 写入 MEMORY.md | — | 仅 memory-core 可用 |
| Flush | memory-core 功能 | 写入 memory/*.md | — | 仅 memory-core 可用 |

---

## 八、复刻要点

### 复刻 memory-core 架构

**必须实现**：

1. **文件存储层**
   - MEMORY.md：长期记忆，启动时加载
   - memory/*.md：每日笔记，可搜索

2. **索引层**
   - 分块策略：~400 token，80 token 重叠
   - SQLite：FTS5（关键词）+ 可选向量索引
   - 文件变更时重新索引

3. **工具接口**
   ```
   memory_search(query) → [{path, startLine, endLine, snippet, score}]
   memory_get(path, from?, lines?) → {text, path}
   ```

4. **关键区别**：索引返回引用+摘要，工具从文件读取完整内容

### 复刻 memory-lancedb 架构

**必须实现**：

1. **向量数据库存储**
   - 独立条目（id + text + vector + metadata）
   - 无文件系统依赖

2. **工具接口**
   ```
   memory_recall(query) → [{id, text, category, score}]
   memory_store(text) → {id, text, ...}
   memory_forget(id) → void
   ```

3. **可选**：auto-recall/auto-capture hooks

### 决策点

| 需求 | 推荐架构 |
|------|----------|
| 人类可编辑、可审查 | memory-core（MEMORY.md） |
| 与笔记系统集成 | memory-core |
| 纯向量搜索、无文件依赖 | memory-lancedb |
| 自动化程度高 | memory-lancedb（内置 auto hooks） |

---

## 九、配置示例

### memory-core（默认）

```json5
{
  plugins: {
    slots: {
      memory: "memory-core"  // 默认值
    },
    entries: {
      "memory-core": {
        config: {
          dreaming: {
            enabled: true,
            frequency: "0 3 * * *"
          }
        }
      }
    }
  },
  memory: {
    backend: "builtin"  // 或 "qmd"
  }
}
```

### memory-lancedb

```json5
{
  plugins: {
    slots: {
      memory: "memory-lancedb"
    },
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            provider: "openai",
            model: "text-embedding-3-small"
          },
          autoRecall: true,
          autoCapture: true
        }
      }
    }
  }
}
```

---

## 十、参考资料

### 官方文档

| 主题 | 链接 |
|------|------|
| Memory 概述 | https://docs.openclaw.ai/concepts/memory |
| Builtin 后端（memory-core） | https://docs.openclaw.ai/concepts/memory-builtin |
| QMD 后端 | https://docs.openclaw.ai/concepts/memory-qmd |
| Honcho 后端 | https://docs.openclaw.ai/concepts/memory-honcho |
| Memory Search | https://docs.openclaw.ai/concepts/memory-search |
| Active Memory | https://docs.openclaw.ai/concepts/active-memory |
| Commitments | https://docs.openclaw.ai/concepts/commitments |
| Dreaming | https://docs.openclaw.ai/concepts/dreaming |
| Memory LanceDB | https://docs.openclaw.ai/plugins/memory-lancedb |

### 代码路径

| 模块 | 路径 |
|------|------|
| memory-core 工具 | `extensions/memory-core/src/tools.ts` |
| memory-core 存储管理 | `extensions/memory-core/src/memory/manager.ts` |
| memory-core Dreaming | `extensions/memory-core/src/dreaming.ts` |
| memory-core Flush | `extensions/memory-core/src/flush-plan.ts` |
| memory-lancedb 主文件 | `extensions/memory-lancedb/index.ts` |
| SDK 接口定义 | `packages/memory-host-sdk/src/host/types.ts` |
| 文件读取（memory-core） | `packages/memory-host-sdk/src/host/read-file.ts` |

### 配置参考

| 配置项 | 链接 |
|------|------|
| Memory 配置参考 | https://docs.openclaw.ai/reference/memory-config |
| Gateway 配置参考 | https://docs.openclaw.ai/gateway/configuration-reference |

