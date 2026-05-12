---
type: source
title: "OpenClaw Memory Architecture Analysis"
status: active
tags: [openclaw, memory, architecture, code-analysis]
created: 2026-05-12
updated: 2026-05-12
related:
  - "[[OpenClaw Memory System Overview]]"
  - "[[OpenClaw Memory Backend Comparison]]"
  - "[[OpenClaw Memory Data Flow]]"
  - "[[OpenClaw Memory Research Corrections]]"
  - "[[OpenClaw MEMORY.md Implementation]]"
---

# OpenClaw Memory 系统架构与代码对应分析

从架构图出发，分析代码结构如何对应，并进行批判性审视。

---

## 一、架构图回顾

```
┌─────────────────────────────────────────────────────────────┐
│                Memory Plugin Slot (plugins.slots.memory)     │
│                                                              │
│  选择一：memory-core          选择二：memory-lancedb         │
│  ┌───────────────────────┐   ┌───────────────────────┐      │
│  │ 文件存储层             │   │ LanceDB 存储层        │      │
│  │ MEMORY.md + memory/*.md│   │ MemoryEntry 表       │      │
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
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Dreaming（可选，memory-core 功能）                      │  │
│  │ Light → REM → Deep → MEMORY.md                         │  │
│  │ cron / heartbeat 触发                                   │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Enhancement Layer                          │
│                                                              │
│  Active Memory：阻塞式记忆检索（before_prompt_build hook）    │
│  Commitments：推断式跟进（enqueue + heartbeat 交付）         │
│  Memory Wiki：结构化知识库（corpus supplement）               │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、代码结构对应

### 2.1 插件系统核心

**位置**：`src/plugins/`

| 文件 | 作用 | 关键代码 |
|------|------|----------|
| `types.ts:2527` | `OpenClawPluginApi` 定义 | 插件注入的 API 对象 |
| `types.ts:2556` | `registerTool` 方法签名 | 注册工具 |
| `types.ts:2560` | `registerHook` 方法签名 | 注册 hooks |
| `slots.ts` | 槽位选择逻辑 | Memory 插件互斥选择 |
| `hooks.ts` | Hook Runner | 执行插件 hooks |
| `loader.ts` | 插件加载器 | 发现和加载插件 |
| `registry.ts` | 插件注册表 | 管理已加载插件 |

### 2.2 Memory 插件层

**Memory 插件通过槽位互斥**：

```typescript
// src/plugins/slots.ts:12-20
const SLOT_BY_KIND: Record<PluginKind, PluginSlotKey> = {
  memory: "memory",
  "context-engine": "contextEngine",
};

const DEFAULT_SLOT_BY_KEY: Record<PluginSlotKey, string> = {
  memory: "memory-core",      // 默认 memory 插件
  contextEngine: "legacy",
};
```

**槽位选择时自动禁用其他同类插件**：

```typescript
// src/plugins/slots.ts:65-148
export function applyExclusiveSlotSelection(params) {
  // 选择 memory-lancedb 时自动禁用 memory-core
  for (const plugin of params.registry.plugins) {
    if (plugin.id !== params.selectedId && hasKind(plugin.kind, kindForSlot)) {
      entries[plugin.id] = { ...entry, enabled: false };
    }
  }
}
```

### 2.3 memory-core 插件

**位置**：`extensions/memory-core/`

| 文件 | 作用 | 关键代码 |
|------|------|----------|
| `openclaw.plugin.json:1` | 插件声明 | `kind: "memory"`, `tools: ["memory_get", "memory_search"]` |
| `index.ts` (入口) | 插件定义 | `definePluginEntry(...)` |
| `src/tools.ts:238` | memory_search 工具 | 搜索索引返回片段引用 |
| `src/tools.ts:400` | memory_get 工具 | 从文件读取内容 |
| `src/memory/manager.ts` | 索引管理器 | 实现 `MemorySearchManager` 接口 |
| `src/dreaming.ts` | Dreaming 功能 | Deep Phase 写入 MEMORY.md |
| `src/flush-plan.ts` | Flush 功能 | compaction 前写入 memory/*.md |

**工具注册**：

```typescript
// extensions/memory-core/src/index.ts (入口)
export default definePluginEntry({
  id: "memory-core",
  kind: "memory",
  register(api) {
    // 注册工具
    const searchTool = createMemorySearchTool({ ... });
    const getTool = createMemoryGetTool({ ... });
    // 工具通过 api.registerTool 注册
  }
});
```

### 2.4 memory-lancedb 插件

**位置**：`extensions/memory-lancedb/`

| 文件 | 作用 | 关键代码 |
|------|------|----------|
| `openclaw.plugin.json:1` | 插件声明 | `kind: "memory"`, `tools: ["memory_recall", "memory_store", "memory_forget"]` |
| `index.ts:678` | memory_recall 工具 | LanceDB 向量搜索 |
| `index.ts:729` | memory_store 工具 | 存储到 LanceDB 表 |
| `index.ts:988` | auto-recall hook | `api.on("before_prompt_build", ...)` |
| `index.ts:1038` | auto-capture hook | `api.on("agent_end", ...)` |

**Auto hooks 注册**：

```typescript
// extensions/memory-lancedb/index.ts:988
api.on("before_prompt_build", async (event) => {
  if (currentCfg.autoRecall) {
    const vector = await embeddings.embed(event.prompt);
    const results = await db.search(vector, 3, 0.3);
    return { prependContext: formatRelevantMemoriesContext(results) };
  }
});

// extensions/memory-lancedb/index.ts:1038
api.on("agent_end", async (event) => {
  if (currentCfg.autoCapture) {
    await db.store({ text, vector, importance, category });
  }
});
```

### 2.5 搜索后端（memory-core 内部）

**后端通过 `memory.backend` 配置选择**：

```typescript
// extensions/memory-core/src/tools.ts:255, 434
const resolved = resolveMemoryBackendConfig({ cfg, agentId });
if (resolved.backend === "builtin") {
  // 使用 SQLite 索引
} else {
  // QMD 路径
}
```

**QMD 管理器**：

```typescript
// extensions/memory-core/src/memory/qmd-manager.ts
// QMD sidecar 进程管理
// 索引 memory/*.md + MEMORY.md 到 QMD collections
```

> **注意**：Honcho 后端仅见于官方文档描述，当前代码库中未找到实现。— 2026-05-12

### 2.6 Enhancement Layer

**Enhancement Layer 通过 hooks 加入系统**：

| 模块 | 加入方式 | 代码位置 |
|------|----------|----------|
| Active Memory | `before_prompt_build` hook | `extensions/active-memory/index.ts:3017` |
| Commitments | 回复后 enqueue + heartbeat 交付 | `src/commitments/runtime.ts:101` |
| Memory Wiki | 独立插件 + corpus supplement | `extensions/memory-wiki/` |

**Active Memory hook 注册**：

```typescript
// extensions/active-memory/index.ts:3017
api.on(
  "before_prompt_build",
  async (event, ctx) => {
    // 1. 检查启用条件
    if (!isEnabledForAgent(config, effectiveAgentId)) return undefined;

    // 2. 构建查询
    const query = buildQuery({ latestUserMessage: event.prompt, recentTurns });

    // 3. 调用记忆工具（通过槽位自动适配）
    const result = await maybeResolveActiveRecall({
      api, config, agentId, query, searchQuery
    });

    // 4. 返回隐藏上下文
    return { prependContext: buildPromptPrefix(result.summary) };
  },
  { timeoutMs: beforePromptBuildTimeoutMs }
);
```

**Commitments 流程**：

```typescript
// src/auto-reply/reply/agent-runner.ts:1534
enqueueCommitmentExtractionForTurn({
  cfg, userText, assistantText, agentId, sessionKey, channel
});

// src/commitments/runtime.ts:101
export function enqueueCommitmentExtraction(input) {
  queue.push(input);
  // debounce 后运行 embedded Pi agent 提取
}

// src/infra/heartbeat-runner.ts:1303
const heartbeatForDelivery = commitmentDeliveryContext ?
  { ...heartbeat, target: "last" } : heartbeat;
// heartbeat 时交付到期 commitments
```

### 2.7 Hook 类型定义

**位置**：`src/plugins/hook-types.ts`

```typescript
// 关键 hook 类型（memory 相关）
type PluginHookBeforePromptBuildEvent = {
  prompt: string;
  messages?: unknown[];
};

type PluginHookBeforePromptBuildResult = {
  prependContext?: string;
};

type PluginHookAgentEndEvent = {
  success: boolean;
  messages?: unknown[];
};
```

### 2.8 Dreaming 机制（memory-core 功能）

**Dreaming 是 memory-core 插件的可选功能，不是独立插件。**

#### 功能定位

Dreaming 实现**短期记忆 → 长期记忆的自动提升**：

| 阶段 | 职责 | 写入位置 | 作用 |
|------|------|----------|------|
| Light | 收集和暂存短期信号 | `memory/.dreams/` | 不写入 MEMORY.md |
| REM | 反思模式和主题 | `memory/.dreams/` | 不写入 MEMORY.md |
| Deep | 评分和提升候选条目 | `MEMORY.md` | **唯一写入 MEMORY.md 的阶段** |

#### 触发机制

Dreaming 通过**两种触发方式**：

| 触发类型 | 触发条件 | 触发时机 |
|----------|----------|----------|
| Cron Job | `dreaming.enabled + dreaming.frequency` 配置 | 定时（如 `0 3 * * *`） |
| Heartbeat | pending system event token | heartbeat run 时检查 |

**Cron Job 注册流程**：

```typescript
// extensions/memory-core/src/dreaming.ts:866
api.on("gateway_start", async (_event, ctx) => {
  // 启动时创建/更新 managed dreaming cron job
  const config = await reconcileManagedDreamingCron({
    reason: "startup",
    startupConfig: ctx.config,
    startupCron: () => resolveCronServiceFromGatewayContext(ctx),
  });
});

// extensions/memory-core/src/dreaming.ts:408
export async function reconcileShortTermDreamingCronJob(params) {
  if (!params.config.enabled) {
    // 禁用时移除 cron job
    for (const job of managed) {
      await cron.remove(job.id);
    }
    return { status: "disabled" };
  }
  // 启用时创建/更新 cron job
  const desired = buildManagedDreamingCronJob(params.config);
  await cron.add(desired);
}
```

**Cron Job 结构**：

```typescript
// extensions/memory-core/src/dreaming.ts:54-65
type ManagedCronJobCreate = {
  name: string;              // MANAGED_DREAMING_CRON_NAME
  description: string;       // "Promote short-term recalls into MEMORY.md"
  enabled: boolean;
  schedule: { kind: "cron"; expr: string; tz?: string };
  sessionTarget: "main" | "isolated";
  wakeMode: "now";
  payload: { kind: "systemEvent"; text: DREAMING_SYSTEM_EVENT_TEXT };
};
```

**Heartbeat 触发检查**：

```typescript
// extensions/memory-core/src/dreaming.ts:888
api.on("before_agent_reply", async (event, ctx) => {
  // 1. 只在 heartbeat 或 cron trigger 时执行
  if (ctx.trigger !== "heartbeat" && ctx.trigger !== "cron") {
    return undefined;
  }

  // 2. 检查是否有 dreaming system event token
  const hasManagedDreamingToken = includesSystemEventToken(
    event.cleanedBody,
    DREAMING_SYSTEM_EVENT_TEXT,
  );

  // 3. 检查是否有 pending dreaming cron event
  const isManagedHeartbeatTrigger =
    ctx.trigger === "heartbeat" && hasPendingManagedDreamingCronEvent(ctx.sessionKey);

  // 4. 执行 dreaming sweep
  if (hasManagedDreamingToken && (isManagedHeartbeatTrigger || ctx.trigger === "cron")) {
    return await runShortTermDreamingPromotionIfTriggered({
      cleanedBody: event.cleanedBody,
      trigger: ctx.trigger,
      workspaceDir: ctx.workspaceDir,
      config,
      ...
    });
  }
});
```

#### 执行流程

**完整 sweep 流程**：

```typescript
// extensions/memory-core/src/dreaming.ts:549-565
// 1. 运行 Light + REM phases
await runDreamingSweepPhases({
  workspaceDir,
  pluginConfig,
  cfg: params.cfg,
  logger: params.logger,
  subagent: params.subagent,
  nowMs: sweepNowMs,
});

// 2. 修复短期存储 artifacts
const repair = await repairShortTermPromotionArtifacts({ workspaceDir });

// 3. 评分候选条目（Deep phase 核心）
const candidates = await rankShortTermPromotionCandidates({
  workspaceDir,
  limit: params.config.limit,
  minScore: params.config.minScore,
  minRecallCount: params.config.minRecallCount,
  minUniqueQueries: params.config.minUniqueQueries,
  recencyHalfLifeDays,
});

// 4. 应用提升（写入 MEMORY.md）
const applied = await applyShortTermPromotions({
  workspaceDir,
  candidates,
  limit, minScore, minRecallCount, minUniqueQueries,
});
```

**评分信号（Deep Phase）**：

```typescript
// extensions/memory-core/src/short-term-promotion.ts
// 六个加权信号
score = frequency(0.24) + relevance(0.30) + diversity(0.15) +
        recency(0.15) + consolidation(0.10) + conceptual(0.06)

// 阈值门控
candidate.score >= minScore          // 默认 0.75
candidate.recallCount >= minRecallCount  // 默认 3
candidate.uniqueQueries >= minUniqueQueries // 默认 2
```

> ~~candidate.score >= minScore          // 默认 0.5~~
> ~~candidate.recallCount >= minRecallCount  // 默认 2~~
> **修正**（2026-05-12）：经代码验证 `short-term-promotion.ts:24-26`，实际默认值为 `minScore=0.75`、`minRecallCount=3`、`minUniqueQueries=2`。与官方文档 Dreaming 页面描述一致。参见 [[OpenClaw Memory Research Corrections]]。

**写入 MEMORY.md**：

```typescript
// extensions/memory-core/src/short-term-promotion.ts:1570-1649
const memoryPath = path.join(workspaceDir, "MEMORY.md");

// 重水化：从活文件重新读取，避免提升已删除内容
const rehydrated = await rehydratePromotionCandidate(workspaceDir, candidate);

// 构建提升段落
const section = buildPromotionSection(toAppend, nowMs, timezone);

// 追加到 MEMORY.md
await fs.writeFile(memoryPath,
  `${header}${withTrailingNewline(existingMemory)}${section}`,
  "utf-8"
);
```

**输出到 DREAMS.md（Dream Diary）**：

```typescript
// extensions/memory-core/src/dreaming-narrative.ts
// 使用 subagent 生成 Dream Diary narrative
await generateAndAppendDreamNarrative({
  subagent: params.subagent,
  workspaceDir: params.workspaceDir,
  data: { phase, snippets, themes },
  nowMs,
  timezone,
  model: params.config.execution?.model,
});
```

#### 配置结构

```typescript
// extensions/memory-core/openclaw.plugin.json:34-194
type DreamingConfig = {
  enabled: boolean;             // 是否启用
  frequency: string;            // cron 表达式（如 "0 3 * * *"）
  timezone: string;             // 时区
  model: string;                // subagent model（可选）
  phases: {
    light: { enabled, lookbackDays, limit, dedupeSimilarity };
    deep: { enabled, limit, minScore, minRecallCount, minUniqueQueries, recencyHalfLifeDays };
    rem: { enabled, lookbackDays, limit, minPatternStrength };
  };
  storage: { mode: "inline" | "separate" | "both"; separateReports: boolean };
};
```

#### 与其他模块的关系

| 模块 | 关系 |
|------|------|
| memory-core | **绑定**：Dreaming 是 memory-core 的功能，依赖 MEMORY.md |
| memory-lancedb | **不兼容**：memory-lancedb 没有 MEMORY.md，无法使用 Dreaming |
| Active Memory | **独立**：Active Memory 检索现有记忆，Dreaming 创建长期记忆 |
| Commitments | **独立**：完全不同的功能路径 |
| Cron System | **依赖**：通过 cron job 定时触发 |
| Heartbeat | **依赖**：heartbeat 时检查 pending dreaming event |

#### 代码位置总结

| 功能 | 代码位置 |
|------|----------|
| Dreaming 入口 | `extensions/memory-core/src/dreaming.ts` |
| Cron Job 管理 | `dreaming.ts:408-491` |
| Hook 注册 | `dreaming.ts:866-926` |
| Phase 执行 | `extensions/memory-core/src/dreaming-phases.ts:1751-1794` |
| Deep 提升逻辑 | `extensions/memory-core/src/short-term-promotion.ts:1551-1649` |
| Dream Diary | `extensions/memory-core/src/dreaming-narrative.ts` |
| 配置声明 | `extensions/memory-core/openclaw.plugin.json:34-194` |

---

## 三、架构层次与代码对应总结

| 层次 | 架构概念 | 代码位置 | 注册方式 |
|------|----------|----------|----------|
| 插件核心 | OpenClawPluginApi | `src/plugins/types.ts:2527` | Host 注入 |
| 槽位选择 | plugins.slots.memory | `src/plugins/slots.ts:12-20` | 配置驱动 |
| Memory 插件 | memory-core, memory-lancedb | `extensions/*/` | `definePluginEntry` + `kind: "memory"` |
| 工具注册 | registerTool | `src/plugins/types.ts:2556` | `api.registerTool(tool)` |
| Hook 注册 | registerHook / api.on | `src/plugins/types.ts:2560` | `api.on("hook_name", handler)` |
| 搜索后端 | builtin / qmd | `extensions/memory-core/src/memory/` | `memory.backend` 配置 |
| Dreaming | memory-core 功能 | `extensions/memory-core/src/dreaming.ts` | cron job + hooks |
| Flush | memory-core 功能 | `extensions/memory-core/src/flush-plan.ts` | compaction 触发 |
| Enhancement | Active Memory, Commitments | `extensions/active-memory/`, `src/commitments/` | hooks / enqueue |
| SDK 接口 | MemorySearchManager | `packages/memory-host-sdk/src/host/types.ts:85` | 类型契约 |

---

## 四、批判性分析

### 4.1 设计优点

#### 1. 槽位系统清晰实现互斥选择

```typescript
// src/plugins/slots.ts
// kind → slot 映射明确
const SLOT_BY_KIND = { memory: "memory", "context-engine": "contextEngine" };
// 选择时自动禁用其他同类插件
```

**优点**：配置层面解决互斥，避免运行时冲突。

#### 2. 插件 API 界面清晰

```typescript
// src/plugins/types.ts:2527-2650
type OpenClawPluginApi = {
  registerTool: (tool, opts?) => void;
  registerHook: (events, handler, opts?) => void;
  // ...其他注册方法
};
```

**优点**：单一入口注入，注册方法命名一致。

#### 3. Hook 类型丰富且类型安全

```typescript
// src/plugins/hook-types.ts
// before_prompt_build, agent_end, gateway_start, heartbeat...
// 每种 hook 有独立的 Event/Result 类型
```

**优点**：生命周期覆盖完整，类型安全。

### 4.2 设计问题

#### 1. Memory 插件概念混淆

**问题**：文档和配置中 "memory backend" 概念覆盖了两层：
- memory-core vs memory-lancedb（并列插件）
- builtin vs QMD（memory-core 内的后端）

**代码体现**：

```typescript
// src/plugins/slots.ts：memory 插件互斥
// memory-core: kind="memory" → slot "memory"
// memory-lancedb: kind="memory" → slot "memory"

// extensions/memory-core/src/tools.ts：后端选择
const resolved = resolveMemoryBackendConfig({ cfg, agentId });
// backend = "builtin" 或 "qmd"
```

**混淆来源**：两层都叫 "backend"，但层级不同。

#### 2. Dreaming 与 memory-core 绑定过紧

**问题**：Dreaming 是 memory-core 的功能，依赖 MEMORY.md 的存在，无法与其他 Memory 插件配合。

**代码体现**：

```typescript
// extensions/memory-core/src/short-term-promotion.ts:1570
const memoryPath = path.join(workspaceDir, "MEMORY.md");
// Deep Phase 直接写入 MEMORY.md

// 如果使用 memory-lancedb，没有 MEMORY.md
// Dreaming 无法工作
```

**设计限制**：

| Memory 插件 | Dreaming 支持 |
|-------------|---------------|
| memory-core | ✓ 支持（依赖 MEMORY.md） |
| memory-lancedb | ✗ 不支持（无 MEMORY.md） |

**问题根源**：Dreaming 应该是 Memory 插件的**可选能力接口**，而非 memory-core 的私有功能。

#### 3. Enhancement Layer 无统一接口

**问题**：不同 Enhancement 模块加入方式不同：

| 模块 | 加入方式 | 依赖 |
|------|----------|------|
| Active Memory | Hook 注册 | 依赖 Memory 插件工具 |
| Commitments | enqueue + heartbeat | 无 Memory 依赖 |
| Memory Wiki | 独立插件 + supplement | 与 Memory 插件并存 |

**代码体现**：

```typescript
// Active Memory: 直接 hook
api.on("before_prompt_build", handler);

// Commitments: 回复后 enqueue
enqueueCommitmentExtraction({ ... });
// 完全不同的路径

// Memory Wiki: corpus supplement 注册
listMemoryCorpusSupplements().push(registration);
```

**问题**：没有统一的 "Enhancement" 接口或契约。

#### 4. 槽位系统扩展性有限

**问题**：当前只有 `memory` 和 `contextEngine` 两个槽位。

```typescript
// src/plugins/slots.ts:12
const SLOT_BY_KIND: Record<PluginKind, PluginSlotKey> = {
  memory: "memory",
  "context-engine": "contextEngine",
};
```

**限制**：
- 新增槽位需要修改核心代码
- 不能动态定义新槽位
- Enhancement Layer 无法用槽位管理

#### 5. Hook 执行顺序和优先级不透明

**问题**：多个插件注册同一 hook 时，执行顺序依赖注册顺序。

```typescript
// src/plugins/hooks.ts
// 多个 before_prompt_build handlers
// 顺序取决于插件加载顺序
```

**风险**：
- memory-lancedb 和 active-memory 都注册 `before_prompt_build`
- 执行顺序不确定
- 可能产生重复注入或冲突

#### 6. Dreaming 触发机制复杂

**问题**：Dreaming 有两种触发路径（cron + heartbeat），增加了理解和维护复杂度。

**代码体现**：

```typescript
// extensions/memory-core/src/dreaming.ts:888
// heartbeat 触发检查
const isManagedHeartbeatTrigger =
  ctx.trigger === "heartbeat" && hasPendingManagedDreamingCronEvent(ctx.sessionKey);

// cron 触发检查
const isManagedCronTrigger = ctx.trigger === "cron";

// 两者的触发条件不同，但最终执行相同逻辑
```

**问题分析**：

| 触发类型 | 检查逻辑 | 复杂度 |
|----------|----------|--------|
| Cron | `ctx.trigger === "cron"` + token 检查 | 简单 |
| Heartbeat | `hasPendingManagedDreamingCronEvent(sessionKey)` | 复杂（需要 peek system events） |

**根源**：heartbeat 触发是为了处理 cron event 在 base session 队列中等待的情况，但这导致了两种触发路径的并存。

### 4.3 可优化方向

#### 1. 明确概念命名

| 当前命名 | 建议命名 | 层级 |
|----------|----------|------|
| memory backend (builtin/qmd) | memory-core search backend | memory-core 内部 |
| memory backend (memory-core/lancedb) | memory plugin slot | 插件层 |

**建议**：文档和配置键名区分层级：
```json5
{
  plugins: {
    slots: {
      memory: "memory-core"  // Memory Plugin Slot
    }
  },
  // memory-core 内部后端
  memorySearch: {
    backend: "builtin"  // Search Backend
  }
}
```

#### 2. 定义 Enhancement 契约

**建议**：增加 `PluginEnhancementKind` 和统一接口：

```typescript
type PluginEnhancementKind = "memory-recall" | "memory-capture" | "follow-up";

type PluginEnhancementApi = {
  registerMemoryEnhancement: (enhancement: MemoryEnhancement) => void;
  // Enhancement 可声明依赖的 memory plugin slot
  dependsOnMemorySlot?: boolean;
};
```

**好处**：
- Enhancement 模块有统一注册路径
- 可管理 Enhancement 与 Memory 插件的依赖关系
- 更清晰的架构图

#### 3. 抽象 Dreaming 为 Memory 插件能力接口

**问题**：当前 Dreaming 是 memory-core 私有功能，无法与其他 Memory 插件配合。

**建议**：定义 Memory 插件的**可选能力接口**：

```typescript
// 建议的 Memory Plugin Capability 接口
type MemoryPluginCapabilities = {
  // 存储能力
  storage: {
    type: "file" | "vector-db" | "cloud";
    hasLongTermMemoryFile: boolean;  // 是否有 MEMORY.md 等长期文件
  };

  // Dreaming 能力（可选）
  dreaming?: {
    enabled: boolean;
    phases: ("light" | "rem" | "deep")[];
    writeTo: "MEMORY.md" | "vector-db" | "cloud";
  };
};

// Memory 插件声明能力
// memory-core: { storage: { type: "file", hasLongTermMemoryFile: true }, dreaming: {...} }
// memory-lancedb: { storage: { type: "vector-db", hasLongTermMemoryFile: false }, dreaming: undefined }
```

**好处**：

| Memory 插件 | 当前 Dreaming | 建议后 |
|-------------|---------------|--------|
| memory-core | 绑定在插件内 | capability 接口暴露 |
| memory-lancedb | 不支持 | 可声明自己的 Dreaming 实现（写入 vector-db） |
| 未来插件 | 需要复刻 memory-core 代码 | 只需实现 capability 接口 |

#### 4. 简化 Dreaming 触发机制

**建议**：统一为 cron-only 触发，heartbeat 只用于检查 cron job 健康状态。

#### 5. Hook 优先级系统

**建议**：hook 注册时支持优先级：

```typescript
api.on("before_prompt_build", handler, {
  priority: 10  // 数值越小优先级越高
});
```

#### 6. 槽位动态扩展

**建议**：槽位定义改为插件声明的一部分：

```json
{
  "id": "my-plugin",
  "kind": "memory",
  "slot": {
    "name": "memory",
    "exclusive": true
  }
}
```

---

## 五、架构演进建议

### 5.1 当前架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Host Core                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ OpenClawPluginApi                                    │   │
│  │   registerTool, registerHook, slots, ...             │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              ↓ 注入
┌─────────────────────────────────────────────────────────────┐
│                    Plugins                                   │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐    │
│  │ memory-core   │  │memory-lancedb │  │active-memory  │    │
│  │ kind: memory  │  │ kind: memory  │  │ (Enhancement) │    │
│  └───────────────┘  └───────────────┘  └───────────────┘    │
│         ↓ 槽位互斥              ↓ 槽位互斥                    │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 建议架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Host Core                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ PluginRegistry                                       │   │
│  │   ├── SlotManager (可扩展槽位)                        │   │
│  │   ├── HookRunner (优先级系统)                         │   │
│  │   ├── EnhancementRegistry (统一契约)                  │   │
│  │   └────────── CapabilityRegistry (插件能力接口)       │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              ↓ 注入
┌─────────────────────────────────────────────────────────────┐
│                    Plugins                                   │
│                                                              │
│  Memory Slot:                     Enhancements:              │
│  ┌───────────────┐  ┌───────────────┐  ┌─────────────────┐  │
│  │ memory-core   │  │memory-lancedb │  │active-memory   │  │
│  │ capabilities: │  │ capabilities: │  │ kind: memory-* │  │
│  │  - storage:   │  │  - storage:   │  └─────────────────┘  │
│  │    file       │  │    vector-db  │          │            │
│  │  - dreaming:  │  │  - dreaming:  │          │            │
│  │    file-based │  │    (可选)     │          │            │
│  └───────────────┘  └───────────────┘          │            │
│         互斥            capability 接口        │            │
│                              │                  │            │
│                              ↓                  ↓            │
│                    cron trigger           enhancement 契约   │
│                                                              │
│  其他 Slot:                                                  │
│  ┌───────────────┐                                          │
│  │ context-engine│                                          │
│  └───────────────┘                                          │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 关键改进点

| 改进 | 当前 | 建议 |
|------|------|------|
| 槽位定义 | 硬编码在 core | 插件声明中定义 |
| Enhancement | 无统一接口 | 定义 `PluginEnhancementKind` |
| Hook 顺序 | 注册顺序 | 支持优先级参数 |
| 概念命名 | backend 混用 | 区分 slot / backend / capability |
| Dreaming | memory-core 私有功能 | Memory Plugin Capability 接口 |
| Dreaming 触发 | cron + heartbeat 双路径 | cron-only，heartbeat 健康检查 |

---

## 六、总结

### 代码与架构对应良好之处

- 槽位系统清晰实现插件互斥
- OpenClawPluginApi 提供一致的注册入口
- Hook 类型覆盖完整生命周期
- Dreaming 三阶段设计合理（Light → REM → Deep）

### 需要改进之处

- "backend" 概念层级混淆
- Enhancement Layer 缺乏统一契约
- 槽位系统扩展性有限
- Hook 执行顺序不透明
- **Dreaming 与 memory-core 绑定过紧**
- **Dreaming 触发机制复杂**

### 复刻建议

如果要复刻此系统：

1. **必须实现**：
   - OpenClawPluginApi 接口 + 槽位选择逻辑
   - Memory 插件的存储能力（file 或 vector-db）

2. **建议增强**：
   - Hook 优先级系统
   - Enhancement 契约
   - Memory Plugin Capability 接口（抽象 Dreaming）

3. **命名规范**：
   - slot：插件层互斥选择
   - backend：插件内部实现选择
   - capability：插件可选能力接口

4. **Dreaming 复刻要点**：
   - 三阶段模型：Light（暂存）→ REM（反思）→ Deep（提升）
   - 阈值门控：minScore=**0.75**、minRecallCount=**3**、minUniqueQueries=**2**
   - 重水化：提升前从活文件读取，避免提升已删除内容
   - 人类可审查：输出到 DREAMS.md
   - 触发机制：建议只用 cron，简化 heartbeat 路径

