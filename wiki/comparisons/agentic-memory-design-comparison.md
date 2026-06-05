---
name: agentic-memory-design-comparison
description: 四个 agentic memory 项目（oG-Memory / mem0 / memOS / Hermes-Agent）在六个核心设计问题上的对比总结，聚焦设计思路差异与可借鉴要点
tags: [ogmemory, mem0, memos, hermes-agent, agentic-memory, comparison, design]
created: 2026-06-03
updated: 2026-06-04
---

# Agentic Memory 设计思路对比

> 来源：oG-Memory 深度分析（wiki-vault）+ GitCode Discussion #3（mem0 2.0）、#4（memOS 2.0）、#6（Hermes-Agent）
> 目标：提炼最有差异化、最凸显设计思路的部分，提出可借鉴要点用于完善 oG-Memory

---

## Q1：如何对接 Agent

| 维度                           | **oG-Memory**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | **mem0 2.0**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | **memOS 2.0**                                                                                                                                                                                                                                                                                                                                                                                                 | **Hermes-Agent**                                                                                                                                                                                                                                                                                                     |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **架构拓扑**                     | 独立 HTTP 服务（out-of-process），Agent 通过中间脚本调用 API                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | 独立 SDK/服务（out-of-process），Agent 以工具形式接入                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | 独立服务（out-of-process），三种接入形态并行                                                                                                                                                                                                                                                                                                                                                                                 | **in-process**：`MemoryStore` 直接作为 Agent 实例的 Python 属性持有，Agent 内部直接操作，无需网络调用                                                                                                                                                                                                                                          |
| **已对接的 Agent**               | **2 个**：① **Claude Code** — 通过 `claude-plugin/` 文件夹接入（Python+Bash 中间脚本 + hooks.json + plugin.json）；② **OpenClaw** — 通过 `openclaw_context_engine_plugin/` 文件夹接入（Node.js 插件，直接 HTTP 调用）                                                                                                                                                                                                                                                                                                                                     | **2 个**：① **Claude Code** — 通过 `.mcp.json`（MCP 配置）+ `hooks/hooks.json`（Hook 配置）+ `skills/` 目录接入；② **OpenCode** — 通过 `opencode.json`（内嵌 MCP 字段）+ `opencode-mem0.ts`（lifecycle hooks）+ `opencode-skills/` 目录接入                                                                                                                                                                                                                                                                                                                                      | **3 类接入形态，已对接多个 Agent**：① **SDK/REST API** — 任何编程语言可通过 HTTP 调用（MemOSClient Python SDK），适合自定义 Agent；② **MCP** — 任何支持 MCP 的 Agent 框架可直接接入，已明确支持 **Cursor**、**Coze Space**；③ **Plugin** — 面向特定 Agent 深度集成，已对接 **OpenClaw**（`@memtensor/memos-local-openclaw-plugin` — 全量写入、混合召回、渐进式检索）和 **Hermes Agent**（`@memtensor/memos-local-plugin` — Reflect2Evolve 分层记忆架构）                                                | **1 个**：Hermes-Agent 自身——记忆系统是 Hermes Agent 的内建组件，不是独立服务，不存在"对接"的概念。但通过 `MemoryProvider` ABC 可桥接外部记忆服务（如 Honcho / Mem0 / Holographic / Supermemory），同一时刻只允许一个                                                                                                                                                        |
| **各 Agent 的 Hook（自动触发）**     | **Claude Code 的 4 个 Hook**：`UserPromptSubmit` → `call_compose.py`（提问时检索注入）、`PostToolUse` → `call_add_session_message.py`（有副作用工具的 I/O 补录到 session，仅匹配 Write/Edit/Bash/NotebookEdit）、`Stop` → `stop_detach.sh`（对话结束异步 after_turn）、`PreCompact` → `call_after_turn.py`（压缩前同步防丢失）。**OpenClaw 的 4 个 lifecycle method**：`assemble()` → prefetch+compose（分层检索注入）、`afterTurn()` → after_turn（增量提取持久化）、`compact()` → prepare_compaction+compact（压缩接管）、`dispose()` → dispose（释放资源）。两个 Agent 的 Hook 点名称和触发时机不同，但映射到同一套 oG-Memory API | **Claude Code 的 6 个 Hook**（定义在 `hooks/hooks.json`）：`Setup` → `ensure_deps.sh`（安装依赖）、`SessionStart` → `on_session_start.sh`（加载记忆）、`UserPromptSubmit` → `on_user_prompt.sh`（注入搜索建议）、`PreToolUse` → `block_memory_write.sh` + `enforce_metadata_defaults.sh`（阻止未授权写入、强制元数据默认值）、`PostToolUse` → `on_post_tool_use.sh` + `on_bash_output.sh`（捕获工具输出）、`PreCompact` → `on_pre_compact.sh` → `on_pre_compact.py`（压缩前存摘要）。**OpenCode 的 lifecycle hooks**（定义在 `opencode-mem0.ts`）：嵌入在 TypeScript 代码中，具体 Hook 点与 Claude Code 类似但不完全相同（OpenCode 有自己的生命周期事件定义） | **Plugin 的 8 个精细 Hook**（定义在 `src/memos/plugins/`，面向 OpenClaw 和 Hermes Agent）：ADD_BEFORE/ADD_AFTER（写入前后干预）、SEARCH_BEFORE/SEARCH_AFTER（检索前后干预）、SEARCH_MEMORY_RESULTS（在去重/rerank 前修改检索结果）、MEM_READER_PRE_EXTRACT（自定义提取 prompt）、MEMORY_ITEMS_AFTER_FINE_EXTRACT（精细提取后处理）、DREAM_BEFORE_PERSIST/DREAM_AFTER_PERSIST（离线反思前后）。**MCP 和 SDK/REST API 没有任何 Hook**——通过 MCP/SDK 接入的 Agent（如 Cursor、Coze Space）无法干预记忆系统内部流程 | **Hermes Agent 内建的拦截点**：`tool_executor.py` 拦截 memory 工具调用 → 直接传 store 实例 + 触发 `on_memory_write()` 桥接通知外部 Provider；`system_prompt.py` 在 session start 时拼冻结快照；`conversation_loop.py` 的 Nudge 计数器每 N 轮触发后台审查；`background_review.py` 在主对话完成后 fork 受限 Agent 审查对话。这些不是"Hook"而是 Hermes 自身的代码逻辑——因为记忆是内建的，不需要通过 Hook 机制与自身对接 |
| **各 Agent 的 Skill/工具（按需调用）** | **Claude Code 的 2 个 Skill**（定义在 `.claude-plugin/plugin.json`）：`/og-compose <关键词>` — Agent 觉得需要检索记忆时主动用；`/og-add-history` — Agent 或用户决定导入本仓库历史对话。**OpenClaw 没有额外的 Skill**——OpenClaw 的所有交互都通过 lifecycle method 自动完成                                                                                                                                                                                                                                                                                                         | **Claude Code 的 3 个 Skill**：`/mem0:remember` — 直接存储记忆；`/mem0:tour` — 浏览记忆；`/mem0:peek` — 快速搜索。**OpenCode 的 Skill**：在 `opencode-skills/` 目录下，具体命令与 Claude Code 类似                                                                                                                                                                                                                                                                                                                                                                                  | **MCP 暴露的工具**（面向 Cursor、Coze Space 等通用 MCP 客户端）：`add_memory`（存储记忆）、`search_memories`（语义搜索）、`chat`（对话）、`create_cube`（创建记忆立方体）等。**SDK 方法**（面向编程接入）：`add_message()`、`search_memory()`、`chat()`、`create_knowledgebase()` 等。**Plugin 没有额外 Skill**——Plugin 的交互都通过 Hook 自动完成                                                                                                                                         | **Hermes Agent 的 memory 工具**（定义在 `tools/registry.py`，Agent 通过 LLM tool calling 自主调用）：支持 `add`（新增条目）、`replace`（短子串匹配替换旧条目）、`remove`（删除条目）。还有 `session_search` 工具（跨会话检索对话历史）                                                                                                                                           |
| **Agent 对记忆的感知**             | Agent 只看到 compose 返回的 `systemPromptAddition` / `additionalContext`，**不知道记忆内部结构**                                                                                                                                                                                                                                                                                                                                                                                                                                          | Agent 通过 MCP tool 调用 add/search，**对记忆有工具级感知**——知道有 add_memory、search_memory 等方法                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | SDK/API 调用者知道方法名（add_message / search_memory / chat 等），知道可以创建 knowledgebase；**Plugin 使用者可注册 Hook 干预提取和检索流程**；MCP 使用者只能调用暴露的工具，**无法干预内部流程**                                                                                                                                                                                                                                                                    | **Agent 完全知道记忆结构**——memory 工具直接操作 store，schema description 中编码了行为引导（何时保存、保存优先级、什么不该保存）                                                                                                                                                                                                                               |
| **旁路数据流**                    | Claude Code 的 `PostToolUse` Hook 仅将有副作用工具（Write/Edit/Bash）的 I/O 补录到 session transcript，不经过 Agent 主循环。OpenClaw 无旁路——所有数据通过 lifecycle method 传入                                                                                                                                                                                                                                                                                                                                                                             | Claude Code 和 OpenCode 的 Hook 自动捕获每 3 turns 的对话，没有独立的工具 I/O 旁路                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | 5 类产生渠道（详见 Q3），工具轨迹作为独立渠道（ToolTrajectoryMemory），全链路记录请求→结果→错误→重试→组合模式                                                                                                                                                                                                                                                                                                                                         | 无旁路——Agent 只有对话和工具调用两条路径                                                                                                                                                                                                                                                                                             |

### 关键概念展开

**out-of-process vs in-process**：oG-Memory、mem0、memOS 都是独立运行的记忆服务，Agent 通过 HTTP/MCP/SDK 等协议与之交互——好处是部署灵活、多 Agent 可共享同一记忆实例、Agent 框架无关；代价是需要网络通信、Agent 需要额外配置接入。Hermes 把 `MemoryStore` 直接作为 Agent 实例的 Python 属性持有——好处是零延迟、零配置、Agent 对记忆有完全控制权；代价是不可跨 Agent 共享、与 Agent 框架耦合。

**Hook vs Skill vs MCP**：Hook 是一定会跑的——Agent 在生命周期节点自动触发，不需要主动决定，保证核心读写流程不遗漏；Skill 是 Agent 按需调用——Agent 自己决定是否搜索更多上下文、是否导入历史；MCP 是标准化协议——让任何支持 MCP 的 Agent 框架（Cursor、Coze Space 等）直接接入。

### 💡 可借鉴要点

1. **Hermes 的"Schema description 编码行为引导"**：mem0 和 oG-Memory 的行为引导都在系统提示词中，但 Hermes 把优先级指导（"User preferences and corrections > environment facts > procedural knowledge"）和保存时机指导（"WHEN TO SAVE: User corrects you, shares preference..."）写在 **工具 schema description** 里。LLM 在工具选择时直接参考 schema，这比系统提示词更精准地在决策点生效——系统提示词是泛泛的背景信息，schema description 是 Agent 正在决策"要不要调用这个工具"时直接看到的上下文。oG-Memory 可以考虑将行为引导从系统提示词迁移到 YAML schema 的 description 字段。

2. **memOS 的"Plugin Hook 机制"**：memOS Plugin 系统有 8 个精细 Hook 点，允许第三方在记忆生命周期关键节点干预——比如 `SEARCH_MEMORY_RESULTS` Hook 让外部逻辑在去重/rerank 之前修改检索结果，`MEM_READER_PRE_EXTRACT` Hook 让外部自定义提取前的 prompt。oG-Memory 目前只有 Hook/Skill 两种对外接口，缺少"提取后干预"和"检索结果干预"的 Hook 点。如果开放这些 Hook，可以让外部逻辑介入核心流程（如特定领域的去重规则、特定业务的检索偏好），而不需要修改 oG-Memory 代码本身。

3. **Hermes 的"内建 + 外部双轨"**：Hermes 的 `MemoryStore`（内建、全量注入、策展式、2,200 字符）+ `MemoryProvider`（外部、按需、能力更强）共存设计很有启发。内建层作为"高置信度事实的快速通道"——Agent 直接持有、全量注入、不需要检索；外部层提供更丰富的能力（向量检索、跨会话关联等）。oG-Memory 目前是纯外部服务模式——可以考虑增加一个轻量内建层作为确定性知识的快速通道，避免每轮都走完整检索流水线来获取 profile/偏好这种极其稳定的信息。

---

## Q2：记忆对象模型如何构建

| 维度           | **oG-Memory**                                                                                                                                                                                                  | **mem0 2.0**                                                                                                   | **memOS 2.0**                                                                                                    | **Hermes-Agent**                                                                                                                                                            |
| ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **记忆分类**     | **7 类 YAML 定义**：profile（用户属性，如姓名职业）、preference（用户偏好，如咖啡口味编码风格）、entity（实体，如人地点组织）、event（时间事件，如会议旅行）、case（问题解决案例）、pattern（行为模式观察）、skill（可复用技能工作流）、tool（工具使用洞察）。每种类型有独立的提取 prompt 和写入策略。**增加新类型只需加 YAML 文件，不改代码** | **3 类 enum**：Procedural（程序记忆——过程/步骤）、Episodic（情景记忆——具体事件经历）、Semantic（语义记忆——事实知识）。enum.py 中定义但不强制约束，实际存储时不按类型分区 | **6 类内容 + 3 层载体**（6 类内容指记忆内容的品类划分：①对话类、②用户画像类、③工具调用类、④文件/URL/图片类、⑤反馈与修正类、⑥系统与元信息；3 层载体指认知模式的分层：明文记忆/激活记忆/参数记忆。详见下方展开）                                                                                        | **2 个 target**：`memory`（Agent 的个人笔记——环境事实、项目惯例、工具怪癖）和 `user`（用户画像——偏好、沟通风格、习惯）。极简但语义角色截然不同——memory 是"我学到的教训"，user 是"关于用户的事实"                                                |
| **粒度**       | **主题级**：一个 preference 节点 = 一个偏好主题（如"coding_style"主题下包含该用户所有编码偏好信息）；一个 entity 节点 = 一个实体的全部关键属性                                                                                                                  | **原子级**：一条事实一条记录（如"用户喜欢巴黎铁塔"就是一条独立 memory），扁平存储                                                                | **主题级**：一条 TextualMemoryItem 是一个完整记忆单元（如"用户喜欢巴黎铁塔"包含 id、memory、metadata、embedding、version、history、sources 等全部字段） | **策展条目级**：MEMORY.md 2,200 字符约 10-15 条 § 分隔条目，每条是 Agent 自己策展后选择保存的一句话事实或偏好                                                                                                   |
| **层级结构**     | **三级索引**：每个 ContextNode 对应 3 个向量索引记录——L0 (abstract, ~100 tokens, 路标式嵌入)、L1 (overview, ~500 tokens, 平衡信号)、L2 (content, ~5000 tokens, 全面细节)。检索时从 L0/L1 种子点递归展开到 L2 叶节点                                           | 扁平：只有 embedding 索引一层，无层级                                                                                       | **3 层载体**（详见下方展开）+ 3 种 backend（详见下方展开）                                                                           | **四层栈**：Layer 1 MemoryStore（策展式 MEMORY.md/USER.md）→ Layer 2 SessionDB（SQLite 原始对话轨迹）→ Layer 3 ContextCompressor（工作记忆缓存）→ Layer 4 External Provider（可选外部记忆服务）。每层对"什么是记忆"定义不同 |
| **Scope 区分** | **user-scope vs agent-scope**：profile/preference/entity/event/pattern 的 owner_scope 是 user（关于用户的知识），case/skill/tool 的 owner_scope 是 agent（Agent 自己积累的经验）。URI 中通过 owner_space 字段区分                              | user_id / agent_id / run_id 在 metadata 中标注，但不做强制分区——所有记忆混在同一存储中                                                | user_id 隔离不同用户的记忆；tag 纯手动（用户自定义标签用于分类/角色/项目隔离，系统不自动生成 tag）；知识库与记忆是两套独立存储                                         | **语义角色区分而非 scope 区分**：memory target 是 Agent 的内省笔记（"我学到的教训"→ 模型产生遵循行为），user target 是对用户的外部观察（"关于用户的事实"→ 模型产生适配行为）。模型对两者的解读方式不同                                               |
| **关系结构**     | `RelationEdge`：有向关系边，4 种类型：related_to（相关）、derived_from（来源）、contradicts（矛盾）、SEQUENCE（时序），带权重 0.0-1.0                                                                                                            | 隐式关联：embedding similarity（向量空间邻近）、shared metadata（相同 user_id/agent_id）、shared entities（相同提取实体）                 | TreeTextMemory 有显式边：SUMMARY/MATERIAL（摘要与原文）、FOLLOWING/PRECEDING（文档分块先后）、LLM 推理出的语义关系                             | 无显式关系——§ 分隔扁平条目，条目间无关联                                                                                                                                                      |

### 关键概念展开

**memOS 2.0 的 6 类内容**：memOS 把"什么东西可以作为记忆"分成 6 个品类，这是**内容维度的品类划分**（"存的是什么东西"）——与载体维度（"处于什么认知状态"）和 backend 维度（"用什么技术存储"）是三个不同的视角：

1. **对话类**（最基础）：用户提问/指令/个人信息、Agent 回答/解释/结论、多轮对话的上下文摘要（不是整段原文，是精炼后）。
2. **用户画像类**（长期偏好）：姓名/年龄/身份/职业、喜好/禁忌/风格偏好（如"喜欢简洁回答"）、常用工具/常用格式/沟通习惯。
3. **工具调用类**（Agent 行为记忆）：调用了什么工具（search/code_interpreter）、入参/出参/调用结果、失败/成功经验/重试逻辑、工具组合模式（如"先搜再总结再画图"）。
4. **文件/URL/图片类**（多模态）：上传文件的内容摘要/关键信息、URL 网页正文摘要、图片内容描述/OCR 结果/视觉特征索引。
5. **反馈与修正类**（记忆进化）：用户说"你记错了""不是这个意思"、Agent 自我反思/修正结论、人工编辑/删除/补充的记忆。
6. **系统与元信息**（管理用）：记忆来源（chat/tool/file/feedback）、时间戳/会话链/用户-Agent 关系、标签/权重/优先级（影响召回排序）。

6 类内容不是独立的存储分区——它们都统一为 TextualMemoryItem 对象，在 metadata 中通过 `source` 字段（conversation / retrieved / web / file / system）标记来源品类，通过 `memory_type` 字段标记生命周期类型（WorkingMemory / LongTermMemory / UserMemory / ToolTrajectoryMemory / SkillMemory / PreferenceMemory / RawFileMemory / ToolSchemaMemory / OuterMemory / Context）。分类的意义是**明确记忆的品类边界**——"所有与用户/Agent 交互产生的、有长期价值的信息，都会被自动或手动转为记忆"，无论是对话、偏好、工具轨迹、文件、反馈还是元信息。

**6 类内容 × 3 层载体 × 3 种 backend 的关系**——这三者是三个独立的分类维度，不存在固定的一对一映射：

| 维度              | 作用                                            | 选择时机                                                                          |
| --------------- | --------------------------------------------- | ----------------------------------------------------------------------------- |
| **6 类内容**       | 描述"存的是什么"——品类边界                               | 写入时由 source 字段和 memory_type 自动标记                                              |
| **3 层载体**       | 描述"处于什么认知状态"——明文（我知道什么）/ 激活（我在想什么）/ 参数（我能做什么） | 运行时由 MemCube 统一调度：新记忆默认进明文记忆，会话中热数据进激活记忆，Dream 提炼的技能进参数记忆                     |
| **3 种 backend** | 描述"用什么技术存储"——只服务于明文记忆这一层                      | 部署时配置 MemCube 的 text_mem backend（Naive/General/Tree），激活记忆和参数记忆各有自己的实现，不受此选择影响 |

具体映射规则：

- **对话类、用户画像类、反馈与修正类** → 写入时默认进入**明文记忆**的 WorkingMemory 或 LongTermMemory / UserMemory 区，运行时热度高的内容会被提升到**激活记忆**（KV Cache）
- **工具调用类** → 写入时进入**明文记忆**的 ToolTrajectoryMemory 区，通过独立接口（SDK `add_tool_trajectory()` / MCP tool/track）写入
- **文件/URL/图片类** → 写入时进入**明文记忆**的 RawFileMemory 区，多模态内容由 MemReader 处理
- **系统与元信息** → 不独立存储，嵌入在每条记忆的 metadata 字段中
- **用户画像类中的偏好** → 有独立的 `pref_mem` 子存储（MemCube 的第四个子存储，继承 BaseTextMemory，在 GeneralMemCube 中与 text_mem 并列），`memory_type = PreferenceMemory`
- **Dream 提炼的技能** → 目标载体是**参数记忆**，但目前 LoRA 实现是 placeholder，实际写入**明文记忆**的 SkillMemory 区（memory_type = SkillMemory）作为过渡

3 种 backend 的适用范围更窄——**只有明文记忆（text_mem）有 3 种 backend 选择**；激活记忆（act_mem）有 KVCache / VLLMKVCache 两种实现；参数记忆（para_mem）有 LoRA 一种实现（目前是 placeholder）。部署时在 MemCube 配置中分别指定各子存储的 backend。

**memOS 2.0 的三层载体**：这是 memOS 2.0 最独特的设计，不是简单的冷热分层，而是**认知模式的分层**：

1. **明文记忆（Plaintext Memory）**——"磁盘式"记忆：可读写、可检索、可编辑的持久化存储。存的是对话摘要、用户偏好、事实、文档内容等文本信息。格式是 TreeTextMemory（图结构 + 向量，底层用 Neo4j 图数据库存储节点+边）。状态有 activated（正常激活，推理默认召回）/ archived（归档，默认不召回）/ deleted（软删除，永不召回）三种。这是"我知道什么"——我积累的知识库。

2. **激活记忆（Activated Memory）**——"RAM 式"记忆：运行时高速缓存。存的是最近几轮对话、KV Cache、临时上下文。速度最快、容量有限、会话结束后可降级为明文记忆。这是"我现在在想什么"——当前的注意力焦点。

3. **参数记忆（Parametric Memory）**——"技能式"记忆：从大量交互中提炼成"技能/知识"的长期记忆（类似模型微调的效果）。存的是高频模式、决策模板、工具调用组合、领域技能。**不可直接编辑**，只能通过大量交互提炼（类似 Dream 模块的工作）；调用效率最高。这是"我能做什么"——内化后无需检索的直觉能力。

三层载体在 MemCube 中各有独立的子存储（`GeneralMemCube` 持有 `text_mem`、`act_mem`、`para_mem`、`pref_mem` 四个独立实例），但共享统一的管控逻辑——明文记忆是核心，所有记忆首先写入明文记忆（通过 TextualMemoryItem 的 metadata.memory_type 字段区分品类），运行时热度高的内容被提升到激活记忆（KV Cache），Dream 提炼的技能写入参数记忆（目前 LoRA 实现为 placeholder，实际仍存为明文记忆中的 SkillMemory）。偏好记忆有独立的 `pref_mem` 子存储（`GeneralMemCube` 中的第四个子实例），但同样继承 `BaseTextMemory`，本质上是明文记忆的一种专化。

**memOS 2.0 的三种 backend**：**三种 backend 只服务于明文记忆（text_mem）这一层载体**，激活记忆和参数记忆各有自己的实现（KVCache/VLLMKVCache 和 LoRA），不受此选择影响。部署时在 MemCube 配置中分别指定各子存储的 backend。明文记忆的三种实现都继承 `BaseTextMemory`，有相同的 add/search/get/dump/load 方法，区别只在底层存储和检索方式：

1. **NaiveTextMemory** — 纯内存实现：Python 列表存储，词集交集计数检索（数 query 和 memory 有多少个词重叠），零依赖。适用于 demo/单测。
2. **GeneralTextMemory** — 向量数据库实现：Qdrant/Milvus 存储，embedding 余弦相似度检索，有真正的语义搜索能力但记忆间没有结构关系（扁平向量集合）。适用于需要语义搜索但不需要记忆间关系的场景。
3. **TreeTextMemory** — 图数据库实现（最完整）：Neo4j 存储（节点+边），多路检索（向量搜索 + BM25 全文搜索 + 图遍历 + 互联网检索 + rerank），记忆有层级分类（WorkingMemory 默认 20 条 / LongTermMemory 默认 1500 条 / UserMemory 默认 480 条），记忆之间有边关系（SUMMARY/MATERIAL, FOLLOWING/PRECEDING, LLM 推理语义关系）。适用于生产环境。

**Hermes-Agent 的双态设计**：Hermes 的内建记忆在同一份数据上维护两种状态：

- **活跃状态**（memory_entries / user_entries）：工具读写此状态，每次变更立即持久化到磁盘。Agent 通过工具响应看到的是实时活跃状态。
- **冻结快照**（_system_prompt_snapshot）：session 开始时一次性从磁盘"拍摄"，注入 system prompt 后在整个 session 内**字节不变**。这保护了 Anthropic 的 prefix cache——system prompt 不变则缓存不失效，token 成本不翻倍。

两者在 session 中段可能不一致（例如 Agent 删除了条目 A，但 system prompt 中仍然是 A）。这个不一致**是有意为之**——活跃状态保证 Agent 能看到自己的变更，冻结快照保证 prefix cache 稳定。唯一的快照刷新时机是上下文压缩——压缩已经使 prefix cache 失效，此时刷新快照不产生额外缓存成本。

### 💡 可借鉴要点

1. **memOS 的"三层载体"**：明文→激活→参数的三层不是冷热分层，而是认知模式分层——"我知道什么 / 我在想什么 / 我能做什么"。oG-Memory 目前只有 L0/L1/L2 三级索引（同一记忆的不同压缩版），缺少"技能层"——可以考虑在 skill/tool 类型之外增加一个 **ParametricMemory** 层：存放从大量交互中提炼的高频决策模板，不可直接编辑、调用时无需检索（直接注入），类似"内化技能"。这是 Dream 模块的天然产出——Dream 提炼的 SkillMemory 应写入 ParametricMemory 而非明文记忆。

2. **Hermes 的"语义角色区分"**：memory 和 user 两个 target 不是 scope 区分，而是**语义角色**区分——"这是我学到的教训" vs "这是关于用户的事实"。模型对两者产生不同行为倾向：memory → 遵循行为，user → 适配行为。oG-Memory 的 profile/preference 虽然也有区分，但注入时混在同一个 identity_context slot 里——可以考虑在注入 prompt 时**显式标注语义角色**（如 "## Agent's accumulated experience" vs "## User's stated preferences"），让模型对两类信息产生不同的行为倾向，而不是把它们当作同一种"背景知识"处理。

3. **oG-Memory 自身的优势：YAML schema 无代码扩展**。其他三个项目都是硬编码类型（mem0 3 种 enum、memOS Pydantic Model、Hermes 2 个 target），增加新记忆类型需要改代码。oG-Memory 的 YAML 驱动 SchemaRegistry + tool_builder 机制让增加新类型只需要加 YAML 文件。这一点**不应放弃**，反而是其他项目可以借鉴的。

4. **memOS 的"反馈与修正类"——oG-Memory 缺少用户修正驱动的记忆进化机制**：memOS 把"反馈与修正"作为独立的记忆品类（第 5 类），用户说"你记错了""不是这个意思"时，修正不是简单地覆盖旧记忆，而是有独立的 `mem_feedback` 模块处理——ArchivedTextualMemory 的 `update_type` 字段显式标记为 `feedback`，旧内容归档为 history 条目保留可追溯性，新内容设为 activated。oG-Memory 目前完全没有这条通道——用户纠正 Agent 后，修正只能通过下一次 after_turn 提取重新跑一遍提取流程来覆盖旧信息（profile 用 ProfilePolicy 替换、preference 用 AggregateTopicPolicy 追加合并），但**没有"旧信息被用户修正了"的显式标记**，也没有修正来源追踪（用户说了什么导致这条记忆被更新？归档的 history 只记录了新旧内容的版本号和合并来源，不区分"用户纠正"和"自然提取到的更完整信息"）。可以考虑：① 增加一个 `feedback` 类型 YAML schema，专门处理用户纠正场景——有独立的提取 prompt（"用户刚才纠正了什么？"）、写入策略（将旧条目标记为 superseded_by_feedback + 保留旧内容摘要 + 写入新修正内容，而非笼统的替换/追加）；② 在 ContextNode 的 provenance_ids 中增加修正来源类型标记（`provenance_type: feedback | extraction | merge`），让检索时知道"这条偏好被用户纠正过"vs"这条偏好是系统自动提取的"——被用户纠正过的记忆应该有更高的置信度权重。

---

## Q3：记忆怎么产生

| 维度 | **oG-Memory** | **mem0 2.0** | **memOS 2.0** | **Hermes-Agent** |
|------|--------------|-------------|--------------|-----------------|
| **产生渠道数** | **1 类**：对话后自动提取（after_turn / PreCompact / SessionCommit / Bootstrap 四个触发时机，但都是"对话后 LLM 提取"这一种机制） | **2 类**：① Agent/用户主动调用 memory.add() 触发 LLM 提取；② Hook 每 3 turns 自动触发提取 | **5 类**（详见下方展开） | **3 类**（详见下方展开） |
| **提取方法论** | **两阶段 + 双重运行**：Phase 1 用便宜 LLM 识别哪些消息范围包含可提取信息（输出 span 列表）；Phase 2 对每个 span 用 LLM function calling 调用 extract_profile/extract_preference 等工具结构化提取。**每个 span 跑两次**（正常温度 + temperature=0），取置信度高/内容丰富的合并。confidence < 0.5 直接丢弃。另有 ReAct 循环模式（Lazy 模式）：LLM 可在提取中调用 read/list 等工具读取已有记忆，有 Safety Check 防止写未读的已存在节点 | **LLM 单次提取**：依赖 LLM prompt 对消息进行提取，没有 span 划分（不先识别哪些片段值得提取再结构化，而是直接整体提取），提取后→结构化输出→向量化→去重（更新记忆）→存储 | **MemReader 5 步流水线**：①捕获 raw dialog → ②MemReader（自研 4B/1.7B 抽取模型）抽取结构化记忆（事实/偏好/时间/对象）→ ③Embedding 去重 + LLM 校验 → ④分类短期/长期 → ⑤写入 MemCube 存储。输出记忆类型：FactMemory / PreferenceMemory / TimeSensitiveMemory | **Nudge 后台审查**：主对话流完成后，在 daemon 线程中 fork 一个受限 Agent 审查对话——受限 Agent 只能用 memory 和 skill 管理工具（白名单控制），不激活外部 provider，审查提示词关注"用户是否透露了值得记住的个人信息/偏好/期望" |
| **自动化程度** | **全自动**，不支持用户手动写入记忆 | **全自动为主**，用户也可通过 `/mem0:remember` skill 手动写入 | **全自动为主**，用户也可通过 add_memory API / 控制台手动写入（手动写入是即时入库、不经过抽取，用于高置信度信息） | **Agent 主动为主**（Agent 自己决定何时调用 memory 工具保存），系统自动为辅（Nudge 每 N 轨触发审查），不支持用户直接写入 |
| **离线处理** | **无**——只有实时提取，没有离线优化 | **无**——没有后台合并/去重/反思机制 | **有 Dream 模块**（详见下方展开） | **无**——但有策展压力间接实现过期（空间满了必须删旧的） |
| **工具轨迹** | PostToolUse Hook 仅将有副作用工具（Write/Edit/Bash）的 I/O 补录到 session transcript，统一进入对话后提取流程——tool 类型有专门的提取 schema（best_for/optimal_params/common_failures 等字段），但没有独立存储渠道 | PostToolUse Hook 自动捕获工具 I/O，但没有独立的工具轨迹记忆类型 | **独立渠道 ToolTrajectoryMemory**：全链路记录工具调用请求（tool name/params）→ 工具返回结果（content/status/error）→ 代码执行轨迹/命令行日志/异常栈 → 多步 Action→Observation→Reflection。每步执行后立即写入（"执行即学习"），通过 SDK add_tool_trajectory() / MCP tool/track 接入。全链路记录、不可丢、可审计、可复现 | 无独立工具轨迹渠道——session_search 可以检索历史对话中的工具调用，但不做结构化提炼 |
| **任务总结** | **无**——session_archive 是压缩产物（将长对话压缩为摘要），不是对任务经验的反思总结 | **无** | **有 TaskSummaryArchiving**：当一个完整任务闭环（多轮对话+多工具调用）标记为 success/fail/abort 后，LLM 生成任务级摘要记忆、合并同任务下的碎片记忆、提炼关键决策点与结果、写入 TaskMemory。目的是减少碎片、提升跨会话检索命中率 | **无**——session_search 可检索历史对话但不做任务级总结 |

### 关键概念展开

**memOS 2.0 的 5 类产生渠道**：

1. **用户主动写入（User Direct Memory）**：用户通过 add_memory API / 控制台 / MCP memories/add 直接输入事实、偏好、待办。即时写入、即时入库、**不经过 LLM 抽取**，用于高置信度、用户明确声明的信息。
2. **对话后自动抽取（Chat-as-Learning）**：最主要来源、默认开启。每轮 user↔assistant 对话后，MemReader（自研 4B/1.7B 模型）抽取结构化记忆，实时异步执行（可配置同步）。
3. **工具轨迹沉淀（ToolTrajectoryMemory）**：Agent 核心。全链路记录工具调用（请求→结果→错误→重试→组合模式），每步执行后立即写入。
4. **任务结束后总结归档（TaskSummaryArchiving）**：长任务必备。任务标记为 success/fail/abort 后，LLM 生成任务级摘要、合并碎片、提炼决策点。
5. **后台自动整理（Dream 离线反思模块）**：核心能力。系统空闲/夜间/低负载时运行，输入全量历史记忆（不修改原始），执行合并重复/相似记忆、提炼实体和技能、关联跨会话记忆补全因果链、标记冷热记忆衰减低价值信息、生成 InsightMemory（洞察）和 DreamDiary（可审计日志）。**关键规则：Dream 不修改原始 raw 记忆，只生成新记忆/新技能；原始记忆永不丢失、永不改写。**

**Hermes-Agent 的 3 类产生渠道**：

1. **Agent 主动调用 memory 工具**：Agent 自己决定保存什么。工具 schema description 中编码了行为引导——何时保存（用户纠正你、分享偏好、你发现环境事实...）、保存优先级（用户偏好和纠正 > 环境事实 > 程序知识）、什么不该保存（不要保存任务进度，用 session_search 找那些）。支持 add（新增）、replace（短子串匹配替换）、remove（删除）。
2. **Nudge 后台审查**：计数器机制——每轮对话递增 `_turns_since_memory`，达到阈值（默认 10）时设置 `_should_review_memory = True`。但**如果 Agent 主动使用了 memory 工具，计数器重置为 0**——主动保存的 Agent 不需要被动审查。审查时在 daemon 线程中 fork 一个受限 Agent，只允许使用 memory 和 skill 管理工具。
3. **外部 provider 自动同步**：每轮推理后 `sync_all()` 将对话持久化到外部后端，具体提取逻辑由各 provider 自行实现。

### 💡 可借鉴要点

1. **memOS 的 Dream 模块——这是 oG-Memory 最大的缺失**。oG-Memory 目前完全依赖实时提取，没有离线优化能力。Dream 的核心价值不只是"合并碎片"（这可以用类型化写入策略部分替代），而是：
   - **提炼实体/技能**：从大量交互中自动发现高频模式和决策模板，生成 SkillMemory（可写入 ParametricMemory 层）
   - **生成洞察（InsightMemory）**：跨会话关联补全因果链，发现单次提取无法发现的深层规律
   - **衰减低价值信息**：不是简单的 TTL 过期，而是基于访问模式的智能衰减——标记冷记忆、降低其召回权重
   
   oG-Memory 应优先增加 Dream/离线反思模块，这是长期记忆质量的关键瓶颈。

2. **memOS 的 ToolTrajectoryMemory**：oG-Memory 的工具 I/O 只是补录到 session transcript 中作为提取输入（和其他对话消息一起进入提取流程），但 memOS 把工具轨迹作为**独立渠道**——全链路记录（请求→结果→错误→重试→组合模式），而且每步执行后立即写入（"执行即学习"）。oG-Memory 可以考虑将 tool 类型的提取从"对话后统一提取"改为"工具调用后立即写入初步记录"，确保工具经验不丢失——如果 after_turn 提取失败或进程中断，工具轨迹仍然有底。

3. **Hermes 的 Nudge 计数器**：不是每 turn 都跑提取（成本高），而是计数器驱动——每 N 轮自动触发审查。而且**Agent 主动使用了 memory 工具时计数器重置**——意味着"主动保存的 Agent 不需要被动审查"。oG-Memory 可以考虑类似的"智能触发"机制：不是每次 after_turn 都跑完整两阶段提取，而是先判断"本轮对话是否有值得提取的新信息"（比如用户是否说了偏好、是否纠正了 Agent、是否有新的环境发现），如果没有就跳过提取——既降低成本，又避免提取噪音。

4. **Hermes 的"策展即选择"理念**：Hermes 通过字符硬限制和 schema 优先级指导，迫使 Agent 在**写入时**就做出取舍——被保存的条目本身就是策展后的高价值内容。oG-Memory 的提取是全自动的（LLM 决定提取什么并全部入库），缺少写入时的策展压力——可以考虑在提取结果写入前加一个**价值评估**步骤（LLM 判断"这条信息是否值得长期保存"），而非默认 confidence > 0.5 就全部入库。confidence 只衡量"提取是否准确"，不衡量"信息是否有长期价值"。

---

## Q4：推理前如何选择正确记忆并注入上下文

| 维度 | **oG-Memory** | **mem0 2.0** | **memOS 2.0** | **Hermes-Agent** |
|------|--------------|-------------|--------------|-----------------|
| **检索策略** | **4 阶段流水线**：① QueryPlanner 将自然语言查询分类为 TypedQuery（memory/skill/resource，推断要检索哪类记忆）；② SeedRetriever 全局向量搜索 L0+L2 种子点，可选 BM25 混合融合；③ HierarchicalSearcher 从 L0/L1 种子点递归展开到 L2 叶节点（分数传播：`final = alpha * child + (1-alpha) * parent`），热度衰减（半衰期 7 天）；④ ResultRanker 过滤叶节点、去重、排序、top-k | 4 步：① 构造 retrieval query → ② metadata filter（按 scope 过滤 organization→user→session）→ ③ embedding similarity + entity enhancement（相关联的记忆） → ④ rerank（综合语义相似度 + BM25 + entity_bonus），得到最终 score | **6 步标准链路**：① 接入与意图理解（确定 user_id + 检索范围 + 模式 fast/fine/mixed） → ② 强过滤（仅 activated 进入候选池，archived/deleted 直接排除） → ③ 混合检索（向量 top K + BM25 补全精确匹配，去重合并） → ④ LLM 精排（综合语义相似度 + 时间衰减 + 访问频次 + 冲突消解） → ⑤ 去重与截断 → ⑥ 注入上下文 + 多模式输出 | **全量注入，不做选择**——不做向量搜索、关键词匹配、相关性评分等动态检索。所有记忆条目在 session 开始时一次性加载，全量注入 system prompt。设计有三重考量：①数据规模极小（500-600 tokens，全量注入成本低于一次检索）；②策展即选择（写入时已筛选，读取时不需要再筛选）；③prefix cache 稳定性（全量注入保证 system prompt 字节不变，保护缓存） |
| **层级展开** | L0 (abstract) → L1 (overview) → L2 (content) 递归展开：先找到路标级摘要，沿关系边展开到平衡信号，最终到全文细节。收敛检测：top-k URI 集连续 N 轮不变则停止 | 无层级——扁平向量检索，无摘要→全文的递进 | 无层级——只有激活/归档两级状态过滤，没有从摘要到全文的递进检索 | 无层级——全量注入不需要层级 |
| **注入方式** | **语义 slot 注入**：compose() 将检索结果按 5 个优先级 slot 组装——identity_context（优先级 0，profile+稳定偏好，不可降级）、session_context（优先级 1，结构化 session 状态）、episodic_context（优先级 5，archive 历史，可降级 content→overview→abstract）、task_context（优先级 5，当前任务）、retrieved_evidence（优先级 8，搜索结果 working set，可降级+可扩展）。注入到 `additionalContext`（Claude Code）或 system prompt 后缀（OpenClaw） | rerank 后的记忆作为 system prompt（长期信息）或 tool injection（动态查询信息）注入 Agent | 明文事实注入 prompt，每条附带来源（会话 ID）、时间戳、置信度。三种输出模式：matches（仅返回相关记忆事实）、instruction（补全为指令片段）、full_instruction（完整 prompt 模板） | **冻结快照注入 system prompt volatile tier**——两个语义角色块：`MEMORY (your personal notes) [34% — 748/2,200 chars]` 和 `USER PROFILE (who the user is) [25% — 344/1,375 chars]`。注入顺序：memory 在前（先"我知道什么"），user 在后（再"用户是谁"） |
| **角色注入** | slot 分区但没有显式角色标注——identity_context 混合了 profile 和 preference，不区分"Agent 的经验"和"用户的偏好" | 无角色注入 | tag/filter 纯手动（用户自定义标签用于分类/角色/项目隔离，**系统不自动生成 tag，不自动根据对话内容打 tag**，检索时需要手动在 filter 里指定 tag） | **显式语义角色标注**——两个块的 header 不是装饰："MEMORY (your personal notes)" 让模型解读为"我学到的教训"→产生遵循行为；"USER PROFILE (who the user is)" 让模型解读为"关于用户的事实"→产生适配行为 |
| **预算控制** | **TokenBudget 128K**：优先级层级分配——优先级 0（identity）先分配、优先级 1（session）次之、优先级 5（archive+task）再分配、优先级 8（working set）最后。预算不足时低优先级 tier 降级到 min_tokens。降级策略：content → overview → abstract | 无显式 token 预算——检索结果直接注入，依赖 rerank 后的 top-k 控制数量 | 截断到模型上下文窗口限制——去重合并高度相似记忆（保留最优版本）+ 按模型上下文窗口压缩到可用 token 预算 | 字符硬限制——MEMORY.md 上限 2,200 字符 ≈ 300-400 tokens，USER.md 上限 1,375 字符 ≈ 200 tokens。全量注入总成本约 500-600 tokens，远低于一次检索的开销。**没有动态预算分配**——空间满了就拒绝写入，Agent 必须先删旧的再加新的 |
| **prefix cache 考量** | **不考虑**——compose 每次返回内容可能不同，system prompt 每轮可能变化 | **不考虑** | **不考虑**——检索结果每轮可能不同 | **核心设计考量**——冻结快照保证 system prompt 在整个 session 内字节不变，保护 Anthropic `system_and_3` prefix cache。选择性召回（每轮内容不同）会导致缓存每轮失效，token 成本翻倍。唯一刷新时机是上下文压缩（压缩已使 prefix cache 失效，此时刷新快照不产生额外成本） |

### 💡 可借鉴要点

1. **Hermes 的"策展即选择 + 全量注入 + prefix cache 保护"闭环**——这是一个完整的设计闭环：
   - 字符硬限制迫使写入时策展 → 已保存的都是高价值 → 全量注入不需要检索 → 全量注入保证 system prompt 稳定 → 稳定保护 prefix cache → 降低 token 成本
   
   oG-Memory 的检索是复杂的 4 阶段流水线，但 Hermes 用一个极简方案在"记忆规模小"的场景下反而更优。oG-Memory 可以考虑**对 identity_context slot（profile + 稳定偏好）采用 Hermes 式全量注入**——这些信息量很小（几十条事实 ≈ 几百 tokens），且极其稳定（不随查询变化），全量注入既保护 cache 又确保关键信息不遗漏。其余 slot（episodic/task/retrieved_evidence）继续走检索流水线。

2. **memOS 的"fast/fine/mixed 三模式"**：不同场景对检索精度需求不同——快速对话用 fast（只向量检索），深度分析用 fine（向量+BM25+rerank），混合场景用 mixed。oG-Memory 目前只有一种检索模式，可以考虑增加模式选择参数。

3. **oG-Memory 自身的优势：语义 slot 注入 + 优先级降级**。其他三个项目要么全量注入（Hermes）要么扁平拼接（mem0/memOS），oG-Memory 的 5 个 slot + TokenBudget 优先级分配 + 降级策略（content → overview → abstract）是最精细的注入设计。**应保留并强化**——特别是在加入 Dream 模块后，identity slot 可以升级为"全量注入+冻结快照"模式，而检索 slot 继续走精细化路线。

---

## Q5：如何处理记忆的过期、冲突、合并、版本变化

| 维度 | **oG-Memory** | **mem0 2.0** | **memOS 2.0** | **Hermes-Agent** |
|------|--------------|-------------|--------------|-----------------|
| **写入策略** | **类型化策略**：不同记忆类型有不同的写入行为——profile 用 ProfilePolicy（固定 URI，新信息替换旧信息，因为用户状态会变化）；preference/entity/pattern 用 AggregateTopicPolicy（slug URI，追加式合并：existing + incoming，保留 provenance_ids）；event/case 用 AppendOnlyPolicy（每次创建新节点，不覆盖历史，timestamp_uuid 保证唯一）；skill/tool 用 SkillToolPolicy（固定 URI，累积式合并：最佳实践追加，usage_count++） | **无类型化策略**：LLM 判断新旧记忆关系后输出 add/update/delete/none 四种动作，没有按记忆类型区分写入行为 | **状态三值**（activated/archived/deleted）+ **两套可选冲突策略**（详见下方展开） | **字符硬限制驱动的策展压力**：没有 TTL，但空间满了必须删旧的。错误信息不只说"满了"，还列出当前所有条目+容量百分比+新条目字符数，让 Agent 有足够上下文做出策展决策（"这条新信息比旧条目 X 更有价值，我应该删除 X"） |
| **冲突检测** | **Safety Check**（提取阶段的语义冲突预防）：在 ReAct 循环提取中，如果 LLM 要写入一个已存在但未读过的节点，自动重读该节点内容并提醒 LLM 考虑现有内容——防止覆盖式冲突。**乐观锁**（并发层面的冲突控制）：写入时 `expected_version` 与实际版本比对，不匹配触发重试（最多 3 次指数退避） | **LLM 判断语义冲突**：新记忆写入时召回相似旧记忆，LLM 判断关系（add/update/delete/none），输出动作标签 | **写入时自动语义冲突检测**（详见下方展开） | **无语义冲突检测**——内建记忆没有检索能力，不做语义层面的冲突判断。**并发写入保护**（详见下方展开） |
| **冲突解决** | 乐观锁重试（最多 3 次）；类型化写入策略隐式解决语义冲突（profile 替换式、preference 追加式、event 只增不改） | linked_memory_ids 关联新旧记忆——不对冲突做显式处理，新旧记忆并存+链接 | **两套可选策略**（详见下方展开） | 漂移检测→拒绝修改+创建 `.bak.<timestamp>` 备份+返回详细错误信息和修复指导 |
| **过期机制** | **无 TTL**——依赖热度衰减（hotness_score 基于半衰期默认 7 天和访问计数衰减）影响检索排序权重，但不影响记忆生命周期（不会自动删除或归档低热度记忆） | **无 TTL**——依赖 recency replacement（近期记忆自然权重更高）和 memory decay（很久没用到的 importance 下降），只有显式调用 delete() 才会删除记忆 | **状态三值 + 热度标记 + 衰减**：activated 正常召回 → archived 默认不召回（合并/长期未访问时自动归档）→ deleted 永不召回（后台定时硬删除）。热度标记影响召回排序，非显式过期时间 | **无 TTL**——策展压力间接实现：空间满了必须删旧的，但删除是 Agent 主动决策而非系统自动过期 |
| **碎片合并** | preference/entity 用 AggregateTopicPolicy（追加式合并：existing overview + incoming overview → 新 overview，保留 provenance_ids 跨版本累积） | **无后台合并机制**——只有写入时的 linked_memory_ids 关联 | **Dream 模块合并**：合并重复/相似记忆（生成新记忆，旧记忆归档为 archived） | **去重**：加载时 dict.fromkeys() 保持顺序去重（先写的优先级更高）；写入时精确字符串匹配去重（语义相近但措辞不同的条目不视为重复） |
| **版本管理** | **扁平 version 数字**：每个 ContextNode metadata 维护 version 字段，写入时 expected_version 与实际版本比对。provenance_ids 跨版本累积追踪来源 | **无版本管理**——只有 linked_memory_ids 关联新旧记忆 | **完整版本链**（latest_override 策略下）：旧 memory_id 通过 obsoleted_by 字段指向新记录，旧记忆标记为 archived，新记忆设为 activated。可追溯从旧到新的完整演进链 | **"时间窗口"式版本管理**：磁盘文件 = 跨 session 持久版本；活跃状态 = session 内 working copy；冻结快照 = session start 时的"发布版本"。三者可能不一致（有意为之），唯一快照刷新时机是上下文压缩 |
| **并发安全** | 乐观锁 + Outbox 异步索引（写入节点后注册 OutboxEvent，异步 Worker 更新向量索引，事件持久化可跨进程重启恢复） | 无特殊并发控制 | 未提及 | **最完整**：文件锁（fcntl LOCK_EX）+ 重新读取（获取姐妹会话最新状态）+ 漂移检测（round-trip mismatch + entry-size overflow）+ 原子写入（tempfile.mkstemp + atomic_replace）+ 威胁扫描（12 种正则+10 种不可见 Unicode） |

### 关键概念展开

**memOS 2.0 的写入时语义冲突检测**：新记忆写入之前，系统会自动检索向量相似度比较高的已有记忆，然后由大模型判断这两个记忆之间的关系，输出四种标签：

- **CONFLICT**：新旧记忆互相矛盾（如"用户喜欢简洁回答" vs "用户偏好详细解释"）
- **DUPLICATE**：新旧记忆是同一件事的重复表述
- **COMPLEMENT**：新旧记忆互相补充（如"用户是工程师" vs "用户擅长 Python"）
- **NEW**：新记忆与已有记忆无关

如果是 CONFLICT，memOS 提供两套可选策略：

- **策略 A：latest_override（默认策略，"最新覆盖"）**：旧记忆标记为 archived（保留历史，不删除），新记忆设为 activated（进入主索引），建立版本链（旧 memory_id 的 obsoleted_by 字段指向新记录）。召回时默认只返回最新的 activated 版本，旧冲突记录不进上下文。
- **策略 B：preserve_parallel（可选策略，"共存模式"）**：新旧记忆均保留为 activated 状态，标记为"冲突对"（记录 conflict_with 字段），检索时按"时间+置信度"排序，最新/高置信度优先。不强制过滤旧记录，由下游业务决定如何使用。

如果是 DUPLICATE，新记忆写入后与旧记忆通过 linked_memory_ids 关联。如果是 COMPLEMENT，两条记忆共存互相补充。

**Hermes-Agent 的并发写入保护**：Hermes 面临的冲突场景全部是写入层面的（因为内建记忆没有语义检索能力，不做语义冲突检测）：

- **姐妹会话并发写入**：两个同时运行的 Hermes session 可能同时写同一个 MEMORY.md。保护机制：获取排他文件锁 → 重新从磁盘读取（获取姐妹写入的最新状态） → 如果检测到漂移则拒绝修改 + 创建备份 → 否则原子写入。
- **外部工具修改**（patch tool / shell / 手动编辑）：漂移检测两个信号——① round-trip mismatch（重解析+序列化不产生相同字节，说明文件被外部工具追加了自由格式文本）；② entry-size overflow（单条目超过存储总限制，说明外部工具把多条目文件变成了一个超长"条目"）。命中时创建 `.bak.<timestamp>` 备份（外部内容不丢失）+ 拒绝修改 + 返回修复指导。
- **原子写入**：`tempfile.mkstemp()` + `atomic_replace()`（os.replace()），读者永远看到完整文件（旧版本或新版本），不会看到空文件或半写文件。
- **威胁扫描**：所有写入经过 12 种正则模式 + 10 种不可见 Unicode 字符检测，覆盖 prompt injection、角色劫持、秘密泄露、SSH 后门等。命中则拒绝写入。

### 💡 可借鉴要点

1. **memOS 的"写入时语义冲突检测"**：oG-Memory 的冲突处理是写入层面的（乐观锁防并发），不是语义层面的——oG-Memory 有 `contradicts` 关系类型定义但从未主动生成过。建议在 `ContextWriter.write_candidates()` 中增加一个**写入前语义冲突检测**步骤：对每个 CandidateMemory 检索已有相似节点，由 LLM 判断关系并输出 CONFLICT/DUPLICATE/COMPLEMENT/NEW 标签，然后根据标签决定写入策略——CONFLICT 时根据配置选择 latest_override 或 preserve_parallel，DUPLICATE 时做去重合并，COMPLEMENT 时共存补充。这比 oG-Memory 当前"隐式通过类型化策略解决"的方式更精确——profile 的 ProfilePolicy 只做了"替换"这一种冲突处理，但 profile 信息也可能存在"补充"而非"矛盾"的情况。

2. **memOS 的两套冲突策略可选**：latest_override（新胜旧）和 preserve_parallel（共存）可选。oG-Memory 目前只有类型化策略（profile 替换、preference 追加），没有"共存模式"——对于某些场景（如用户说"我喜欢简洁回答"后又说"这次请详细解释"），这两条偏好其实是不同语境下的不同要求，preserve_parallel 保留两者让下游决策比强制覆盖更合理。可以考虑在 MergePolicy 中增加 preserve_parallel 选项。

3. **Hermes 的威胁扫描**：12 种正则+10 种不可见 Unicode 检测覆盖 prompt injection / 角色劫持 / 秘密泄露 / SSH 后门等。oG-Memory 完全没有这个安全层——如果未来开放用户手动写入记忆或 Agent 自由编辑记忆内容，这个安全层必不可少。至少应该在用户手动写入时做威胁扫描。

4. **Hermes 的"策展压力间接实现过期"**：没有 TTL，但空间满了必须删旧的。关键是系统不只返回"满了/没满"的二元信号，还返回**策展上下文**——当前所有条目列表、容量百分比、新条目字符数。这让 Agent 有足够信息做策展决策而非盲目删除。oG-Memory 的记忆量远大于 Hermes（可能上千条 vs 十几条），不能直接复制字符硬限制，但"**提供策展上下文而非二元拒绝**"的理念可以借鉴——比如在检索结果超过 token 预算时，不只截断，还返回被截断条目的摘要列表，让上层决定哪些值得保留。

5. **memOS 的"状态三值 + 容量上限"驱动记忆生命周期——oG-Memory 没有状态驱动召回也没有容量管理**：memOS 的 activated/archived/deleted 三值状态不是标签而是**召回的硬开关**——检索流程第一步强过滤（recall.py 所有查询路径强制 `status="activated"`），archived/deleted 记忆直接排除，根本不进候选池。这个状态开关配合多个自动触发源：（1）**冲突归档**——CONFLICT 且 latest_override 策略下旧记忆自动 `activated → archived`，新记忆 `generated → activated`；（2）**容量淘汰**——每种 memory_type 有硬上限（WorkingMemory 20 / LongTermMemory 1500 / UserMemory 480），达到 80% 阈值时 `remove_oldest_memory` 按时间排序删除最旧的（DETACH DELETE，不是 archived 而是硬删除）；（3）**Dream 维护**——DreamMemoryLifecycle 记录 last_hit_at / hit_count / usefulness_score / invalidated_by_feedback，长期未命中的记忆衰减归档、低 usefulness 归档、被反馈推翻的立即归档（目前是 placeholder，但字段和规则已经定义好）。这三层机制保证了候选池始终是"活的"——数量可控、内容有效、过时信息自动退出。oG-Memory 的现状对比鲜明：**hotness_score 衰减只影响排序权重**（`blended = (1-alpha)*semantic + alpha*hotness`），冷记忆排在后面但仍可以被召回；**NodeStatus 有 ARCHIVED 状态定义和 `archive_node()` 方法，OutboxWorker 也有处理 `ARCHIVE_CONTEXT` 事件的路径，但没有生产代码调用它们**——接口和处理路径都准备好了，却没有任何自动触发源；**没有容量管理**——记忆库无限增长，没有按类型设上限或淘汰机制。随着使用时间增长，记忆库会膨胀到大量低热度但仍可召回的历史记忆，挤占检索资源和 compose 的 token 预算。可以考虑三步渐进式改造：① **先加热度门槛**（最低成本）——在 HierarchicalSearcher 或 ResultRanker 中，hotness_score 低于阈值（如 0.05）的候选直接跳过不进结果集，等效于 memOS 的"archived 不进候选池"行为，但不需要改存储结构，只是检索层的过滤规则；② **再激活自动归档触发源**（中等成本）——把已有但未使用的 `archive_node()` 和 OutboxWorker 的 `ARCHIVE_CONTEXT` 处理路径激活起来，触发源包括：冲突且 ProfilePolicy 替换旧内容时旧节点自动归档（而非只标记 version）、用户 feedback 纠正后旧内容归档（Q2 第 4 点提到的 feedback 机制）、容量淘汰（每种 memory_type 设上限，超过阈值时最旧的归档而非硬删除，保留溯源能力）；③ **最后加 Dream 维护**（高成本，与 P0 Dream 模块同步）——Dream 维护时根据 hit_count 和 usefulness_score 将长期低价值记忆归档，与 memOS 的 DreamMemoryLifecycle 类似。三步不冲突，第①步可以先独立上线，第②步在加 feedback 机制时同步做，第③步等 Dream 模块完成后再对接。

---

## Q6：如何衡量记忆的好坏

| 维度 | **oG-Memory** | **mem0 2.0** | **memOS 2.0** | **Hermes-Agent** |
|------|--------------|-------------|--------------|-----------------|
| **评估数据集** | LoCoMo（多轮对话数据集，测试记忆检索和理解能力） | LoCoMo | **4 个学术 benchmark + 1 个自建 benchmark**（详见下方展开） | 未提及 |
| **核心指标** | 正确率提升 + token 消耗降低 + 提取质量（命中率、置信度分布）+ 检索质量（recall、precision、MRR）+ 组装质量（session 组装的 token 预算利用率） | **BLEU Score**（n-gram 匹配）+ **F1 Score**（Token-level Precision/Recall）+ **LLM Judge**（大模型判断 CORRECT/WRONG，衡量语义正确性） | **三类指标体系**（详见下方展开） | 未提及 |
| **对比基线** | 无记忆 baseline（数据待填） | LoCoMo 上的多种 baseline：RAG / full-context / chatgpt 内置记忆 / zep / langmem / 其他记忆平台 | **7 个对比基线**（详见下方展开）。官方 README 显示：LoCoMo 75.80、LongMemEval +40.43%、PrefEval-10 +2568%、PersonaMem +40.75% | 未提及 |
| **评估脚本** | 公开：tests/e2e/（eval_quality/eval_retrieval/eval）+ tests/benchmark/（extraction_quality/assemble_session） | 公开：evaluation/ 目录下有各记忆技术的实现（mem0/openai/zep/rag/langmem）+ metrics/ + evals.py + run_experiments.py | **公开且可一键运行**（详见下方展开） | 未提及 |
| **细分评估** | **三维度分别评估**：提取质量、检索质量、组装质量——可以定位问题在哪个环节 | 不细分——端到端评估，只看最终问答效果 | **按 benchmark 类型细分**：LoCoMo 按 4 个类别（single hop / multi hop / temporal reasoning / open domain）分别计算指标；LongMemEval 按 7 个场景类别细分；PrefEval 按偏好维度（违反偏好/承认偏好/幻觉偏好/有用回复）分别评估；PersonaMem 按主题类别分别计算 accuracy；LongBench v2 按难度（easy/hard）和长度（short/medium/long）+ 领域分别计算 | 未提及 |

### 关键概念展开

**memOS 2.0 的 5 个 benchmark 数据集**：

1. **LoCoMo**（学术 benchmark）：多轮对话数据集，测试记忆检索和理解能力。包含 4 个问题类别——single hop（单步推理）、multi hop（多步推理）、temporal reasoning（时间推理）、open domain（开放域问答）。数据文件 `locomo10.json` 在 evaluation/data/ 中提供。

2. **LongMemEval**（学术 benchmark）：长期记忆评估数据集，测试跨会话记忆能力。覆盖 7 个场景类别（具体类别名称需从数据集中读取，数据目录存在但数据文件未直接包含在仓库中——需从 HuggingFace 下载）。

3. **LongBench v2**（学术 benchmark）：长上下文理解 benchmark v2 版本，测试模型在长文档上的理解能力。按难度（easy/hard）和长度（short/medium/long）分层评估，还按领域（domain）分别计算 accuracy。

4. **PrefEval**（学术 benchmark）：偏好评估数据集，专门测试记忆系统对用户偏好的理解和使用能力。这是 memOS 特别强调的 benchmark——README 中显示 PrefEval-10 的提升为 +2568%（对比 baseline）。评估维度包括 4 个偏好相关指标（详见下方）。

5. **PersonaMem**（自建 benchmark）：memOS 自建的个性化记忆评估数据集，测试记忆系统在不同主题类别下的个性化表现。按主题类别（topic）分别计算 accuracy，并支持多次运行取标准差。这是调研同事提到的"自建 benchmark"。

**memOS 2.0 的三类指标体系**：

每个 benchmark 评估流程都包含统一的流水线：ingestion（将对话数据写入记忆系统）→ search（检索记忆）→ responses（基于检索结果生成回答）→ eval/judge（LLM 判断回答正确性）→ metric（计算各项指标）。指标分三类：

1. **词法指标（Lexical）**：F1、ROUGE-1/2/L、BLEU-1/2/3/4、METEOR——衡量生成回答与参考答案的文本重叠程度。

2. **语义指标（Semantic）**：BERT F1、Similarity——用 BERT 模型衡量语义相似度，比词法指标更能捕捉"表达不同但语义相同"的情况。

3. **LLM Judge 指标**：用大模型（gpt-4o-mini）判断回答是否正确（CORRECT/WRONG），多次运行取平均值和标准差。这是最接近"人类判断"的评估方式。

此外还有辅助指标：context_tokens（注入上下文的 token 数量，衡量记忆注入的 token 成本）、duration（response/search/total 的耗时 ms，衡量系统效率）。

**PrefEval 的 4 个偏好维度**——这是 memOS 评估体系中最独特的设计：

- **违反偏好（Violate Preference）**：助手回答是否违反了用户明确声明的偏好？LLM 判断回答中的推荐是否与用户偏好矛盾且没有承认用户偏好。
- **承认偏好（Acknowledge Preference）**：助手回答是否明确提及或引用了用户偏好？不只是"基于我们的对话..."这种模糊引用，而是需要具体指出偏好内容。
- **幻觉偏好（Hallucinate Preference）**：助手对用户偏好的复述是否与原始偏好一致？如果复述改变了含义或意图，则判定为幻觉。
- **有用回复（Helpful Response）**：助手是否提供了实质性、相关的回答，而非含糊地道歉或声称无法回答？

这 4 个维度不是简单的"回答正确率"，而是**偏好层面的细粒度评估**——分别考察记忆系统是否正确理解了用户偏好（不幻觉）、是否在回答中使用了偏好（承认）、是否没有违反偏好（不违反）、以及整体回答是否有帮助。

**memOS 2.0 的 7 个对比基线**：从 evaluation/scripts/ 中的代码可以看到，memOS 不仅评估自身（`memos-api` / `memos-api-online`），还与 6 个其他系统做对比：

1. **mem0** — mem0 基础版（无图存储）
2. **mem0-graph** — mem0 图存储版
3. **memobase** — MemoBase 记忆系统
4. **memu** — MemU 记忆系统
5. **supermemory** — SuperMemory 记忆系统
6. **zep** — Zep 记忆系统
7. **RAG** — 传统 RAG 方案（作为最基础的 baseline，用 chunk_size + num_chunks 参数化）

此外还有 **OpenAI 内置记忆** 作为 baseline（`run_openai_eval.sh` 中的 `locomo_openai.py`），和 **full-context**（全量对话传入 LLM，无记忆系统）作为参考。

**评估脚本的可复现性**：每个 benchmark 都有对应的 `run_*.sh` 一键运行脚本，流程统一为 ingestion → search → responses → eval → metric 五步。支持 `--lib` 参数切换不同记忆系统、`--workers` 参数控制并行度、`--top_k` 参数控制检索数量。PrefEval 的脚本还支持 `--add-turn` 参数（0/10/300），控制写入偏好的对话轮数——这是一个有趣的实验维度，可以研究"更多对话轮次是否带来更好的偏好记忆"。

### 💡 可借鉴要点

1. **memOS 的多 benchmark 评估体系**——这是所有项目中最全面的评估设计：4 个学术 benchmark（LoCoMo / LongMemEval / LongBench v2 / PrefEval）覆盖不同能力维度（多轮对话问答 / 跨会话长期记忆 / 长上下文理解 / 偏好理解使用），加上 1 个自建 benchmark（PersonaMem）测试个性化表现。oG-Memory 目前只在 LoCoMo 上评估——可以考虑扩展到 LongMemEval（跨会话记忆是 oG-Memory 的核心场景）和 PrefEval（偏好记忆是 oG-Memory 的 profile/preference 类型的核心价值）。

2. **memOS 的 PrefEval 偏好维度评估**——4 个偏好维度（违反/承认/幻觉/有用）不是简单的"回答正确率"，而是**偏好层面的细粒度评估**。oG-Memory 的 preference 类型记忆目前没有专门的评估维度——可以参考 PrefEval 的设计，增加偏好层面的评估：① 偏好是否被正确提取（不幻觉）；② 偏好是否被注入到回答中（承认）；③ 回答是否违反了已知偏好；④ 整体回答是否有帮助。

3. **mem0 和 memOS 的 LLM Judge 指标**：BLEU/F1 是字符串匹配指标，但语义正确性需要更深的判断——mem0 用 LLM Judge（CORRECT/WRONG），memOS 也用 LLM Judge（多次运行取平均+标准差）。oG-Memory 目前只统计"错误数"和"no info 回答占比"，缺少语义正确性评估——可以考虑增加 LLM Judge 步骤，并用多次运行取标准差来衡量评估结果的稳定性。

4. **细分评估是 oG-Memory 的优势**：提取/检索/组装三个维度分别评估，可以定位问题在哪个环节。memOS 的评估是端到端的（ingestion→search→response→judge→metric），没有区分提取/检索/组装各自的质量。**应保持并强化**——特别是可以在 memOS 的 LoCoMo/LongMemEval 流水线上叠加 oG-Memory 的三维度细分评估。

5. **memOS 的 add-turn 实验维度**：PrefEval 支持 `--add-turn` 参数（0/10/300），控制写入偏好的对话轮数。这是一个很好的实验设计——可以研究"更多对话轮次是否带来更好的偏好记忆"。oG-Memory 可以借鉴这种参数化实验维度，比如研究"提取轮数阈值（每 N 轮提取一次 vs 每次 after_turn 提取）对记忆质量的影响"。

---

## 优先级排序：oG-Memory 最应借鉴的改进

| 优先级 | 改进项 | 来源 | 预期收益 | 实现难度 |
|--------|--------|------|---------|---------|
| **P0** | **Dream/离线反思模块** | memOS | 消除碎片、提炼技能、生成洞察——长期记忆质量的关键瓶颈 | 高（新模块） |
| **P1** | **写入时语义冲突检测**（检索相似旧记忆→LLM 判断 CONFLICT/DUPLICATE/COMPLEMENT/NEW→根据标签决定写入策略） | memOS | 防止矛盾记忆共存、自动识别 DUPLICATE/COMPLEMENT | 中（在 ContextWriter 中增加检索+LLM 判断步骤） |
| **P1** | **identity slot 全量注入 + 冻结快照** | Hermes | 保护 prefix cache、确保关键信息不遗漏、降低检索成本 | 低（只改 compose 注入逻辑） |
| **P2** | **记忆语义角色标注**（注入 prompt 时显式标注 "Agent 的经验" vs "用户的偏好"） | Hermes | 引导模型对两类信息产生不同行为倾向（遵循 vs 适配） | 低（改 prompt 模板） |
| **P2** | **Nudge 智能触发**（判断"本轮是否有值得提取的新信息"后决定是否执行提取，而非每 turn 都跑完整两阶段） | Hermes | 降低提取成本、避免提取噪音 | 中（改 after_turn 触发逻辑） |
| **P2** | **LLM Judge 语义正确性评估** | mem0 | 评估语义层面的回答正确性，而非仅字符串匹配 | 低（增加评估步骤） |
| **P3** | **Plugin Hook 点**（提取后干预 SEARCH_MEMORY_RESULTS / 检索结果后干预 MEM_READER_PRE_EXTRACT 等） | memOS | 开放第三方干预核心流程的能力，不需要改 oG-Memory 代码本身 | 中（增加 Hook 注册机制） |
| **P3** | **preserve_parallel 冲突策略** | memOS | 某些场景保留矛盾记忆共存比强制覆盖更合理 | 低（MergePolicy 增加选项） |
| **P3** | **检索模式选择**（fast/fine/mixed） | memOS | 不同场景不同精度需求 | 低（增加配置参数） |
| **P3** | **ParametricMemory 层**（从大量交互提炼的不可编辑决策模板，Dream 的天然产出） | memOS | "技能式"记忆——内化后无需检索的直觉能力 | 高（新载体+Dream 产出对接） |

---

## 相关链接

- [[ogmemory-deep-analysis]] — oG-Memory 六维度深度分析（本对比的基准）
- [[ogmemory]] — oG-Memory 项目概述
- [[locomo]] — LoCoMo 数据集