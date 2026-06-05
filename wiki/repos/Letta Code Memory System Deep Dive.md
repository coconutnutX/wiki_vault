---
type: repo
title: "Letta Code Memory System Deep Dive"
updated: 2026-06-01
tags: [letta, memory, memfs, reflection, compaction, recall, dream, agent]
---

# Letta Code 记忆系统深度调研

> 仓库路径：`/data/Workspace2/repos/letta-code`
> 基于 2026-06-01 的源码，核心关注三个问题：记忆的定义与存储、检索与注入、dream (reflection) 机制。

---

## 1. 什么被当成"记忆"？粒度、层级、存储

### 1.1 记忆的定义：三层体系

Letta Code 的记忆不是单一概念，而是由三个显式层级构成：

| 层级 | 名称 | 内容 | 读写权限 | 是否始终注入上下文 |
|------|------|------|----------|-------------------|
| **In-context (System Prompt Blocks)** | 记忆块 (Memory Blocks) | persona、human、project 等身份/偏好/约定 | 可写（Agent 主动编辑） | ✅ 始终在 system prompt 中 |
| **External Memory (MemFS files)** | 外部记忆 | 技能(SKILL.md)、参考文档、项目细节 | 可写 | ❌ 按需读取，仅在 system prompt 中展示目录树 + description |
| **Experience (Recall Memory)** | 对话历史 | 所有 past conversations | 不可写（自动存储） | ❌ 通过 recall 子代理搜索 |

代码定义：[`src/agent/memory.ts:13-27`](src/agent/memory.ts#L13-L27)

```ts
export const GLOBAL_BLOCK_LABELS = ["persona", "human"] as const;
export const PROJECT_BLOCK_LABELS = [] as const;
export const MEMORY_BLOCK_LABELS = [...GLOBAL_BLOCK_LABELS, ...PROJECT_BLOCK_LABELS] as const;
```

**关键发现**：`PROJECT_BLOCK_LABELS` 目前为空数组——原 `skills/loaded_skills` 已在 LET-7353 中移除，技能现在通过 system reminders 注入而非作为记忆块。

#### 三层详解

**层1：Memory Blocks（核心记忆 / 系统提示记忆）**

这是 Agent 的"自我"所在。每个 block 是 system prompt 中的一个可编辑段落：

- `persona` — Agent 的身份、价值观、行为准则（跨项目共享）
- `human` — 关于用户的知识（跨项目共享）
- 其他 block 可由 Agent 或 reflection 创建

Block 存储在 MemFS 的 `system/` 目录下，文件格式为 `.md`（Markdown + YAML frontmatter）。

**层2：External Memory（外部记忆 / 按需记忆）**

MemFS 中 `system/` 目录之外的所有 `.md` 文件：

- `skills/` — 程序性记忆（技能）
- `reference/`、`project/` 等自定义目录 — 参考材料、项目知识
- 根目录散落文件 — 如 `notes.md`

这些文件**不在 system prompt 中注入正文**，仅在 prompt 中注入目录树和 description，Agent 需主动用 `Read` / `Bash` 工具读取。

**层3：Recall Memory（经验 / 对话历史）**

所有对话消息自动持久化到 Letta 后端数据库。Agent 不能修改，只能通过 recall 子代理搜索。搜索模式包括 hybrid（向量+全文 RRF）、vector、fts 三种。

[`src/agent/prompts/recall_subagent.md:27-31`](src/agent/prompts/recall_subagent.md#L27-L31)

```
- hybrid (default): Combines vector similarity + full-text search with RRF scoring
- vector: Semantic similarity search (good for conceptual matches)
- fts: Full-text search (good for exact phrases)
```

### 1.2 记忆粒度：一个文件 = 一个主题

每个记忆文件（`.md`）是一个**自含的语义单元**，而不是一条条原子事实。格式：

```
---
description: 文件用途的一句话描述
read_only: true/false  (可选)
---

Markdown 正文
```

- 文件名 = 记忆标签 (label)：`system/persona.md` → label `system/persona`
- description 字段用于 Agent 检索决策（"这个文件跟我现在的问题相关吗？"）
- 文件可嵌套在多级目录中：`project/tooling/bun.md` → label `project/tooling/bun`

[`src/tools/impl/memory.ts:509-543`](src/tools/impl/memory.ts#L509-L543) 中 `parseMemoryFile` 强制要求 frontmatter 包含 `description`：

```ts
if (!description || !description.trim()) {
  throw new Error("memory: target file frontmatter is missing 'description'");
}
```

`src/agent/prompts/remember.md` 明确指导 Agent 判断粒度：

> Be concise - distill the information to its essence
> Avoid duplicates - check if similar information already exists
> Match existing formatting of memory blocks

### 1.3 记忆层级结构：`system/` vs 外部

MemFS 的目录结构本身就是层级：

```
memory/
├── system/           ← 始终在上下文中（In-context）
│   ├── persona.md    ← Agent 身份
│   ├── human.md      ← 用户画像
│   └── ...           ← 其他核心上下文
├── skills/           ← 程序性记忆
├── reference/        ← 参考材料
├── project/          ← 项目知识
└── notes.md          ← 顶层散落
```

**层级规则**（reflection 子代理指令 [`builtin/reflection.md:23-29`](src/agent/subagents/builtin/reflection.md#L23-L29)）：

> - **Prompts** (`system/`): Always in-context. Reserve for identity, preferences, conventions.
> - **Skills** (`skills/`): Procedural memory.
> - **External memory** (everything else): Reference material retrieved on-demand.

Agent 可以把文件从 `system/` 移到外部（降低为按需可见），或反过来（提升为始终可见）。这类似于人类记忆的"工作记忆 vs 长期存储"切换。

### 1.4 存储位置与格式

**存储位置**：

| 场景 | 存储路径 |
|------|----------|
| Cloud backend (Letta API) | `~/.letta/agents/{agentId}/memory/`（git 仓库，push 到 Letta 云端） |
| Local backend | 由 `LETTA_LOCAL_BACKEND_DIR` 配置或默认存储目录 |

[`src/agent/memory-filesystem.ts:43-54`](src/agent/memory-filesystem.ts#L43-L54)

```ts
export function getMemoryFilesystemRoot(agentId: string, homeDir = homedir()): string {
  return join(homeDir, MEMORY_FS_ROOT, MEMORY_FS_AGENTS_DIR, agentId, MEMORY_FS_MEMORY_DIR);
}
// 即 ~/.letta/agents/{agentId}/memory/
```

**存储格式**：Git 仓库中的 Markdown 文件（`.md`），每个文件带 YAML frontmatter。

**Git-backed 同步机制**：

- 本地写入 → `git commit + git push`（cloud）或 `git commit`（local）
- 多 Agent 并发写入时使用 replay 机制：若 push 失败（远端有更新），先 pull，再在远端最新之上 replay 本地改动

[`src/agent/memory-git.ts`](src/agent/memory-git.ts) 中的 `commitAndSyncMemoryWrite` 实现了完整的冲突解决流程。

**`memory_filesystem` block**：这是一个特殊的 read-only block（[`src/agent/memory-constants.ts:1`](src/agent/memory-constants.ts#L1)），内容为 MemFS 的目录树视图，不可通过 memory 工具修改。

---

## 2. 检索策略、注入方式、Token 预算控制

### 2.1 检索策略：全量读取 + 按需探测

Letta 的记忆检索**不是向量搜索/图检索**。它用的是一种"全量展示骨架 + 按需深读"的策略：

#### System Prompt Blocks（层1）— 全量注入

所有 `system/` 下的文件内容**完整注入**到 system prompt 中，无需"检索"。因为它们被设计为 Agent 每次醒来都必须知道的"自我"。

#### External Memory（层2）— 目录树 + description 注入，正文按需读取

System prompt 中注入的不是文件正文，而是：

1. **目录树**（带 description）—— 让 Agent 知道有哪些文件可读
2. **`<projection>` 标签**—— 指示文件在 `$MEMORY_DIR` 中的路径

[`src/backend/local/system-prompt-compilation.ts:116-162`](src/backend/local/system-prompt-compilation.ts#L116-L162) 中 `renderExternalProjection` 和 `renderMemfsProjection` 的实现：

```ts
function renderMemfsProjection(memoryDir: string): { content: string; revision?: string } {
  // 1. persona → <self> 标签，完整注入
  // 2. system/ 其他文件 → <memory> 标签下的 XML 树结构，完整注入
  // 3. 非system文件 → <external_projection> 树结构（只有路径+description，无正文）
}
```

注入的 XML 结构示例：

```xml
<self>
  <projection>$MEMORY_DIR/system/persona.md</projection>
  [persona正文]
</self>

<memory>
  <human>
    <projection>$MEMORY_DIR/system/human.md</projection>
    <description>What I'm learning about the person...</description>
    [human正文]
  </human>
</memory>

<external_projection>
  $MEMORY_DIR/
  ├── system/
  │   ├── persona.md
  │   └── human.md
  ├── skills/
  │   └── using-slack/
  │       └── SKILL.md (procedural memory for Slack integration)
  └── reference/
  │   └── api.md (API reference documentation)
</external_projection>
```

#### Recall Memory（层3）— 通过子代理搜索

Agent 不直接搜索对话历史。它通过 `recall` 子代理调用 `letta messages search` CLI 命令。搜索由 Letta 后端 API 提供，支持向量搜索、全文搜索、混合搜索。

[`src/agent/prompts/recall_subagent.md`](src/agent/prompts/recall_subagent.md) 定义了搜索策略：

```
Strategy 1: Needle + Expand (Recommended)
  → search → get message_id → list before/after → 展开上下文

Strategy 2: Date-Bounded Search
  → 按时间范围过滤

Strategy 3: Semantic Search
  → vector mode for vague topics
```

### 2.2 注入方式：System Prompt 内嵌 XML

记忆注入到 system prompt 中，使用**XML 标签**作为语义 slot：

[`src/backend/local/system-prompt-compilation.ts:291-296`](src/backend/local/system-prompt-compilation.ts#L291-L296)

```ts
function injectCoreMemory(rawSystemPrompt: string, coreMemory: string): string {
  const prompt = rawSystemPrompt.includes(CORE_MEMORY_VARIABLE)
    ? rawSystemPrompt
    : `${rawSystemPrompt.trimEnd()}\n\n${CORE_MEMORY_VARIABLE}`;
  return prompt.replaceAll(CORE_MEMORY_VARIABLE, coreMemory);
}
```

System prompt 模板中包含 `{CORE_MEMORY}` 占位符，编译时替换为所有记忆内容的 XML 渲染。

**注入格式总结**：

| 记忆层级 | 注入位置 | 注入格式 | 注入量 |
|----------|----------|----------|--------|
| persona | `<self>` 标签 | 完整正文 | 全量 |
| system/ 其他 | `<memory>` 标签下的嵌套 XML | 完整正文 + `<projection>` 路径 + `<description>` | 全量 |
| 外部文件 | `<external_projection>` 标签 | 目录树 + description | 仅骨架 |
| recall | 不注入 | Agent 通过 recall 子代理搜索 | 按需 |
| 技能列表 | `<available_skills>` 标签 | 目录树 + description | 仅骨架 |

**记忆元数据也注入**：

[`src/backend/local/system-prompt-compilation.ts:275-289`](src/backend/local/system-prompt-compilation.ts#L275-L289)

```xml
<memory_metadata>
- AGENT_ID: agent-xxx
- CONVERSATION_ID: conv-xxx
- System prompt last recompiled: 2026-06-01 12:00:00 AM UTC+0000
- 42 previous messages between you and the user are stored in recall memory
</memory_metadata>
```

### 2.3 Token 预算控制

#### System Prompt 编译中的预算

没有显式的 token budget 截断机制。system prompt 编译时**全量注入** `system/` 文件内容。预算控制通过以下方式间接实现：

1. **设计约束**：reflection 子代理的指令明确要求"Keep files concise — move verbose content to external memory"
2. **`memory_filesystem` block 是 read-only**，阻止 Agent 滥用树视图空间

[`src/agent/subagents/builtin/reflection.md:23`](src/agent/subagents/builtin/reflection.md#L23):

> Reserve for identity, preferences, conventions, and active project context the agent needs on every turn. Keep files concise — move verbose content to external memory.

#### Reflection 子代理的上下文预算

[`src/agent/subagents/context-budget.ts:11-18`](src/agent/subagents/context-budget.ts#L11-L18)

```ts
export const REFLECTION_STARTUP_CONTEXT_TOKEN_LIMIT = 16_000;
export const REFLECTION_STARTUP_CONTEXT_CHAR_LIMIT = 16_000 * 4; // = 64,000 chars
export const REFLECTION_PARENT_MEMORY_SNAPSHOT_CHAR_LIMIT = 40_000;
```

Reflection 子代理启动时的总上下文预算约 64,000 chars（≈16k tokens），其中父 Agent 的记忆快照最多占 40,000 chars。超出时会截断或省略非核心文件。

[`src/cli/helpers/reflection-transcript.ts:342-350`](src/cli/helpers/reflection-transcript.ts#L342-L350) 中 `buildMemoryPreviewNotice`:

```
[Memory preview truncated: startup context is capped at ~16k estimated tokens.
Full file available at {absolutePath}; read it directly if needed.]
```

#### Compaction（对话压缩）中的预算

[`src/backend/local/compaction.ts`](src/backend/local/compaction.ts) 实现了两种压缩模式：

- **`all`**：全部消息压缩为一条摘要（500 word limit）
- **`sliding_window`**（默认）：保留最近 70% 的消息，压缩前面的 30%（300 word limit）

[`src/backend/local/compaction.ts:36-37`](src/backend/local/compaction.ts#L36-L37)

```ts
export const LOCAL_DEFAULT_COMPACTION_MODE = "sliding_window";
export const LOCAL_DEFAULT_SLIDING_WINDOW_PERCENTAGE = 0.3;
```

#### Memory Reminder Token 控制

[`src/cli/helpers/memory-reminder.ts`](src/cli/helpers/memory-reminder.ts) 控制何时提醒 Agent 检查记忆（但不直接控制注入量）。

---

## 3. Dream（Reflection）机制

Letta Code 不使用"dream"一词，但它的 **reflection 子代理** 在功能上等同于 dream：后台自动运行，审视对话，更新记忆。

### 3.1 触发机制

Reflection 有两种触发方式：

| 触发类型 | 配置值 | 说明 |
|----------|--------|------|
| **step-count** | `reflectionTrigger: "step-count"` | 每隔 N 个 turn 自动触发（默认 25） |
| **compaction-event** | `reflectionTrigger: "compaction-event"`（**默认**） | 每次 compaction（对话压缩）完成后触发 |

[`src/cli/helpers/memory-reminder.ts:18-19`](src/cli/helpers/memory-reminder.ts#L18-L19)

```ts
export type ReflectionTrigger = "off" | "step-count" | "compaction-event";
const DEFAULT_REFLECTION_SETTINGS: ReflectionSettings = {
  trigger: "compaction-event",
  stepCount: DEFAULT_STEP_COUNT, // 25
};
```

**触发流程**：

1. 对话发生 compaction → `buildCompactionMemoryReminder()` 被调用
2. 或每 25 个 turn → `shouldFireStepCountTrigger()` 返回 true
3. → `launchReflectionSubagent()` 在后台启动 reflection 子代理

[`src/cli/helpers/reflection-launcher.ts:93-220`](src/cli/helpers/reflection-launcher.ts#L93-L220) 是核心入口。

**防重入机制**：如果已有 reflection 在运行，新触发会被跳过（[`src/cli/helpers/reflection-launcher.ts:111-116`](src/cli/helpers/reflection-launcher.ts#L111-L116)）。

### 3.2 谁来处理？

Reflection 由一个**专门的子代理**处理，不是硬编码逻辑。

子代理定义：[`src/agent/subagents/builtin/reflection.md`](src/agent/subagents/builtin/reflection.md)

```yaml
name: reflection
description: Background agent that reflects on recent conversations and updates memory files
tools: Bash
model: inherit          # 继承父 Agent 的模型
memoryBlocks: none       # 不继承父 Agent 的记忆块
mode: stateless          # 无状态运行
permissionMode: memory   # 只能操作记忆文件
```

**权限隔离**：reflection 子代理只能操作 `memory` 权限模式的文件——即只能读写 MemFS，不能执行一般命令。

### 3.3 输入是什么？

Reflection 子代理的输入由两部分构成：

**1. 对话 transcript（JSON 消息数组）**

[`src/cli/helpers/reflection-transcript.ts:981-1023`](src/cli/helpers/reflection-transcript.ts#L981-L1023) 中 `buildAutoReflectionPayload`：

- 从 transcript.jsonl 中选取**尚未被 reflect 过的消息段**
- 格式化为 ChatML-style JSON：`[{ role, content, tool_calls }]`
- 系统提示被过滤（去除 `<memory>`、`<self>`、`<human>` 等动态块）

[`src/cli/helpers/reflection-transcript.ts:881-906`](src/cli/helpers/reflection-transcript.ts#L881-L906) 中 `filterSystemPromptForReflection`：

```ts
const tagsToStrip = ["memory", "self", "human", "available_skills", "system-reminder", "memory_metadata"];
```

**2. 父 Agent 的记忆快照**

[`src/cli/helpers/reflection-transcript.ts:351-437`](src/cli/helpers/reflection-transcript.ts#L351-L437) 中 `buildParentMemorySnapshot`：

- 遍历 MemFS 目录，收集所有 `.md` 文件
- 构建 `<parent_memory>` XML：
  - `<memory_filesystem>` 目录树
  - `system/` 文件：完整内容（在 40k char 预算内）
  - 非 system 文件：只有 description（超出预算则截断/省略）

快照注入在 reflection prompt 中：

[`src/cli/helpers/reflection-transcript.ts:77-95`](src/cli/helpers/reflection-transcript.ts#L77-L95)

```ts
export function buildReflectionSubagentPrompt(input: ReflectionPromptInput): string {
  lines.push("Review the conversation transcript and update memory files...");
  lines.push(`The primary agent's memory filesystem is located at: ${input.memoryDir}`);
  lines.push("In-context memory ... are stored in the `system/` folder...");
  if (input.parentMemory) {
    lines.push(input.parentMemory);  // <parent_memory> XML
  }
  return lines.join("\n");
}
```

**Transcript 状态追踪**：

[`src/cli/helpers/reflection-transcript.ts:22-31`](src/cli/helpers/reflection-transcript.ts#L22-L31)

```ts
interface ReflectionTranscriptState {
  schema_version: "v2_message_id";
  reflected_through_message_id?: string;     // 上次 reflect 到哪条消息
  total_completed_turns: number;             // 总完成 turn 数
  reflected_completed_turns: number;         // 已 reflect 的 turn 数
  turns_since_last_successful_reflection: number;
}
```

每次 reflect 成功后，`reflected_through_message_id` 更新为最新消息 ID，下次只 reflect 新增部分。

### 3.4 怎么处理？

Reflection 子代理按 5 个 Phase 执行（[`src/agent/subagents/builtin/reflection.md`](src/agent/subagents/builtin/reflection.md)）：

**Phase 1 — Investigate**：读取当前记忆全景，理解结构。

**Phase 2 — Extract**：从对话中提取候选学习内容，按优先级排序：

1. Mistakes and corrections（错误和纠正）
2. Preferences and patterns（偏好和模式）
3. New durable facts（新的持久事实）
4. Contradictions（矛盾）

然后过滤：
- 短暂的？→ 不存（一次性细节）
- 已有？→ 不存
- 可泛化？→ 才存（"用户偏好短章节" 是持久，"周二编辑了第三章" 不是）
- **"If nothing survives filtering, make no changes."**

**Phase 3 — Update**：精准更新记忆文件：
- 路由到正确的层级（`system/` vs 外部）
- 已有文件涵盖的 → 更新，不新建
- persona/human 文件 → 手术式编辑，绝不全量重写
- **矛盾解决**：新信息与旧信息矛盾时，在源头修正旧信息，不要并排存放两个矛盾版本

**Phase 4 — Review**：检查是否存在过时内容、交叉引用断裂、层级错位。

**Phase 5 — Commit and push**：`git add + commit + push`，提交信息包含 Agent ID 和 Parent Agent ID 溯源。

### 3.5 输出是什么？

Reflection 的输出是：

1. **MemFS 文件的变更**（git commit + push）
2. **一份报告**（Summary、Changes made、Skipped、Commit reference、Issues）

[`src/cli/helpers/reflection-launcher.ts:153-167`](src/cli/helpers/reflection-launcher.ts#L153-L167) 中 `onComplete` 回调：

```ts
onComplete: async ({ success, error, agentId: reflectionAgentId }) => {
  // 1. 更新 transcript state（标记已 reflect 的消息范围）
  await finalizeAutoReflectionPayload(...);
  // 2. 处理 completion（触发 system prompt recompile）
  const completionMessage = await handleMemorySubagentCompletion(...);
}
```

**System Prompt 重编译**：Reflection 成功后，父 Agent 的 system prompt 会被重新编译，以反映 MemFS 中最新的记忆内容。

[`src/cli/helpers/memory-subagent-completion.ts`](src/cli/helpers/memory-subagent-completion.ts) 中 `handleMemorySubagentCompletion` 负责：
- 触发 `recompileAgentSystemPrompt`
- 向用户发送 completion message

### 3.6 冲突如何解决？

Reflection 中有三层冲突解决机制：

#### 1. 矛盾记忆冲突（内容层）

Reflection 子代理指令明确要求：

> **Contradiction resolution**: If new information contradicts an existing memory entry, fix the stale entry at the source. Do not append the new version alongside the old — that leaves two conflicting records. Update or replace the outdated content.

（[`src/agent/subagents/builtin/reflection.md:69`](src/agent/subagents/builtin/reflection.md#L69)）

#### 2. Git 写入冲突（同步层）

当 reflection 子代理的 `git push` 与远端有冲突时：

[`src/agent/memory-git.ts`](src/agent/memory-git.ts) 中的 `commitAndSyncMemoryWrite`：
- push 失败 → pull（获取远端最新）
- 在远端最新之上 **replay** 本地操作
- 若 replay 也产生相同内容 → 跳过（"replayed but no effective change"）
- replay 成功 → push replay 结果

[`src/tools/impl/memory.ts:125-137`](src/tools/impl/memory.ts#L125-L137):

```ts
const commitResult = await commitAndSyncMemoryWrite({
  memoryDir,
  pathspecs: affectedPaths,
  reason,
  author,
  syncMode,
  replay: async () => applyMemoryCommand(memoryDir, args, { replaying: true }),
});
```

#### 3. 并发 Reflection 冲突（调度层）

同一 Agent + 同一对话中，只允许一个 reflection 子代理运行：

[`src/cli/helpers/reflection-launcher.ts:111`](src/cli/helpers/reflection-launcher.ts#L111)

```ts
if (isReflectionSubagentActive(getSubagents(), agentId, conversationId)) {
  return { launched: false, reason: "already_active" };
}
```

### 3.7 Sleeptime Persona（旧版 Dream）

Letta 还保留了一个 `sleeptime.md` persona 模板（[`src/agent/prompts/sleeptime.md`](src/agent/prompts/sleeptime.md)），描述了一种更激进的实时记忆管理模式：

> "I am a sleep-time memory management agent. I observe the conversation between the user and their primary agent, then actively maintain memory blocks..."

这是一个独立的 Agent persona，与当前的 reflection 子代理机制不同。Sleeptime 是实时运行的（边对话边更新记忆），reflection 是后台离线运行的。

---

## 4. 与 OpenG-Memory / OpenClaw 的对比

| 维度 | Letta Code | OpenClaw / oG-Memory |
|------|-----------|----------------------|
| 记忆定义 | 三层：Blocks + MemFS + Recall | Core/Archival/Recall 三层但以 block 为原子单位 |
| 记忆粒度 | 一文件一主题（Markdown），可多级嵌套 | 一 block 一主题（文本段落），扁平 |
| 存储格式 | Git 仓库中的 .md 文件 | Letta Server 上的 block 对象（JSON） |
| 检索策略 | 全量注入 system/ + 目录树 + recall CLI 搜索 | 全量注入 core + 向量搜索 archival |
| 注入方式 | XML 标签 slot（`<self>` `<memory>` `<external_projection>`） | 拼接到 system prompt 文本 |
| Dream 机制 | Reflection 子代理（5 Phase 流水线） | 三个 dream category（reinforce/consolidate/forget） |
| 冲突解决 | 内容层：修正源头 / Git层：replay / 调度层：防重入 | 评分+优先级 |
| Token 预算 | 间接（设计约束 + 16k reflection 预算 + compaction） | 显式截断 |

---

## 5. 关键文件索引

| 文件 | 功能 | 关键行 |
|------|------|--------|
| [`src/agent/memory.ts`](src/agent/memory.ts) | Block 标签定义、默认 block 加载 | L13-27: GLOBAL/PROJECT labels |
| [`src/agent/memory-filesystem.ts`](src/agent/memory-filesystem.ts) | MemFS 目录管理、git 仓库初始化、tree 渲染 | L43-54: 存储路径, L405-556: applyMemfsFlags |
| [`src/agent/memory-git.ts`](src/agent/memory-git.ts) | Git-backed 记忆同步、冲突解决、replay | commitAndSyncMemoryWrite: 全文件 |
| [`src/tools/impl/memory.ts`](src/tools/impl/memory.ts) | memory 工具实现（str_replace/insert/delete/create/rename） | L102-169: 核心逻辑, L509-543: frontmatter 解析 |
| [`src/tools/impl/memory-apply-patch.ts`](src/tools/impl/memory-apply-patch.ts) | 批量 patch 操作 | 全文件 |
| [`src/backend/local/system-prompt-compilation.ts`](src/backend/local/system-prompt-compilation.ts) | System prompt 编译：记忆 → XML → 注入 | L220-262: renderMemfsProjection, L291-296: injectCoreMemory |
| [`src/backend/local/compaction.ts`](src/backend/local/compaction.ts) | 对话压缩（sliding_window / all） | L36-37: 默认模式, L482-548: plan 滑窗 |
| [`src/cli/helpers/reflection-launcher.ts`](src/cli/helpers/reflection-launcher.ts) | Reflection 子代理启动入口 | L93-220: launchReflectionSubagent |
| [`src/cli/helpers/reflection-transcript.ts`](src/cli/helpers/reflection-transcript.ts) | Transcript 管理、payload 构建、state 追踪 | L77-95: prompt构建, L351-437: 父记忆快照, L881-906: 系统提示过滤, L981-1023: payload构建 |
| [`src/cli/helpers/memory-reminder.ts`](src/cli/helpers/memory-reminder.ts) | Reflection 触发配置 | L18-19: ReflectionTrigger 类型, L38-40: 默认设置 |
| [`src/agent/subagents/context-budget.ts`](src/agent/subagents/context-budget.ts) | Reflection 上下文预算 | L11-18: 16k token / 40k char limits |
| [`src/agent/subagents/builtin/reflection.md`](src/agent/subagents/builtin/reflection.md) | Reflection 子代理 system prompt（5 Phase 流水线） | 全文件 |
| [`src/agent/subagents/builtin/recall.md`](src/agent/subagents/builtin/recall.md) | Recall 子代理定义 | 全文件 |
| [`src/agent/subagents/builtin/memory.md`](src/agent/subagents/builtin/memory.md) | Memory defrag 子代理定义 | 全文件 |
| [`src/agent/prompts/letta.md`](src/agent/prompts/letta.md) | 主 Agent system prompt（记忆架构描述） | 全文件 |
| [`src/agent/prompts/sleeptime.md`](src/agent/prompts/sleeptime.md) | Sleeptime persona（旧版实时记忆管理） | 全文件 |
| [`src/agent/prompts/persona_memo.mdx`](src/agent/prompts/persona_memo.mdx) | "Memo" persona 模板 | 全文件 |
| [`src/agent/prompts/human_memo.mdx`](src/agent/prompts/human_memo.mdx) | "Memo" human 模板 | 全文件 |
| [`src/agent/prompts/remember.md`](src/agent/prompts/remember.md) | `/remember` 命令提示 | 全文件 |
| [`src/agent/prompts/memory_check_reminder.txt`](src/agent/prompts/memory_check_reminder.txt) | 周期性记忆检查提醒 | 全文件 |
| [`src/agent/personality.ts`](src/agent/personality.ts) | 人格选项与记忆 block 初始化 | L33-70: PERSONALITY_OPTIONS, L618-707: applyPersonalityToMemory |
| [`src/agent/memory-constants.ts`](src/agent/memory-constants.ts) | Read-only block 标签 | L1: READ_ONLY_BLOCK_LABELS |
| [`src/agent/memory-runtime.ts`](src/agent/memory-runtime.ts) | 运行时 MemFS 状态检查 | L13-24: isActiveMemfsEnabled |