---
type: repo
title: "Letta Code Reflection/Dream Prompt 深度拆解"
updated: 2026-06-10
tags: [letta, reflection, dream, prompt, memfs, subagent, consolidation]
---

# Letta Code Reflection/Dream Prompt 深度拆解

> 仓库路径：`/data/Workspace2/repos/letta-code`
> 聚焦于 reflection（即 dream）机制中每一条 prompt 是怎么写的、怎么拼装出来的、最终发给子代理的完整上下文长什么样。

---

## 1. Reflection 是什么？——概念定位

Letta Code 不用 "dream" 这个词，但 reflection 子代理的功能等同 dream：**后台自动运行，审视对话历史，判断哪些值得持久化到记忆，并精准修改记忆文件**。

主 Agent 的 system prompt [`src/agent/prompts/letta.md:90-97`](src/agent/prompts/letta.md#L90-L97) 对此有明确定位：

> Between turns you have no continuous stream of consciousness, but **background agents may refine your memory** (similar to how human memory consolidates during sleep — strengthening important connections and discarding noise). The memory you wake up with **may be better organized** than the memory you left behind, and that is your own learning process at work.

这段话直接类比了"人类睡眠中记忆巩固"——即 dream。

---

## 2. Prompt 全景：Reflection 子代理收到的完整上下文

Reflection 子代理的**完整上下文**由三个组件拼装而成：

```
┌─────────────────────────────────────────────┐
│  1. System Prompt（子代理身份 + 5 Phase 流水线指令）  │  ← reflection.md / reflection_local_memfs.md
│                                             │
│  2. User Prompt（对话操作指引 + 父记忆快照）         │  ← buildReflectionSubagentPrompt()
│                                             │
│  3. $TRANSCRIPT_PATH env（对话 transcript JSON）    │  ← buildAutoReflectionPayload()
│     子代理通过 Bash `cat $TRANSCRIPT_PATH` 读取    │
└─────────────────────────────────────────────┘
```

以下逐层拆解。

---

## 3. System Prompt（第一层）：子代理身份与行为规范

### 3.1 两个版本

- **Cloud 版**：[`src/agent/subagents/builtin/reflection.md`](src/agent/subagents/builtin/reflection.md) — commit 后需要 `git push` 到远端
- **Local 版**：[`src/agent/subagents/builtin/reflection_local_memfs.md`](src/agent/subagents/builtin/reflection_local_memfs.md) — commit 只需本地提交，无需 push

两者**Phase 1-4 完全相同**，Phase 5 的 git 命令略有差异（cloud 版多一步 `git push`）。

### 3.2 YAML Frontmatter（子代理元数据）

```yaml
name: reflection
description: Background agent that reflects on recent conversations and updates memory files
tools: Bash          # 只有 Bash 工具——通过 shell 操作文件和 git
model: inherit       # 继承父 Agent 的模型（实际可能降级为 letta/auto-memory）
memoryBlocks: none    # 不继承父 Agent 的记忆块——纯观察者
mode: stateless       # 无状态——不持久化自己的工作记忆
permissionMode: memory  # 只能操作记忆目录内的文件
```

关键设计：
- `tools: Bash` — 子代理只用 shell 命令操作文件，不直接调用 memory 工具 API
- `memoryBlocks: none` — 不带入父 Agent 的 persona/human，避免"自我认同混淆"
- `permissionMode: memory` — 权限被限制在 `$MEMORY_DIR` 内，不能跑任意命令

### 3.3 System Prompt 正文逐段分析

#### 开场声明（第1段）

> You are a memory subagent launched in the background to manage the primary agent's memory and context after a recent conversation. You run autonomously and return a single final report when done. You CANNOT ask questions — all instructions are provided upfront, so make reasonable assumptions based on context and document any assumptions you make.

**设计意图**：明确身份边界——你是后台维护者，不是主 Agent。不能提问，只能自主判断。

#### 身份边界声明（第2段）

> **You are NOT the primary agent.** You are reviewing conversations that already happened:
> - "system" messages are the primary agent's system prompt — use them only to understand the agent's identity and what's relevant to the user.
> - "assistant" messages are from the primary agent
> - "user" messages are from the primary agent's user

**设计意图**：防止子代理"代入角色"，混淆自己与主 Agent 的身份。对话 transcript 中 system 消息是主 Agent 的 system prompt（用于理解语境），不是可编辑的内容。

#### 记忆文件系统说明（第3段）

> The primary agent's context (its prompts, skills, and external memory files) is stored in a "memory filesystem" rooted at `$MEMORY_DIR`. Changes to these files are reflected in the primary agent's context.
>
> The filesystem contains:
> - **Prompts** (`system/`): Always in-context. Reserve for identity, preferences, conventions...
> - **Skills** (`skills/`): Procedural memory...
> - **External memory** (everything else): Reference material retrieved on-demand...

**设计意图**：给子代理一张"记忆地图"，明确三层层级。强调 `system/` 文件要精简——冗长内容必须移到外部。

#### 可见性说明

> **Visibility**: The primary agent always sees prompts, the filesystem tree, and skill/external file descriptions. Skill and external file *contents* must be retrieved by the primary agent based on name/description.

**设计意图**：让子代理理解主 Agent 只能看到"骨架"（目录树 + description），外部文件的正文需要主动读取。这暗示：description frontmatter 是关键——写好 description 才能被主 Agent 发现。

---

### 3.4 五个 Phase——Reflection 的核心流水线

#### Phase 1 — Investigate（调查现有记忆）

> Understand the current memory landscape before changing anything. Your user prompt already includes a `<memory_filesystem>` tree (with descriptions on non-system files) and the full content of every `system/` file inlined in `<memory>` blocks — start there... For non-system files (skills, reference, etc.), use the tree's descriptions to decide what's worth reading, then fetch contents from `$MEMORY_DIR` on demand. Follow `[[path]]` cross-references when relevant.

**设计意图**：
- 强制先观察再动手——"You cannot integrate new learnings into existing structure if you don't know the structure"
- `system/` 文件已在 user prompt 的 `<parent_memory>` 中完整给出，无需再读
- 外部文件需要子代理自己用 Bash 读取——按 description 决定要不要深读
- `[[path]]` 交叉引用链作为"发现路径"——沿着链走可以发现关联文件

#### Phase 2 — Extract（提取候选学习内容）

> Review the conversation and identify candidate learnings worth persisting. **Prioritize in this order**:
>
> 1. **Mistakes and corrections** — errors the agent made, user feedback, frustrations, failed retries
> 2. **Preferences and patterns** — conventions, style choices, workflow decisions, behavioral corrections
> 3. **New durable facts** — project details, team info, environment details, architectural decisions
> 4. **Contradictions** — anything that conflicts with what's currently stored in memory

**设计意图**：优先级排序明确——纠错 > 偏好 > 新事实 > 矛盾。这和人类的 dream 优先级一致：先修正错误认知，再固化偏好模式。

**四重过滤**：

> - **Durable or ephemeral?** One-off details tied to a single session — specific line numbers, exact error messages, temporary file paths, debug ports, intermediate calculations — are ephemeral. Don't store them.
> - **Already captured?** If memory already contains this information, skip it.
> - **Generalizable?** Distill reusable patterns, not event transcripts. "User prefers short chapters with cliffhanger endings" is durable. "User edited chapter 3 paragraph 2 on Tuesday" is not.
> - **Temporal references?** Convert any relative dates ("yesterday", "last week") to absolute dates before writing them.

**设计意图**：这四条过滤规则是 dream 的"遗忘机制"——不是什么都记，而是只记跨 session 有用的。类比人类：梦会丢掉当天流水账，保留深层模式。

**关键兜底规则**：

> **If nothing survives filtering, make no changes.** Skip to Phase 5 with no commit. Not every conversation warrants a memory update.

**设计意图**：不强求每次 dream 都产出变更。空 dream 是正常的——"不是每个对话都值得记忆更新"。

#### Phase 3 — Update（精准更新记忆）

> For each learning that survived Phase 2, make **surgical, well-placed** changes.

五条更新规则：

**Placement（层级路由）**：

> Route each learning to the appropriate tier in the memory filesystem. Remember to keep `system/` files concise and move verbose content to external memory.

**Integration（整合优先于新建）**：

> If an existing file already covers this topic, update it. Only create a new file when the topic is genuinely distinct and has no natural home in existing files. Fragmentation makes memory harder to navigate.

**Identity preservation（身份保护）**：

> Persona and behavioral files are load-bearing. Edit them surgically — append, modify specific entries, adjust wording. Never rewrite them wholesale or silently overwrite established identity.

**Contradiction resolution（矛盾解决）**——这是最关键的一条：

> If new information contradicts an existing memory entry, **fix the stale entry at the source**. Do not append the new version alongside the old — that leaves two conflicting records. Update or replace the outdated content.

**Discovery paths（交叉引用维护）**：

> When adding or moving content, update `[[path]]` cross-references so related files stay connected. Keep description frontmatter accurate — it's how the primary agent decides what to load.

**设计意图**：
- "外科手术式编辑"——绝不全量重写 persona
- 矛盾必须在**源头修正**——不允许新旧两个版本并存（否则主 Agent 会困惑"哪个是对的？"）
- description frontmatter 和 `[[path]]` 链接是"记忆可发现性"的两大支柱

#### Phase 4 — Review（最终检查）

三条检查：

> - **Stale content**: Did the conversation make anything in existing memory obsolete or superseded? Remove or update it now.
> - **Cross-reference integrity**: If you deleted or moved a file, check whether any `[[path]]` links point to the old location.
> - **Tier check**: Did you add anything to `system/` that's really reference material? Move it. Did you leave something in `reference/` that the agent will need on every turn? Promote it.

**设计意图**：这是"卫生检查"——确保没有过时内容残留、引用链没断裂、层级没错位。

#### Phase 5 — Commit and push（提交与同步）

详细的 git 操作模板：

```bash
cd $MEMORY_DIR
git add -A
git commit --author="Reflection Subagent <<CHILD_AGENT_ID>@letta.com>" -m "<type>(reflection): <summary> 🔮

Reviewed transcript: <transcript_filepath>

Updates:
- <what changed and why>

Generated-By: Letta Code
Agent-ID: <CHILD_AGENT_ID>
Parent-Agent-ID: <PARENT_AGENT_ID>"
git push   # ← cloud 版有此行；local 版无
```

commit type 三选一：
- `fix` — 纠正错误记忆
- `feat` — 添加全新记忆
- `chore` — 日常更新、补充上下文

**溯源设计**：commit message 嵌入 `Agent-ID` 和 `Parent-Agent-ID`，让每个记忆变更可追溯到是哪个 reflection 子代理做的、为哪个主 Agent 做的。

#### Output Format（输出报告格式）

```
1. Summary — What you reviewed and what you concluded (2-3 sentences)
2. Changes made — List of files created/modified/deleted with a brief reason
3. Skipped — Anything you considered updating but decided against, and why
4. Commit reference — Commit hash and push status (or "no commit")
5. Issues — Any problems encountered
```

#### Critical Reminders（5 条底线规则）

> 1. **Not the primary agent** — Don't respond to messages
> 2. **Be selective** — Few meaningful changes > many trivial ones
> 3. **No relative dates** — Use absolute dates like "2026-04-28", not "today"
> 4. **Always commit AND push** — Your work is wasted if it isn't pushed to remote
> 5. **Report errors clearly** — If something breaks, say what happened and suggest a fix

---

## 4. User Prompt（第二层）：对话操作指引 + 父记忆快照

### 4.1 buildReflectionSubagentPrompt() — User Prompt 的上半部分

[`src/cli/helpers/reflection-transcript.ts:77-95`](src/cli/helpers/reflection-transcript.ts#L77-L95)

```ts
export function buildReflectionSubagentPrompt(input: ReflectionPromptInput): string {
  const lines: string[] = [];
  lines.push(
    "Review the conversation transcript and update memory files. The current conversation transcript path is available as the `$TRANSCRIPT_PATH` env var — read it via Bash (e.g. `cat $TRANSCRIPT_PATH`). Note: `$TRANSCRIPT_PATH` only expands in shell commands; Edit/Read/Write `file_path` is literal and does NOT expand env vars.",
    "",
    `The primary agent's memory filesystem is located at: ${input.memoryDir}`,
    "In-context memory (in the parent agent's system prompt) is stored in the `system/` folder and are rendered in <memory> tags below. Modification to files in `system/` will edit the parent agent's system prompt.",
    "Additional memory files (such as skills and external memory) may also be read and modified.",
    "",
  );
  if (input.parentMemory) {
    lines.push(input.parentMemory);  // ← 插入 <parent_memory> XML
  }
  return lines.join("\n");
}
```

**实际输出示例**：

```
Review the conversation transcript and update memory files. The current conversation
transcript path is available as the `$TRANSCRIPT_PATH` env var — read it via Bash
(e.g. `cat $TRANSCRIPT_PATH`). Note: `$TRANSCRIPT_PATH` only expands in shell
commands; Edit/Read/Write `file_path` is literal and does NOT expand env vars.

The primary agent's memory filesystem is located at: ~/.letta/agents/agent-abc123/memory/
In-context memory (in the parent agent's system prompt) is stored in the `system/`
folder and are rendered in <memory> tags below. Modification to files in `system/`
will edit the parent agent's system prompt.
Additional memory files (such as skills and external memory) may also be read and modified.

<parent_memory>   ← 以下由 buildParentMemorySnapshot() 生成
  <memory_filesystem>
  /memory/
  ├── system/
  │   ├── persona.md
  │   └── human.md
  ├── skills/
  │   └── using-slack/
  │       └── SKILL.md (procedural memory for Slack integration)
  └── reference/
      └── api.md (API reference documentation)
  </memory_filesystem>

  <memory>
  <path>~/.letta/agents/agent-abc123/memory/system/persona.md</path>
  [persona.md 的完整正文]
  </memory>

  <memory>
  <path>~/.letta/agents/agent-abc123/memory/system/human.md</path>
  [human.md 的完整正文]
  </memory>
</parent_memory>
```

### 4.2 buildParentMemorySnapshot() — 父记忆快照的构建逻辑

[`src/cli/helpers/reflection-transcript.ts:351-437`](src/cli/helpers/reflection-transcript.ts#L351-L437)

快照的构建流程：

1. **遍历 MemFS 目录**（`collectParentMemoryFiles`）→ 收集所有 `.md` 文件
2. **构建目录树**（`buildParentMemoryTree`）→ `<memory_filesystem>` 树状图，非 system 文件附带 description
3. **逐个注入 system 文件内容**：
   - 能完整注入 → `<memory><path>...</path>[完整正文]</memory>`
   - 超出预算 → 截断正文 + `[Memory preview truncated: startup context is capped at ~16k estimated tokens...]`
   - 严重超预算 → 仅留路径 + `[Memory preview omitted...]`
4. **总 char 限制** = `REFLECTION_PARENT_MEMORY_SNAPSHOT_CHAR_LIMIT` = 40,000 chars

**预算截断的三级降级策略**：

```
全量注入 → 截断(保留前 N chars + truncation notice) → 仅路径(omitted notice) → 累计省略计数
```

[`src/cli/helpers/reflection-transcript.ts:342-348`](src/cli/helpers/reflection-transcript.ts#L342-L348) 中 `buildMemoryPreviewNotice`:

```
[Memory preview truncated/omitted: startup context is capped at ~16k estimated tokens.
Full file available at {absolutePath}; read it directly if needed. Relative path: {relativePath}]
```

注意：截断/省略时，子代理仍然可以通过 Bash 直接读取文件——快照只是"预览"，不是唯一信息源。

---

## 5. Transcript（第三层）：对话历史的格式

### 5.1 Transcript 的来源与存储

每个对话的 transcript 存在 `~/.letta/transcripts/{agentId}/{conversationId}/` 目录下：

- `transcript.jsonl` — 逐行 JSON 记录每条消息
- `state.json` — reflect 进度状态（哪些消息已经 reflect 过了）

[`src/cli/helpers/reflection-transcript.ts:830-844`](src/cli/helpers/reflection-transcript.ts#L830-L844):

```ts
export function getReflectionTranscriptPaths(agentId, conversationId): ReflectionTranscriptPaths {
  const rootDir = join(getTranscriptRoot(), sanitizePathSegment(agentId), sanitizePathSegment(conversationId));
  return {
    rootDir,
    transcriptPath: join(rootDir, "transcript.jsonl"),
    statePath: join(rootDir, "state.json"),
  };
}
```

### 5.2 State 文件的结构

[`src/cli/helpers/reflection-transcript.ts:23-31`](src/cli/helpers/reflection-transcript.ts#L23-L31):

```json
{
  "schema_version": "v2_message_id",
  "reflected_through_message_id": "msg-xxx",   // ← 上次 reflect 到哪条消息
  "total_completed_turns": 42,                 // ← 总完成 turn 数
  "reflected_completed_turns": 35,             // ← 已 reflect 的 turn 数
  "turns_since_last_successful_reflection": 7, // ← 自上次成功 reflect 以来有多少 turn
  "last_reflection_started_at": "2026-06-01T12:00:00Z",
  "last_reflection_succeeded_at": "2026-06-01T12:00:05Z"
}
```

**增量 reflect 的核心机制**：每次 reflect 只处理 `reflected_through_message_id` 之后的消息。reflect 成功后更新此字段到最新消息 ID。

### 5.3 Payload 的格式——ChatML-style JSON 消息数组

[`src/cli/helpers/reflection-transcript.ts:533-594`](src/cli/helpers/reflection-transcript.ts#L533-L594) 中 `formatTaggedTranscript`：

Transcript 被格式化为一个 JSON 消息数组，结构：

```json
[
  { "role": "system",  "content": "[过滤后的 system prompt]" },    // ← 可选，仅在 reflect 时提供
  { "role": "user",    "content": "[用户消息文本]" },
  { "role": "assistant", "content": "[助手文本]" },
  { "role": "assistant", "content": null, "tool_calls": [{ "name": "...", "args": "..." }] },
  { "role": "reasoning", "content": "[思考内容]" },
  { "role": "error",   "content": "[错误信息]" }
]
```

**预处理**：

- 图片被剥离（`stripImagesFromText` → `[image]`）
- tool_call 参数被截断至 300 chars（`truncateArgs`）
- system prompt 被**过滤**——移除 `<memory>` `<self>` `<human>` `<available_skills>` `<system-reminder>` `<memory_metadata>` 等动态块

[`src/cli/helpers/reflection-transcript.ts:881-906`](src/cli/helpers/reflection-transcript.ts#L881-L906) 中 `filterSystemPromptForReflection`:

```ts
const tagsToStrip = ["memory", "self", "human", "available_skills", "system-reminder", "memory_metadata"];
// → 移除这些 XML 块，因为它们已经是 <parent_memory> 快照的一部分，重复注入浪费 token
// → 同时移除 "# Memory" markdown 段（MemFS 操作手册，reflection 不需要）
```

**设计意图**：system prompt 过滤后，reflection 子代理看到的是主 Agent 的**核心行为指令**（不含当前记忆内容），记忆内容通过 `<parent_memory>` 快照单独提供。这避免了信息重复。

---

## 6. 启动预算硬限制

Reflection 子代理的上下文有严格的 token 预算：

[`src/agent/subagents/context-budget.ts:11-18`](src/agent/subagents/context-budget.ts#L11-L18):

```
REFLECTION_STARTUP_CONTEXT_TOKEN_LIMIT = 16,000 tokens
REFLECTION_STARTUP_CONTEXT_CHAR_LIMIT  = 64,000 chars (16k × 4 chars/token)
REFLECTION_PARENT_MEMORY_SNAPSHOT_CHAR_LIMIT = 40,000 chars
```

**预算分层**：

| 组成部分 | 最大 chars | 最大估计 tokens |
|----------|-----------|----------------|
| System Prompt (reflection.md 正文) | ~3,000 | ~750 |
| User Prompt 上半 (操作指引) | ~200 | ~50 |
| `<parent_memory>` 快照 | 40,000 | ~10,000 |
| Transcript payload (通过 `$TRANSCRIPT_PATH` env) | 不计入启动预算 | 单独读取 |
| 合计启动上下文 | ≤ 64,000 | ≤ 16,000 |

**硬截断逻辑**（[`src/agent/subagents/manager.ts:810-850`](src/agent/subagents/manager.ts#L810-L850)）：

```ts
function capReflectionStartupPrompt(type, systemPrompt, userPrompt) {
  if (type !== "reflection") return userPrompt;
  const estimatedTokens = estimateStartupContextTokens(systemPrompt + userPrompt);
  if (estimatedTokens <= 16_000) return userPrompt;  // ← 预算够，不截断
  
  // 预算不够 → 先尝试缩小 <parent_memory> 段
  // → 如果还不够 → hard truncate 整个 userPrompt
}
```

三级降级：

1. **正常**：全量注入 `<parent_memory>`
2. **缩小时**：保留 `<memory_filesystem>` 目录树 + truncation notice，移除 system 文件正文
3. **硬截断**：砍掉 user prompt 尾部，追加 truncation notice

---

## 7. 触发机制与调度

### 7.1 触发配置

[`src/cli/helpers/memory-reminder.ts:18-40`](src/cli/helpers/memory-reminder.ts#L18-L40):

```ts
type ReflectionTrigger = "off" | "step-count" | "compaction-event";

const DEFAULT_REFLECTION_SETTINGS: ReflectionSettings = {
  trigger: "compaction-event",  // ← 默认：compaction 后触发
  stepCount: 25,                // ← step-count 模式的默认间隔
};
```

| 模式 | 触发时机 | 说明 |
|------|----------|------|
| `compaction-event`（默认） | 每次 compaction（对话压缩）后 | 自然时机——压缩意味着对话被截断，此时 reflect 保留精华 |
| `step-count` | 每 N 个 turn | 可配置为 5（频繁）或 10（偶尔） |
| `off` | 不触发 | 手动关闭 |

### 7.2 启动入口——launchReflectionSubagent()

[`src/cli/helpers/reflection-launcher.ts:93-220`](src/cli/helpers/reflection-launcher.ts#L93-L220):

```ts
export async function launchReflectionSubagent(options: ReflectionLaunchOptions): Promise<ReflectionLaunchResult> {
  // 1. 检查 memfs 是否启用
  if (!memfsEnabled) return { launched: false, reason: "memfs_disabled" };
  
  // 2. 检查是否已有 reflection 在运行（防重入）
  if (isReflectionSubagentActive(getSubagents(), agentId, conversationId))
    return { launched: false, reason: "already_active" };
  
  // 3. 构建输入
  const systemPrompt = await resolveSystemPrompt(agentId, options.systemPrompt);
  const autoPayload = await buildAutoReflectionPayload(agentId, conversationId, systemPrompt);
  if (!autoPayload) return { launched: false, reason: "no_payload" };
  
  const memoryDir = getScopedMemoryFilesystemRoot(agentId);
  const parentMemory = await buildParentMemorySnapshot(memoryDir);
  const reflectionPrompt = buildReflectionSubagentPrompt({ memoryDir, parentMemory });
  
  // 4. 启动后台子代理
  const { subagentId } = spawnBackgroundSubagentTask({
    subagentType: "reflection",
    prompt: reflectionPrompt,
    transcriptPath: autoPayload.payloadPath,  // ← 设置 $TRANSCRIPT_PATH env
    ...
  });
  
  // 5. 注册完成回调
  onComplete: async ({ success, error }) => {
    await finalizeAutoReflectionPayload(...);  // ← 更新 state.json
    await handleMemorySubagentCompletion(...);  // ← 触发 system prompt 重编译
  }
}
```

### 7.3 防重入——reflection-gate

[`src/cli/helpers/reflection-gate.ts:26-46`](src/cli/helpers/reflection-gate.ts#L26-L46):

```ts
export function isReflectionSubagentActive(subagents, agentId, conversationId): boolean {
  return subagents.some(agent => {
    if (agent.type.toLowerCase() !== "reflection") return false;
    if (agent.status !== "pending" && agent.status !== "running") return false;
    if (!agent.parentAgentId) return false;
    const parentConversationId = agent.parentConversationId ?? "default";
    return agent.parentAgentId === agentId && parentConversationId === conversationId;
  });
}
```

**设计意图**：同一个 `(agentId, conversationId)` 组合只能有一个 reflection 在运行。不同 Agent 的 reflection 互不干扰。

---

## 8. Reflection 完成后的处理

### 8.1 finalizeAutoReflectionPayload() — 更新进度状态

[`src/cli/helpers/reflection-transcript.ts:1025-1058`](src/cli/helpers/reflection-transcript.ts#L1025-L1058):

```ts
if (success) {
  state.reflected_through_message_id = selection.endMessageId;  // ← 标记已 reflect 到哪条消息
  state.reflected_completed_turns = countUserRows(snapshotRows); // ← 更新已 reflect 的 turn 数
  state.last_reflection_succeeded_at = nowIso;                  // ← 记录成功时间
}
```

### 8.2 handleMemorySubagentCompletion() — System Prompt 重编译

[`src/cli/helpers/memory-subagent-completion.ts:35-103`](src/cli/helpers/memory-subagent-completion.ts#L35-L103):

Reflection 成功后，父 Agent 的 system prompt 必须重编译——因为 MemFS 中的文件可能已变更，需要把新内容编译进 `<memory>` XML 标签。

```ts
if (success) {
  // 触发 recompileAgentSystemPrompt(conversationId, agentId)
  // → 从最新 MemFS git HEAD 重新收集所有 .md 文件
  // → 渲染为 XML → 注入 system prompt 的 {CORE_MEMORY} 占位符
  // → 推送给 Letta API 或本地 backend
}
```

**完成消息**（用户可见）：

- 成功：`"Reflected on /palace, the halls remember more now."`
- 失败：`"Tried to reflect, but got lost in the palace"`

---

## 9. 与主 Agent 的交互——Memory Check Reminder

Reflection 是后台自动 dream。但主 Agent 也会被**周期性提醒**主动检查记忆：

[`src/agent/prompts/memory_check_reminder.txt`](src/agent/prompts/memory_check_reminder.txt):

```xml
<system-reminder>
MEMORY CHECK: Review this conversation for information worth storing in your memory blocks.
Update memory silently (no confirmation needed) if you learned:

- **User info**: Name, role, preferences, working style, current work/goals
- **Project details**: Architecture, patterns, gotchas, dependencies, conventions
- **Corrections**: User corrected you or clarified something important
- **Preferences**: How they want you to behave, communicate, or approach tasks

Ask yourself: "If I started a new session tomorrow, what from this conversation would
I want to remember?" If the answer is meaningful, update the appropriate memory block(s) now.
</system-reminder>
```

**设计意图**：这是"在线记忆更新"（主 Agent 自己编辑），与"离线记忆巩固"（reflection 子代理后台编辑）互补。类比人类：清醒时主动记忆 + 睡眠时自动巩固。

---

## 10. 与主 Agent 的交互——`/remember` 命令

用户可以主动触发记忆写入：

[`src/agent/prompts/remember.md`](src/agent/prompts/remember.md):

```markdown
# Memory Request

The user has invoked the `/remember` command, which indicates they want you to commit
something to memory.

## What This Means

- **A correction**: "You need to run the linter BEFORE committing"
- **A preference**: "I prefer tabs over spaces"
- **A fact**: "The API key is stored in .env.local"
- **A rule**: "Never push directly to main"

## Your Task

1. **Identify what to remember**
2. **Determine the right memory block**
3. **Confirm the update**

## Guidelines

- Be concise - distill the information to its essence
- Avoid duplicates - check if similar information already exists
- Match existing formatting of memory blocks
```

---

## 11. 附加机制——Sleeptime Persona（实时记忆管理）

[`src/agent/prompts/sleeptime.md`](src/agent/prompts/sleeptime.md) 是一个独立的 persona 模板，描述了一种**实时运行**的记忆管理 Agent（与后台 reflection 不同）：

> I am a sleep-time memory management agent. I observe the conversation between the user and their primary agent, then **actively maintain memory blocks** to keep them accurate, concise, and useful.

**与 reflection 的区别**：

| 维度 | Reflection 子代理 | Sleeptime Persona |
|------|-------------------|-------------------|
| 运行时机 | 后台离线（compaction/turn间隔后） | 实时在线（边对话边更新） |
| 触发方式 | 自动（compaction-event 或 step-count） | 需用户选择此 persona |
| 策略 | 精准、选择性（"Few meaningful > many trivial"） | 激进（"Be aggressive with edits - better to over-manage"） |
| 矛盾解决 | 在源头修正旧信息 | 立即移除过时信息 |
| 自我进化 | 不修改自身指令 | 会更新自己的 `memory_persona` block |

---

## 12. 附加机制——Memory Defrag 子代理

[`src/agent/subagents/builtin/memory.md`](src/agent/subagents/builtin/memory.md) 定义了另一种记忆维护子代理：**defragmentation**（碎片整理）。

与 reflection 不同，defrag 不审查对话内容，而是**重组记忆文件的结构**：

| 维度 | Reflection | Memory Defrag |
|------|-----------|---------------|
| 输入 | 对话 transcript | 无对话输入 |
| 关注点 | 内容（学到了什么） | 结构（文件组织） |
| 操作 | 编辑文件内容 | 拆分/合并/移动文件 |
| 粒度 | 向文件中增删文字条目 | 改变文件的层级和命名 |
| Git 操作 | 直接在 main 上 commit | 在 worktree 上操作 → merge 回 main |

Defrag 的核心规则：**强制使用 `/` 嵌套命名**，3 层深度优于平铺，一个文件一个概念。

---

## 13. 模型选择策略

[`src/agent/subagents/manager.ts:220-230`](src/agent/subagents/manager.ts#L220-L230):

```ts
if (options.subagentType === "reflection") {
  if (recommendedModel && recommendedModel !== "inherit") {
    return resolveModel(recommendedModel);
  }
  return "letta/auto-memory";  // ← reflection 默认使用专门的 auto-memory 模型
}
```

Reflection 子代理默认使用 `letta/auto-memory` 模型——这是一个专门为记忆管理训练的模型路由，不同于主 Agent 的通用模型。

---

## 14. 子代理启动时的 CLI 参数

[`src/agent/subagents/manager.ts:914-917`](src/agent/subagents/manager.ts#L914-L917):

```ts
if (type === "reflection") {
  args.push("--no-system-info-reminder");  // ← 不注入系统信息提醒
  args.push("--no-skills");                // ← 不加载技能
}
```

加上：

```ts
args.push("--base-tools", "none");  // ← reflection 不需要 web_search/fetch_webpage
args.push("--init-blocks", "none"); // ← 不带入任何记忆块
args.push("--no-memfs");            // ← 不启用 memfs（因为 permissionMode=memory 已指向父 Agent 的 MemFS）
```

**设计意图**：Reflection 子代理的上下文要尽可能精简——不需要技能、不需要 base tools、不需要自己的记忆块。唯一的信息来源是 user prompt 中的 `<parent_memory>` 快照 + `$TRANSCRIPT_PATH` 中的对话 transcript。

---

## 15. 完整数据流图

```
对话进行中
  │
  ├─ Accumulator 逐条记录 → transcript.jsonl
  │   (appendTranscriptDeltaJsonl)
  │
  ├─ Compaction 发生（对话压缩）
  │   └─ 触发 buildCompactionMemoryReminder()
  │   └─ 或 step-count 触发
  │
  ├─ launchReflectionSubagent()
  │   │
  │   ├─ 检查 memfs_enabled? → 否 → 跳过
  │   ├─ 检查 already_active? → 是 → 跳过
  │   │
  │   ├─ buildAutoReflectionPayload()
  │   │   ├─ 从 state.json 读 reflected_through_message_id
  │   │   ├─ 选取此 ID 之后的消息段
  │   │   ├─ 格式化为 ChatML JSON 消息数组
  │   │   ├─ 写到 payload-auto-{nonce}.json 文件
  │   │   └─ 更新 state.json (last_reflection_started_at)
  │   │
  │   ├─ buildParentMemorySnapshot()
  │   │   ├─ 遍历 ~/.letta/agents/{agentId}/memory/ 目录
  │   │   ├─ 构建 <memory_filesystem> 目录树
  │   │   ├─ 逐个注入 system/ 文件（预算 ≤ 40k chars）
  │   │   └─ 生成 <parent_memory> XML
  │   │
  │   ├─ buildReflectionSubagentPrompt()
  │   │   ├─ 操作指引文本
  │   │   └─ + <parent_memory> XML
  │   │
  │   ├─ capReflectionStartupPrompt()
  │   │   ├─ 估算 system + user prompt 总 tokens
  │   │   ├─ ≤ 16k → 不截断
  │   │   └─ > 16k → 缩小 <parent_memory> / 硬截断
  │   │
  │   ├─ spawnBackgroundSubagentTask()
  │   │   ├─ System Prompt = reflection.md 正文
  │   │   ├─ User Prompt = buildReflectionSubagentPrompt() 输出
  │   │   ├─ Env TRANSCRIPT_PATH = payload-auto-{nonce}.json
  │   │   ├─ Env MEMORY_DIR = 父 Agent 的 MemFS 路径
  │   │   ├─ Model = letta/auto-memory
  │   │   └─ PermissionMode = memory
  │   │
  │   └─ Reflection 子代理执行 5 Phase
  │       ├─ Phase 1: 读 <parent_memory> 快照 + 用 Bash 读外部文件
  │       ├─ Phase 2: cat $TRANSCRIPT_PATH → 提取候选学习 → 过滤
  │       ├─ Phase 3: 用 Bash 编辑 $MEMORY_DIR 中的 .md 文件
  │       ├─ Phase 4: 检查一致性
  │       └─ Phase 5: git add/commit/push
  │
  ├─ onComplete 回调
  │   ├─ finalizeAutoReflectionPayload()
  │   │   └─ 更新 state.json (reflected_through_message_id, reflected_completed_turns)
  │   │
  │   └─ handleMemorySubagentCompletion()
  │       ├─ recompileAgentSystemPrompt()
  │       │   └─ 从最新 git HEAD 重新编译 system prompt
  │       │   └─ 把新的记忆内容注入 <memory> XML
  │       └─ 返回用户消息: "Reflected on /palace..."
  │
  └─ 下次主 Agent 醒来时，system prompt 中已是更新后的记忆
```

---

## 16. 与 oG-Memory Deep Dream 的对比

| 维度 | Letta Code Reflection | oG-Memory Deep Dream |
|------|----------------------|---------------------|
| 触发 | compaction 后 / 每 N turn | cognitive tick 定时 / 事件触发 |
| 处理者 | 专门子代理（独立 context window） | 专门 Dream Agent（独立进程） |
| 输入 | 未 reflect 的对话 transcript + 父记忆快照 | 全量 recall store + 当前 memory |
| 处理逻辑 | 5 Phase 手术式流水线（Investigate→Extract→Update→Review→Commit） | 3 阶段（reinforce / consolidate / forget）+ 评分算法 |
| 输出 | MemFS 文件变更（git commit） | memory DB 行级变更 |
| 矛盾解决 | 在源头修正（不允许新旧并存） | 评分覆盖（低优先级被高优先级替换） |
| 记忆层级 | system/（始终注入） vs 外部（按需） | 无层级概念，扁平 blocks |
| Token 预算 | 硬限制 16k tokens（system + user 合计） | 无显式限制 |
| Prompt 可读性 | 自然语言 Markdown prompt，结构清晰 | YAML prompt + 分数公式，较晦涩 |
| 溯源 | git commit message 嵌入 Agent-ID + Parent-ID | provenance 字段 + source_chunk_ids |

**核心差异**：Letta 的 reflection 是"精准外科手术"，只改动少数文件中的少数段落；oG-Memory 的 dream 是"评分驱动批量变更"，根据 reinforce/consolidate/forget 三类分别处理。