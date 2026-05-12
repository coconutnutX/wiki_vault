---
type: source
title: "OpenClaw Memory Research Corrections"
status: active
tags: [openclaw, memory, corrections, methodology]
created: 2026-05-12
updated: 2026-05-12
related:
  - "[[OpenClaw Memory System Overview]]"
  - "[[OpenClaw Memory Backend Comparison]]"
  - "[[OpenClaw Memory Architecture Analysis]]"
  - "[[OpenClaw Memory Data Flow]]"
  - "[[OpenClaw MEMORY.md Implementation]]"
---

# OpenClaw Memory 系统调研：误解与纠正记录

本文档记录调研过程中产生的错误理解、易混淆点和纠正过程，作为学习笔记和后续参考。

---

## 误解一：memory-core 和 memory-lancedb 是同一层的不同后端

### 错误理解

最初认为架构是：
```
Memory Core（核心层）
    ↓ 后端选择（可替换）
    ├── builtin（SQLite，默认）
    ├── qmd（sidecar）
    ├── honcho（云端）
    └── lancedb（向量数据库）
```

认为 memory-lancedb 只是 memory-core 的一个可选后端，类似于 QMD 或 Honcho。

### 实际情况

**memory-core 和 memory-lancedb 是并列的 Memory 插件**，完全独立：

```
plugins.slots.memory 槽位选择
    ├── memory-core（文件存储 + SQLite 索引）
    └── memory-lancedb（LanceDB 向量存储）
```

两者 `kind: "memory"`，通过槽位互斥选择，不是同一层的不同实现。

### 证据

1. **插件声明**：两者都有独立的 `openclaw.plugin.json`，`id` 不同，`kind` 都是 `"memory"`
2. **工具不同**：memory-core 提供 `memory_search/get`，memory-lancedb 提供 `memory_recall/store/forget`
3. **存储不同**：memory-core 用文件系统（MEMORY.md），memory-lancedb 用 LanceDB 表
4. **MEMORY.md 只在 memory-core 中存在**

### 纠正后的理解

| 插件 | 存储模型 | 工具 | MEMORY.md |
|------|----------|------|-----------|
| memory-core | 文件 + SQLite 索引 | search, get | **存在** |
| memory-lancedb | LanceDB 表 | recall, store, forget | **不存在** |

---

## 误解二：MEMORY.md 始终存在，是所有 Memory 插件的核心

### 错误理解

认为 MEMORY.md 是 OpenClaw Memory 系统的核心文件，所有 memory 插件都依赖它。

### 实际情况

**MEMORY.md 只在使用 memory-core 时存在**。

memory-lancedb 完全不使用 MEMORY.md：
- 存储在 LanceDB 表中（MemoryEntry）
- 每条记忆是独立的 `{id, text, vector, category, importance}` 条目
- 搜索返回完整条目，不返回文件片段引用

### 证据

```bash
# memory-lancedb 中没有 MEMORY.md 相关代码
grep -rn "MEMORY.md" extensions/memory-lancedb/
# 输出：(无结果)

# memory-core 中大量引用 MEMORY.md
grep -rn "MEMORY.md" extensions/memory-core/src/
# 输出：多处引用（索引、搜索、Dreaming 写入等）
```

### 纠正后的理解

- memory-core：MEMORY.md 是核心，SQLite 只是索引层
- memory-lancedb：没有 MEMORY.md，LanceDB 是唯一存储

---

## 误解三：SQLite 是 memory-core 的存储层

### 错误理解

认为 memory-core 的架构是：
```
MEMORY.md → 写入 SQLite → 搜索返回 SQLite 内容
```

认为 SQLite 存储记忆内容，类似于 LanceDB 在 memory-lancedb 中的作用。

### 实际情况

**SQLite 只是索引层，不存储完整内容**：

```
MEMORY.md + memory/*.md  ← 实际内容存储（人类可编辑）
    ↓ 分块索引
SQLite                   ← 索引层（path + line range + embedding）
    ↓ 搜索
返回 (path, startLine, endLine, score, snippet)
    ↓ memory_get
从文件读取实际完整内容
```

### 证据

```typescript
// packages/memory-host-sdk/src/host/types.ts
type MemorySearchResult = {
  path: string;        // 文件路径，如 "MEMORY.md"
  startLine: number;   // 行号范围
  endLine: number;
  snippet: string;     // 文本片段摘要
  score: number;
};

// packages/memory-host-sdk/src/host/read-file.ts
export async function readMemoryFile(params: {
  workspaceDir: string;
  relPath: string;
  from?: number;
  lines?: number;
}): Promise<MemoryReadResult> {
  // 直接从文件系统读取
  const absPath = path.resolve(params.workspaceDir, params.relPath);
  const content = await readRegularFile({ filePath: absPath });
  return buildMemoryReadResult({ content, ... });
}
```

### 纠正后的理解

- SQLite 存储：`{path, line_range, embedding}`（索引引用）
- 实际内容在 MEMORY.md + memory/*.md 文件中
- 搜索返回引用+摘要（snippet），memory_get 从文件读取完整内容

> **微妙修正**（2026-05-12）：原文说"搜索只返回引用"过于简化。实际上 `memory_search` 返回的 `MemorySearchResult` 包含 `snippet` 字段（文本片段摘要），但完整内容仍需 `memory_get` 从文件读取。两种表述都有部分正确。

---

## 误解四：QMD 和 Honcho 与 memory-lancedb 同级

### 错误理解

认为 QMD、Honcho、LanceDB 都是并列的 Memory 后端选项：

```
Memory Backend 选择
    ├── builtin
    ├── qmd
    ├── honcho
    └── lancedb
```

### 实际情况

**QMD 和 Honcho 是 memory-core 的搜索后端**，不是并列的 Memory 插件：

```
memory-core 插件
    ↓ memory.backend 配置选择搜索后端
    ├── builtin（SQLite，默认）
    ├── qmd（sidecar 进程）
    └── honcho（云端服务 + 额外工具）

memory-lancedb 插件（完全独立）
    ↓ 只有 LanceDB 作为后端
```

### 证据

```typescript
// memory-core/src/tools.ts:255
const { resolveMemoryBackendConfig } = await loadMemoryToolRuntime();

// memory-core/src/tools.ts:434
const resolved = resolveMemoryBackendConfig({ cfg, agentId });
if (resolved.backend === "builtin") { ... }
// qmd 也走这个路径
```

```json5
// 配置示例
{
  memory: {
    backend: "builtin"  // 或 "qmd"，这是 memory-core 的后端选择
  }
}
```

### 纠正后的理解

| 名称 | 层级 | 作用 |
|------|------|------|
| memory-core | Memory 插件 | 文件存储 + 提供工具 |
| memory-lancedb | Memory 插件 | LanceDB 存储 + 提供工具 |
| builtin | 搜索后端 | memory-core 的 SQLite 索引 |
| qmd | 搜索后端 | memory-core 的 sidecar 索引 |
| honcho | 搜索后端+插件 | memory-core 的云端索引 + 额外工具 |

> **代码验证补充**（2026-05-12）：QMD 在 `memory-host-sdk` 中有独立引擎实现。Honcho **仅见于官方文档描述，当前代码库中未找到实现代码**，可能尚未发布或已移除。

---

## 误解五（2026-05-12 新增）：Dreaming 阈值默认值混淆

### 错误内容

[[OpenClaw Memory Architecture Analysis]] 中 §2.8 评分信号部分记录的默认阈值：

~~`candidate.score >= minScore          // 默认 0.5`~~
~~`candidate.recallCount >= minRecallCount  // 默认 2`~~

### 正确值

经代码验证 `extensions/memory-core/src/short-term-promotion.ts:24-26`：

- `DEFAULT_PROMOTION_MIN_SCORE = 0.75`
- `DEFAULT_PROMOTION_MIN_RECALL_COUNT = 3`
- `DEFAULT_PROMOTION_MIN_UNIQUE_QUERIES = 2`

与官方文档 Dreaming 页面一致。

### 混淆来源

之前的 wiki 页面 [[OpenClaw MEMORY.md Implementation]] 记录的是正确值（0.75/3/2），但架构分析文档中写了错误值（0.5/2/2）。可能是在不同时间点查看不同版本的代码或配置时引入的偏差。

### 影响

- `minScore` 从 0.5 → 0.75：晋升门槛更高，更少候选条目会被写入 MEMORY.md
- `minRecallCount` 从 2 → 3：需要更多次被 recall 才能晋升，更保守

---

## 误解六（2026-05-12 新增）：搜索结果返回格式

### 过度简化

多处文档表述为"搜索只返回引用 (path, line_range)"。

### 实际情况

`memory_search` 返回的 `MemorySearchResult` 包含：
- `path`: 文件路径
- `startLine` / `endLine`: 行号范围
- `snippet`: **文本片段摘要**（非空）
- `score`: 相关度分数

搜索确实返回了文本内容（snippet），但这个 snippet 是摘要/截断版，不是完整内容。要获取完整文本，仍需调用 `memory_get` 从文件系统读取。

### 纠正

应表述为："搜索返回文件片段引用**和文本摘要**，完整内容用 `memory_get` 从文件读取。"

---

## 误解七（2026-05-12 新增）：Flush 机制的归属

### 错误理解

暗示 Flush 是框架核心功能，不依赖具体 memory 插件。

### 实际情况

**Flush 机制位于 `extensions/memory-core/src/flush-plan.ts`**，是 memory-core 的一部分。

替换 memory-core 为 memory-lancedb 后：
- compaction 前的自动保存功能**不可用**
- memory-lancedb 有自己的 auto-capture hook（agent_end 时触发），但机制完全不同

### 影响

选择 memory-lancedb 时，需要意识到：
1. 没有 Memory Flush（compaction 前自动保存到每日笔记）
2. 没有 Dreaming（短期 → 长期自动晋升）
3. 依赖 auto-capture hook 和手动 `memory_store` 替代

---

## 误解八（2026-05-12 新增）：memory-lancedb 的 Dreaming 支持

### 错误理解

`openclaw-memory-research-mistakes.md` 原文有"待后续确认"的问题："Dreaming 在 memory-lancedb 中的实现？文档说 memory-lancedb 也有 dreaming 配置，具体作用是什么？"

### 确认结果

memory-lancedb 的 `openclaw.plugin.json` schema 接受 `dreaming` 配置对象（第 112-114 行），但**代码中未实际实现任何 dreaming 功能**。

- `index.ts` 中 dreaming 配置仅被传递给配置解析，没有对应的处理逻辑
- 实际的 dreaming 功能（Light/REM/Deep phases、MEMORY.md 写入）全部在 memory-core 中

### 结论

这个 schema 字段可能是为了未来兼容性预留，或者是从 memory-core 的 schema 模板继承而来。当前使用 memory-lancedb 时，dreaming 配置不产生任何效果。

---

## 误解九（2026-05-12 新增）：Dreaming 只写入 DREAMS.md，不写入 MEMORY.md

### 错误理解

看到 Dreaming 输出 DREAMS.md（Dream Diary），误以为 Dreaming 的全部输出都只写 DREAMS.md，不写 MEMORY.md。

### 实际情况

Dreaming 有**两个不同的写入目标**，职责完全不同：

| 输出文件 | 写入阶段 | 内容性质 | Agent 是否加载 |
|----------|----------|----------|----------------|
| **MEMORY.md** | Deep Phase 晋升 | 长期记忆条目（durable facts） | 是，每次 bootstrap 加载 |
| **DREAMS.md** | 各 Phase 汇总后 | Dream Diary 叙述摘要 | 否，仅供人类审查 |

官方文档明确说：
> "Long-term promotion still writes only to `MEMORY.md`."

> "DREAMS.md — Dream Diary and dreaming sweep summaries for **human review**"

### DREAMS.md 的定位

- 类似于"训练日志"——记录 Dreaming 做了什么
- 包含 phase summaries、主题反思、晋升摘要
- **不进入 Agent prompt**，Agent 不会在启动时加载它
- 人类可以在 Obsidian 或文件系统中审查

### MEMORY.md 的定位

- Dreaming Deep Phase 评分通过的候选条目**追加到 MEMORY.md**
- 这些条目成为 Agent 的长期记忆
- **每次会话启动时通过 bootstrap 加载到 prompt**

### 混淆来源

名称相似（DREAMS.md vs Dreaming）容易让人误以为 Dreaming 只写 DREAMS.md。实际上 DREAMS.md 是 Dreaming 的"副产品日志"，MEMORY.md 才是 Dreaming 的"核心产出"。

---

## 易混淆点总结

### 1. 插件 vs 后端

| 概念 | 定义 | 示例 |
|------|------|------|
| Memory 插件 | 提供 memory 工具的顶层插件 | memory-core, memory-lancedb |
| 搜索后端 | memory-core 内部的索引实现 | builtin, qmd, honcho |

### 2. 存储 vs 索引

| 插件 | 存储位置 | 索引位置 | 搜索返回 |
|------|----------|----------|----------|
| memory-core | 文件（MEMORY.md） | SQLite/QMD/Honcho | 文件片段引用+摘要 |
| memory-lancedb | LanceDB 表 | LanceDB 自身 | 完整条目 |

### 3. 槽位 vs 配置

| 配置项 | 作用 |
|------|------|
| `plugins.slots.memory` | 选择 Memory 插件（memory-core 或 memory-lancedb 或 "none"） |
| `memory.backend` | 选择 memory-core 的搜索后端（builtin、qmd） |

### 4. 工具名差异

| 插件 | 工具 | 语义 |
|------|------|------|
| memory-core | `memory_search` | 搜索 → 返回引用+摘要 |
| memory-core | `memory_get` | 获取 → 从文件读完整内容 |
| memory-lancedb | `memory_recall` | 回忆 → 返回完整条目 |
| memory-lancedb | `memory_store` | 存储 → 写入 LanceDB |
| memory-lancedb | `memory_forget` | 遗忘 → 删除条目 |

### 5. 功能归属

| 功能 | 所属插件 | 替换后影响 |
|------|----------|------------|
| Flush | memory-core | 丢失（无 compaction 前自动保存） |
| Dreaming | memory-core | 丢失（无短期→长期晋升） |
| Auto-recall | memory-lancedb | — |
| Auto-capture | memory-lancedb | — |

---

## 调研方法论反思

### 产生误解的原因

1. **文档描述不够精确**：官方文档把 builtin、qmd、honcho、lancedb 都放在 "Memory backends" 卡片组中，容易误解为同一层
2. **命名相似**：`memory-core`、`memory-lancedb`、`memory-backend` 都带 "memory"，层级关系不明显
3. **未仔细阅读代码**：仅看文档和架构图，未深入代码确认存储/索引/工具的实际关系
4. **阈值来源不一致**：不同时间查看不同代码位置，导致阈值默认值记录错误

### 避免误解的方法

1. **查看插件声明**：`openclaw.plugin.json` 中的 `kind` 和 `contracts.tools` 能明确插件定位
2. **检查工具实现**：工具返回的数据结构能揭示存储模型
3. **追踪数据流向**：从写入 → 存储 → 索引 → 搜索 → 返回，完整追踪
4. **grep 关键文件**：如 MEMORY.md 是否被引用，能快速判断存储模型
5. **对照官方文档验证**：用官方文档做基准，用代码做最终验证
6. **标注数据来源**：记录每个结论来自文档还是代码，便于回溯

### 正确的调研顺序

1. 先看 `openclaw.plugin.json`（插件定位）
2. 再看工具定义（工具语义和返回格式）
3. 再看存储实现（数据结构）
4. 然后看配置和文档（理解用户视角）
5. 最后对照官方文档做交叉验证

---

## 概念解释：重水化（Rehydrate）

### 初次理解困惑

看到代码中的 `rehydratePromotionCandidate` 时，不理解"重水化"的含义，以为是某种复杂的水合化学反应概念。

### 实际含义

**重水化（rehydrate）** 是 Dreaming Deep Phase 的关键机制，用于验证候选记忆条目的有效性。

### 问题背景

短期召回记录存储在 `memory/.dreams/` 中，记录的是**引用**（"脱水"状态）：

```typescript
// 短期召回记录
{
  path: "memory/2024-01-15.md",
  startLine: 10,
  endLine: 20,
  score: 0.8,
  snippet: "用户偏好 TypeScript..."  // 只是缓存，可能过时
}
```

这些引用指向的 `memory/*.md` 文件可能已经：
- **删除**：用户删除了该日记文件
- **修改**：内容已变化，行号不再对应
- **过时**：snippet 缓存与实际内容不符

### 重水化的作用

在 Deep Phase 提升（写入 MEMORY.md）之前，从**活文件**重新读取实际内容：

```typescript
// extensions/memory-core/src/short-term-promotion.ts:1611-1617
for (const candidate of selected) {
  // 从活文件重新读取，验证内容是否存在
  const rehydrated = await rehydratePromotionCandidate(workspaceDir, candidate);
  if (rehydrated && !isContaminatedDreamingSnippet(rehydrated.snippet)) {
    rehydratedSelected.push(rehydrated);
  }
}
```

### 比喻理解

- **脱水**：短期召回记录只保存引用（path + line range），像脱水蔬菜
- **重水化**：从活文件加水恢复，读取实际内容，像脱水蔬菜加水复原
- **过滤**：如果文件已删除或内容不存在，重水化失败 → 跳过该候选条目

### 为什么必须重水化

防止**提升已删除内容**：如果用户删除了 `memory/2024-01-15.md`，但不重水化，系统可能会把过时的 snippet 写入 MEMORY.md，造成"僵尸记忆"。

### 设计启示

这是一个重要的**数据完整性保护机制**：

| 场景 | 不重水化的后果 | 重水化后的结果 |
|------|----------------|----------------|
| 文件已删除 | 提升过时内容到 MEMORY.md | 跳过该候选，不提升 |
| 文件被修改 | 提升错误行号的内容 | 读取最新内容或跳过 |
| snippet 过时 | MEMORY.md 包含错误记忆 | 更新为实际内容 |

**教训**：在设计"从缓存/索引写入持久存储"的逻辑时，必须验证源数据的有效性。

---

## 待后续确认的问题

~~1. **Dreaming 在 memory-lancedb 中的实现**：文档说 memory-lancedb 也有 `dreaming` 配置，具体作用是什么？~~
**已确认**（2026-05-12）：schema 接受配置但未实现。参见误解八。

~~2. **Honcho 与 memory-core 的共存方式**：Honcho 既是搜索后端又提供额外工具，具体如何集成？~~
**部分确认**（2026-05-12）：Honcho 仅见于官方文档描述，代码库中未找到实现。待 Honcho 发布后确认。

~~3. **Memory Wiki 的定位**：是否只能与 memory-core 配合，还是也能与 memory-lancedb 配合？~~
**部分确认**：官方文档说 memory-wiki "does not replace the active memory plugin"，它是独立的知识层，理论上可与任何 memory 插件配合。但具体实现细节待进一步验证。

### 相关 Wiki 页面

- [[OpenClaw Memory System Overview]] — 系统综合介绍
- [[OpenClaw Memory Backend Comparison]] — 后端差异分析
- [[OpenClaw Memory Architecture Analysis]] — 架构与代码对应
- [[OpenClaw Memory Data Flow]] — 数据流时间点
- [[OpenClaw MEMORY.md Implementation]] — MEMORY.md 源码分析
- [[OpenClaw Memory System]] — 抽象设计模式
