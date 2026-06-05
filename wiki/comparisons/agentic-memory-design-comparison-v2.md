---
name: agentic-memory-design-comparison-v2
description: 五个 agentic memory 项目（oG-Memory / mem0 / memOS / Hermes-Agent / Cognee）在六个核心设计问题上的对比总结，V2 版本新增 Cognee 对比列
tags: [ogmemory, mem0, memos, hermes-agent, cognee, agentic-memory, comparison, design]
created: 2026-06-04
updated: 2026-06-04
---

# Agentic Memory 设计思路对比 V2

> 来源：GitCode Discussion #9（oG-Memory / mem0 / memOS / Hermes-Agent 四项目精简对比）+ Discussion #8（Cognee 六问题调研）+ V1 文档（Q4-Q6 其他项目细节）
> 对比项目：**oG-Memory** · **mem0 2.0** · **memOS 2.0** · **Hermes-Agent** · **Cognee**
> 目标：提炼最有差异化、最凸显设计思路的部分，提出可借鉴要点用于完善 oG-Memory

---

## Q1：如何对接 Agent

| 维度                       | **oG-Memory**                                                                                                                                                   | **mem0 2.0**                                                                                                                             | **memOS 2.0**                                                                                                                                                                                                                  | **Hermes-Agent**                                                                                                                                         | **Cognee**                                                                                                                                                                                                                                                                                                                                                            |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **架构拓扑**                 | 独立 HTTP 服务（out-of-process），Agent 通过中间脚本调用 API                                                                                                                   | 独立 SDK/服务（out-of-process），Agent 以工具形式接入                                                                                                  | 独立服务（out-of-process），三种接入形态并行                                                                                                                                                                                                  | **in-process**：`MemoryStore` 直接作为 Agent 实例的 Python 属性持有，Agent 内部直接操作，无需网络调用                                                                              | 独立服务（out-of-process），Agent 通过中间脚本调用 SDK 或 HTTP API。有 Local 模式（SDK 直调）和 Cloud 模式（HTTP API 代理）                                                                                                                                                                                                                                                                          |
| **已对接的 Agent**           | **2 个**：① Claude Code（`claude-plugin/`）② OpenClaw（`openclaw_context_engine_plugin/`）                                                                            | **2 个**：① Claude Code（`.mcp.json` + `hooks/hooks.json` + `skills/`）② OpenCode（`opencode.json` + `opencode-mem0.ts` + `opencode-skills/`） | **3 类接入形态**：① SDK/REST API（任何编程语言）② MCP（已支持 Cursor、Coze Space）③ Plugin（已对接 OpenClaw 和 Hermes Agent）                                                                                                                            | **1 个**：Hermes-Agent 自身（内建组件）。通过 `MemoryProvider` ABC 可桥接外部服务                                                                                            | **2 个**：① Claude Code（`cognee-integrations/claude-code/`）② OpenClaw（`cognee-openclaw` npm 插件）                                                                                                                                                                                                                                                                         |
| **各 Agent 的 Hook（自动触发）** | Claude Code 4 个 Hook：UserPromptSubmit→检索注入、PostToolUse→I/O 补录、Stop→after_turn、PreCompact→同步防丢失。OpenClaw 4 个 lifecycle method：assemble/afterTurn/compact/dispose | Claude Code 6 个 Hook：Setup/SessionStart/UserPromptSubmit/PreToolUse/PostToolUse/PreCompact。OpenCode lifecycle hooks 嵌入 TypeScript        | **Plugin 的 8 个精细 Hook**：ADD_BEFORE/ADD_AFTER、SEARCH_BEFORE/SEARCH_AFTER、SEARCH_MEMORY_RESULTS、MEM_READER_PRE_EXTRACT、MEMORY_ITEMS_AFTER_FINE_EXTRACT、DREAM_BEFORE_PERSIST/DREAM_AFTER_PERSIST。**MCP 和 SDK/REST API 没有任何 Hook** | Hermes 内建拦截点：tool_executor 拦截 memory 工具→触发 on_memory_write()；system_prompt 拼 frozen snapshot；conversation_loop Nudge 计数器；background_review fork 受限 Agent | Claude Code 5 个 Hook：UserPromptSubmit→session-context-lookup（搜索历史记忆注入）、PostToolUse→store-to-session（工具结果存缓存）、Stop→store-to-session --stop、PreCompact→pre-compact（保存锚点）、SessionEnd→sync-session-to-graph（持久化到图谱）。OpenClaw 4 个 lifecycle：registerService.start→/remember、before_prompt_build→/recall、agent_end→/remember+/update+/forget、session_end→/remember+/improve |
| **各 Agent 的 Skill/工具**   | Claude Code 2 个 Skill：/og-compose、/og-add-history。OpenClaw 无额外 Skill                                                                                            | Claude Code 3 个 Skill：/mem0:remember、/mem0:tour、/mem0:peek。OpenCode Skills 在 opencode-skills/                                            | MCP 暴露工具（add_memory/search_memories/chat/create_cube等）。SDK 方法（add_message/search_memory/chat/create_knowledgebase等）。Plugin 无额外 Skill                                                                                           | Hermes memory 工具：add/replace/remove + session_search                                                                                                     | Claude Code 3 个 Skill：cognee-remember（写入图谱）、cognee-search（跨会话搜索，可按类别过滤）、cognee-sync（临时记忆升级为永久图谱）。还有 sub-agent 机制：cognee-recall.md 定义跨会话永久图谱搜索子 agent                                                                                                                                                                                                                  |

### 💡 可借鉴要点

1. **Cognee 的 sub-agent 检索机制——oG-Memory 的检索只在主对话流中执行，缺少"深层检索可以 offload 到独立子 agent"的能力**：Cognee 的 Claude Code 插件定义了 `cognee-recall.md`，当主 agent 需要深层/跨会话搜索时，可以 spawn 一个专用的轻量检索子 agent——子 agent 只负责执行检索、返回结果给主 agent，用户完全不感知。这和 UserPromptSubmit Hook 注入的即时上下文检索是互补关系：Hook 只搜当前 session 的短期缓存，sub-agent 搜跨会话的永久图谱知识。oG-Memory 目前的 `/og-compose` Skill 是在主对话流中同步执行的——Agent 调用 compose 后拿到检索结果继续对话，检索耗时直接计入用户等待时间。如果检索需要多步推理（比如先搜一个关键词、发现线索后再沿关系边展开、再搜下一个关键词），多轮检索在主对话流中会显著增加延迟和 token 消耗。可以考虑：① 为 Claude Code 插件增加一个 `og-deep-search` Skill，定义为一个可被 spawn 的子 agent——只允许调用 compose/read 等 oG-Memory API，返回结构化检索结果给主 agent，用户不感知中间过程；② 在 compose 流水线中增加"深度模式"——QueryPlanner 判断 query 需要多步推理时，不再在主对话流中一次同步执行，而是 spawn 子 agent 执行多轮 HierarchicalSearcher 展开再汇总返回。好处是：主对话流不被长时间检索阻塞、子 agent 可以跑多轮推理而不消耗主对话的 token budget、用户感知到的是"回答更准确"而非"等了很久"。

---

## Q2：记忆对象模型如何构建

| 维度           | **oG-Memory**                                                                                                                             | **mem0 2.0**                                                  | **memOS 2.0**                                                                               | **Hermes-Agent**                                                                                                                                     | **Cognee**                                                                                                                                                                                                    |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- | ------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **记忆分类**     | **7 类 YAML 定义**：profile/preference/entity/event/case/pattern/skill/tool。**增加新类型只需加 YAML 文件，不改代码**                                         | **3 类 enum**：Procedural/Episodic/Semantic。定义但不强制约束，实际存储不按类型分区 | **6 类内容 + 3 层载体**（6 类内容指品类划分：①对话类②用户画像类③工具调用类④文件/URL/图片类⑤反馈与修正类⑥系统与元信息；3 层载体指认知分层：明文/激活/参数） | **2 个 target**：`memory`（Agent 个人笔记——"我学到的教训"）和 `user`（用户画像——"关于用户的事实"）。语义角色截然不同                                                                      | **两层分类**：短期记忆（QAEntry/TraceEntry/FeedbackEntry，按 user_id+session_id 隔离）+ 长期记忆（知识内容：EntityType/Entity/Event；技能工具：Skill/SkillRun/SkillImprovementProposal/Tool；摘要：TextSummary/CodeSummary/GlobalContextSummary） |
| **层级结构**     | **三级索引**：每个 ContextNode 对应 L0(abstract~100tokens)/L1(overview~500tokens)/L2(content~5000tokens) 三个向量索引记录，检索时从 L0/L1 种子点递归展开到 L2 叶节点       | 扁平：只有 embedding 索引一层，无层级                                      | **3 层载体**（详见下方展开）+ 3 种 backend（详见下方展开）                                                      | **四层栈**：Layer1 MemoryStore（策展式 MEMORY.md/USER.md）→ Layer2 SessionDB（SQLite 原始对话）→ Layer3 ContextCompressor（工作记忆缓存）→ Layer4 External Provider（可选外部服务） | **图结构而非嵌套层级树**。有动态生长的摘要树（TextSummary→GlobalContextSummary→Root），但层数由数据量决定、不固定。检索不做逐层 drill-down，而是 Root+top-3 bucket+详细 triplet 一次性组合为上下文前缀                                                                   |
| **Scope 区分** | **user-scope vs agent-scope**：profile/preference/entity/event/pattern 的 owner_scope 是 user，case/skill/tool 是 agent。URI 中 owner_space 字段区分 | user_id/agent_id/run_id 在 metadata 中标注，但不强制分区                 | user_id 隔离；tag 纯手动；知识库与记忆是两套独立存储                                                            | **语义角色区分而非 scope 区分**：memory→"我学到的教训"，user→"关于用户的事实"                                                                                                 | **不区分 user/agent scope**。scope 是 dataset 级，Agent 就是带 parent_user_id 的 User，多 Agent 共享通过 Dataset ACL 权限实现                                                                                                      |

### 关键概念展开

**memOS 2.0 的 6 类内容 × 3 层载体 × 3 种 backend——三个独立的分类维度，实现上有配套关系但不是一对一映射**：

memOS 把记忆分成了三个独立的分类维度：
- **内容维度（6 类内容 / memory_type）**——"存的是什么东西"：对话类、用户画像类、工具调用类、文件/URL/图片类、反馈与修正类、系统与元信息。对应 `TreeNodeTextualMemoryMetadata.memory_type` 字段，有 10 个枚举值（WorkingMemory、LongTermMemory、UserMemory、OuterMemory、ToolSchemaMemory、ToolTrajectoryMemory、RawFileMemory、SkillMemory、PreferenceMemory、Context）
- **载体维度（3 层载体）**——"处于什么认知状态"：明文记忆（"我知道什么"）、激活记忆（"我现在在想什么"）、参数记忆（"我能做什么"）。对应 `GeneralMemCube` 的 4 个子存储实例（`text_mem`、`act_mem`、`para_mem`、`pref_mem`）
- **backend 维度（3 种 backend）**——"用什么技术存储"：NaiveTextMemory（纯内存）、GeneralTextMemory（向量数据库）、TreeTextMemory（图数据库，最完整）

![image.png](https://raw.gitcode.com/user-images/assets/9439076/e98fd1e2-495f-41ca-a673-49c165742dd9/image.png 'image.png')

**6 类内容不只是标签，它影响了全链路处理**：

- **写入时**：`MemoryManager._process_memory()` 根据 memory_type 决定写入路径——WorkingMemory/LongTermMemory/UserMemory/OuterMemory 会**双写**（同时创建一个 WorkingMemory 临时节点 + 一个永久 graph memory 节点），而 ToolSchemaMemory/ToolTrajectoryMemory/RawFileMemory/SkillMemory/PreferenceMemory **只写 graph memory**，不创建 WorkingMemory 临时节点。这意味着不同内容类型的写入路径不同，不是简单地打标签然后统一入库。
- **容量管理**：每种 memory_type 有**独立的硬上限**（WorkingMemory 20 / LongTermMemory 1500 / RawFileMemory 1500 / UserMemory 480），达到 80% 阈值时 `remove_oldest_memory` 按时间排序硬删除最旧的。不同内容类型有不同的容量配额，防止某类记忆膨胀挤占其他类的空间。
- **检索时**：`GraphMemoryRetriever.retrieve()` 的 `memory_scope` 参数按 memory_type 分区检索——WorkingMemory 全量返回 top-k，其他类型走混合检索（向量+图+BM25+全文），每种 memory_type 有独立的检索池。所有检索路径强制 `status="activated"` 过滤。
- **Dream 时**：`TargetMemoryType` 定义了 Dream 的写入目标——LONG_TERM、SKILL、PREFERENCE、PROFILE、INSIGHT、DREAM_DIARY，不同内容类型由 Dream 提炼后路由到不同的目标载体。
- **反馈时**：`ArchivedTextualMemory.update_type` 有 `feedback` 选项，专门标记用户修正来源，与 `conflict`/`duplicate`/`extract`/`unrelated` 区分。

**6 类内容和 3 层载体之间的配套关系**：

所有 6 类内容**首先都写入明文记忆**（text_mem），memory_type 作为 `TreeNodeTextualMemoryMetadata.memory_type` 元数据标签区分品类——所以"内容维度"和"载体维度"不是一对一映射，而是**内容维度的所有品类都住在明文记忆这一层载体里**，memory_type 只是元数据标签。但有两个例外打破了这个规则：

1. **PreferenceMemory 有独立的 pref_mem 子存储**：偏好类内容不仅在 text_mem 中以 `memory_type=PreferenceMemory` 存储，还有一套**完全独立的 pref_mem 存储和处理管线**——有自己的 `PreferenceTextualMemoryMetadata`（含 preference_type/preference/dialog_id/original_text 等专化字段）、有自己的 `get_memory()` 提取方法、有自己的 `search()` 检索方法、有自己的 `add()` 写入方法。写入时 `core.py` 并行调用 `process_textual_memory()` 和 `process_preference_memory()`——同一个对话消息既写入 text_mem 又写入 pref_mem。检索时也并行搜索两个子存储。这意味着偏好类内容在 memOS 中是**双重存储**：一份是 text_mem 中的通用明文记忆（可被常规检索命中），一份是 pref_mem 中的专化偏好记忆（有自己的提取和检索逻辑）。

2. **WorkingMemory 是过渡态而非持久态**：WorkingMemory 不是"第 5 类内容"，而是 LongTermMemory/UserMemory/OuterMemory 在写入时创建的**临时快照**——async 模式下先写入 WorkingMemory（立即生效），后由 mem_reader 提炼为 LongTermMemory/UserMemory 再合并覆盖。sync 模式下 `_cleanup_working_memory()` 会在每次 add 后删除超出上限的 WorkingMemory。所以 WorkingMemory 的生命周期是"临时→被永久节点替代→清理"，不是独立的内容品类。

**3 种 backend 只服务于明文记忆（text_mem 和 pref_mem）**：激活记忆（act_mem）有 KVCache/VLLMKVCache 两种实现，参数记忆（para_mem）有 LoRA 一种实现（目前 placeholder）。部署时在 MemCube 配置中分别指定各子存储的 backend——明文记忆选 Naive/General/Tree 之一，激活记忆选 KVCache/VLLMKVCache 之一，参数记忆选 LoRA，偏好记忆也选 Naive/General/Tree 之一。

**memOS 2.0 的三层载体**：这是 memOS 2.0 最独特的设计，不是简单的冷热分层，而是**认知模式的分层**：

1. **明文记忆（Plaintext Memory）**——"磁盘式"记忆：可读写、可检索、可编辑的持久化存储。存对话摘要、用户偏好、事实、文档内容等文本信息。格式是 TreeTextMemory（图结构+向量，底层用 Neo4j）。状态有 activated/archived/deleted 三种。这是"我知道什么"——我积累的知识库。**所有 6 类内容都住在这层**，靠 memory_type 元数据标签区分品类。
2. **激活记忆（Activated Memory）**——"RAM 式"记忆：运行时高速缓存。存最近几轮对话、KV Cache、临时上下文。速度最快、容量有限、会话结束后可降级为明文记忆。这是"我现在在想什么"。
3. **参数记忆（Parametric Memory）**——"技能式"记忆：从大量交互中提炼成"技能/知识"的长期记忆（类似模型微调的效果）。存高频模式、决策模板、工具调用组合、领域技能。**不可直接编辑**，只能通过大量交互提炼。调用效率最高。这是"我能做什么"——内化后无需检索的直觉能力。

**Hermes-Agent 的双态设计**：Hermes 的内建记忆在同一份数据上维护两种状态：

- **活跃状态**（memory_entries / user_entries）：工具读写此状态，每次变更立即持久化到磁盘。
- **冻结快照**（_system_prompt_snapshot）：session 开始时一次性从磁盘"拍摄"，注入 system prompt 后在整个 session 内**字节不变**。保护 Anthropic prefix cache——system prompt 不变则缓存不失效，token 成本不翻倍。

两者在 session 中段可能不一致（有意为之）——活跃状态保证 Agent 能看到自己的变更，冻结快照保证 prefix cache 稳定。唯一刷新时机是上下文压缩。

**Cognee 的短期 vs 长期记忆**：

- **短期记忆**：存到 Redis 或文件系统缓存，按 (user_id, session_id) 键值隔离。session 结束后消失，可通过 `improve()` 持久化。
  - QAEntry：问答对话轮次（问题+上下文+回答+评分）
  - TraceEntry：Agent 工具调用步骤（参数+返回值+状态）
  - FeedbackEntry：对已有 QA 的显式评分+文字反馈
- **长期记忆**：存到知识图谱（关系数据库+向量数据库+图数据库），跨 session 共享，存入后一直存在直到调用 `forget()` 主动删除
  - 知识内容：EntityType/Entity/Event（is_a/SUMMARIZED_IN 语义关系）
  - 技能工具：Skill/SkillRun/SkillImprovementProposal/Tool
  - 摘要：TextSummary/CodeSummary/GlobalContextSummary
- **短期→长期转换**：通过 `improve()` 将短期记忆转换为长期图谱知识；短期更新会影响长期检索优先级；`improve()` 时将长期增量同步到短期作为背景补充

### 💡 可借鉴要点

1. **memOS 的"反馈与修正类"——oG-Memory 缺少用户修正驱动的记忆进化机制**：memOS 把"反馈与修正"作为独立的记忆品类（第 5 类），用户说"你记错了"时，修正不是简单覆盖旧记忆，而是有独立的 `mem_feedback` 模块处理——ArchivedTextualMemory 的 `update_type` 字段显式标记为 `feedback`，旧内容归档为 history 条目保留可追溯性，新内容设为 activated。oG-Memory 没有这条通道——用户纠正后修正只能通过下次 after_turn 提取来覆盖旧信息，但没有"旧信息被用户修正了"的显式标记，也没有修正来源追踪。可以考虑：① 增加 `feedback` 类型 YAML schema（独立提取 prompt+写入策略）；② 在 provenance_ids 中增加 `provenance_type: feedback | extraction | merge` 标记。

2. **Cognee 的 SkillRun + SkillImprovementProposal——oG-Memory 缺少技能执行的显式反馈闭环**：Cognee 的 SkillRun 直接写入永久图谱（不经过 session 缓存），记录选了哪个技能、调了什么工具、成功了没、用户反馈如何。SkillImprovementProposal 基于多个 SkillRun 的统计数据提出修改 Skill 的提案。这形成了一个闭环：执行→记录→统计→改进提案→下次成功率提高。oG-Memory 的 skill 类型只有 tool_builder 提取的"最佳实践"字段（best_for/optimal_params/common_failures），但没有执行记录和改进提案——只知道"这个工具可以这样用"，不知道"上次这样用了，效果如何"。可以考虑增加 SkillRun 记录和 SkillImprovementProposal 机制。

---

## Q3：记忆怎么产生

| 维度        | **oG-Memory**                                                                                                                            | **mem0 2.0**                                            | **memOS 2.0**                                                                                             | **Hermes-Agent**                                                 | **Cognee**                                                                                                                |
| --------- | ---------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| **产生渠道数** | 1 类：对话后自动提取（after_turn / PreCompact / SessionCommit / Bootstrap 四个触发时机，但都是"对话后 LLM 提取"这一种机制）                                             | 2 类：① Agent/用户主动调用 memory.add() ② Hook 每 3 turns 自动触发提取 | **5 类**（详见下方展开）                                                                                           | **3 类**（详见下方展开）                                                  | **4 类**：① 用户手动 remember() ② 工具结果沉淀（先存 cache，每N步持久化到图谱）③ 后台整理 improve() ④ ❌ 不支持对话自动抽取（需显式调用 remember）                      |
| **提取方法论** | **两阶段 + 双重运行**：Phase1 便宜 LLM 识别可提取 span；Phase2 对每个 span 用 function calling 结构化提取，跑两次取置信度高/内容丰富的合并。confidence<0.5 丢弃。另有 ReAct 循环（Lazy 模式） | **LLM 单次提取**：直接整体提取→结构化输出→向量化→去重→存储                     | **MemReader 5 步流水线**：①捕获 raw dialog→②MemReader（自研 4B/1.7B 抽取模型）→③Embedding 去重+LLM 校验→④分类短期/长期→⑤写入 MemCube | **Nudge 后台审查**：主对话完成后 fork 受限 Agent 审查对话，审查提示词关注"用户是否透露了值得记住的信息" | **单次 LLM 调用**，没有多轮迭代。去重有 DataPoint 的 Dedup() 字段注解（identity_fields 去重），去噪靠用户反馈/频率权重过滤低质量节点。**不支持对话自动抽取**——需显式调用 remember() |
| **自动化程度** | **全自动**，不支持用户手动写入                                                                                                                        | **全自动为主**，用户可通过 `/mem0:remember` 手动写入                   | **全自动为主**，用户可通过 add_memory API 手动写入（即时入库、不经过抽取）                                                           | **Agent 主动为主**（Agent 决定何时调用 memory 工具），系统自动为辅（Nudge 每 N 轮触发审查）   | **半自动**。remember() 需用户主动调用，但评分反馈和工具结果记录是自动的                                                                               |
| **离线处理**  | **无**——只有实时提取                                                                                                                            | **无**——没有后台合并/去重/反思机制                                   | **有 Dream 模块**（详见下方展开）                                                                                    | **无**——但有策展压力间接实现过期                                              | **有后台整理**（improve() 做持久化、权重、索引、实体描述合并），但**需手动触发**，不存在自动周期优化或离线反思。没有 dreaming                                              |
| **工具轨迹**  | PostToolUse Hook 仅将有副作用工具 I/O 补录到 session transcript，tool 类型有专门提取 schema 但没有独立存储渠道                                                       | PostToolUse Hook 自动捕获工具 I/O，但没有独立的工具轨迹记忆类型              | **独立渠道 ToolTrajectoryMemory**：全链路记录请求→结果→错误→重试→组合模式，每步执行后立即写入                                             | 无独立工具轨迹渠道——session_search 可检索但不做结构化提炼                            | **有 TraceEntry**（session 级：参数+返回值+状态）和 **SkillRun**（直接写入永久图谱：选了哪个技能+调了什么工具+成功否+用户反馈），但 TraceEntry 是短期记忆会消失                |
| **任务总结**  | **无**——session_archive 是压缩产物                                                                                                             | **无**                                                   | **有 TaskSummaryArchiving**：任务闭环后 LLM 生成任务级摘要、合并碎片、提炼决策点                                                   | **无**——session_search 可检索但不做任务级总结                                | **无**                                                                                                                     |

### 关键概念展开

**memOS 2.0 的 5 类产生渠道**：

1. **用户主动写入（User Direct Memory）**：即时入库、**不经过 LLM 抽取**，用于高置信度信息
2. **对话后自动抽取（Chat-as-Learning）**：MemReader（自研 4B/1.7B 模型）实时异步抽取
3. **工具轨迹沉淀（ToolTrajectoryMemory）**：全链路记录，每步执行后立即写入（"执行即学习"）
4. **任务结束后总结归档（TaskSummaryArchiving）**：任务闭环后 LLM 生成任务级摘要
5. **后台自动整理（Dream 离线反思模块）**：核心能力。合并重复/相似记忆、提炼实体和技能、关联跨会话记忆补全因果链、生成 InsightMemory 和 DreamDiary。**Dream 不修改原始 raw 记忆，只生成新记忆/新技能**

**Hermes-Agent 的 3 类产生渠道**：

1. **Agent 主动调用 memory 工具**：工具 schema description 编码了行为引导——何时保存、保存优先级、什么不该保存
2. **Nudge 后台审查**：计数器机制——Agent 主动使用了 memory 工具时计数器重置为 0
3. **外部 provider 自动同步**：每轮推理后 `sync_all()` 持久化到外部后端

**Cognee 的产生流程**：

```
Agent (Claude Code)
    │
    │  hook（自动） 或 skill（主动）
    v
  中间脚本（scripts/，Python，含 cognee SDK + aiohttp）
    │
    ├─ Local 模式 ─→ cognee.remember / cognee.recall / cognee.improve（SDK 直调）
    │
    └─ Cloud 模式 ─→ HTTP API 端点（后端服务器代理到 cognee 服务）
```

### 💡 可借鉴要点

1. **memOS 的 ToolTrajectoryMemory**：oG-Memory 的工具 I/O 只是补录到 session transcript 中作为提取输入，但 memOS 把工具轨迹作为**独立渠道**——全链路记录，而且每步执行后立即写入（"执行即学习"）。oG-Memory 可以考虑将 tool 类型的提取从"对话后统一提取"改为"工具调用后立即写入初步记录"，确保工具经验不丢失——如果 after_turn 提取失败或进程中断，工具轨迹仍然有底。

2. **Hermes 的"策展即选择"理念**：Hermes 通过字符硬限制和 schema 优先级指导，迫使 Agent 在**写入时**就做出取舍——被保存的条目本身就是策展后的高价值内容。oG-Memory 的提取是全自动的（LLM 决定提取什么并全部入库），缺少写入时的策展压力——可以考虑在提取结果写入前加一个**价值评估**步骤（LLM 判断"这条信息是否值得长期保存"），而非默认 confidence > 0.5 就全部入库。confidence 只衡量"提取是否准确"，不衡量"信息是否有长期价值"。

3. **Cognee 的 SkillImprovementProposal 自改进闭环**：Cognee 的 SkillRun（技能执行记录）→ SkillImprovementProposal（改进提案）形成了一个"执行→记录→统计→改进→下次成功率提高"的闭环。oG-Memory 的 skill 类型只有"最佳实践"字段，没有执行反馈——只知道"这个工具可以这样用"，不知道"上次这样用了，效果如何"。可以考虑在 skill 类型中增加 execution_log 字段，记录每次使用的成败和反馈，并定期提炼改进提案。

---

## Q4：推理前如何选择正确记忆并注入上下文

| 维度                  | **oG-Memory**                                                                                                                                 | **mem0 2.0**                                                                          | **memOS 2.0**                                                               | **Hermes-Agent**                                                                      | **Cognee**                                                                                                                                                                                                                                                                                                                                |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **检索策略**            | **4 阶段流水线**：① QueryPlanner 分类 TypedQuery ② SeedRetriever 全局向量搜索 L0+L2 种子点 ③ HierarchicalSearcher 递归展开到 L2 叶节点（分数传播+热度衰减）④ ResultRanker 过滤去重排序 | 4 步：构造 retrieval query→metadata filter→embedding similarity+entity enhancement→rerank | **6 步标准链路**：①意图理解 ②强过滤（仅 activated）③混合检索（向量+BM25）④LLM 精排 ⑤去重截断 ⑥注入上下文+多模式输出 | **全量注入，不做选择**——所有记忆条目在 session 开始时一次性加载全量注入 system prompt。数据规模极小（500-600 tokens）      | **向量+图混合为主**，也有纯向量/纯关键词/纯图等多种策略。核心算法是 **Brute-Force Triplet Search**：① query 嵌入向量在 5 个向量集合中搜索（Entity_name, TextSummary_text, EntityType_name, DocumentChunk_text, EdgeType_relationship_name）② 根据命中 ID 从图数据库加载节点+边+邻居（支持 k 跳）③ triplet_score = effective_distance(node1)+effective_distance(edge)+effective_distance(node2)④ 筛选最小的 k 个三元组 |
| **层级展开**            | L0→L1→L2 递归展开，收敛检测：top-k URI 集连续 N 轮不变则停止                                                                                                     | 无层级——扁平向量检索                                                                           | 无层级——只有 activated/archived 两级状态过滤                                           | 无层级——全量注入不需要                                                                          | 有动态生长的摘要树（TextSummary→GlobalContextSummary→Root），但检索不做逐层 drill-down，而是 Root+top-3 bucket+详细 triplet 一次性组合为上下文前缀                                                                                                                                                                                                                           |
| **注入方式**            | **语义 slot 注入**：5 个优先级 slot（identity/session/episodic/task/retrieved_evidence），注入到 additionalContext 或 system prompt 后缀                        | rerank 后的记忆作为 system prompt 或 tool injection 注入                                       | 明文事实注入 prompt，每条附带来源+时间戳+置信度。三种输出模式：matches/instruction/full_instruction    | **冻结快照注入 system prompt volatile tier**——MEMORY 和 USER PROFILE 两个语义角色块                 | 语义 slot、消息拼接、prompt 后缀均支持。有语义 slot 分区但**只分两个区**（session/graph），不分身份/会话/搜索三层                                                                                                                                                                                                                                                               |
| **角色注入**            | slot 分区但没有显式角色标注——identity_context 混合了 profile 和 preference                                                                                   | 无角色注入                                                                                 | tag/filter 纯手动（系统不自动生成 tag）                                                 | **显式语义角色标注**——MEMORY("your personal notes")→遵循行为，USER PROFILE("who the user is")→适配行为 | 无显式角色标注                                                                                                                                                                                                                                                                                                                                   |
| **预算控制**            | **TokenBudget 128K**：优先级层级分配，不足时低优先级降级（content→overview→abstract）                                                                             | 无显式 token 预算——依赖 rerank 后的 top-k 控制数量                                                 | 截断到模型上下文窗口限制——去重合并高度相似记忆+压缩到可用 token 预算                                     | 字符硬限制——MEMORY.md 上限 2,200 chars ≈ 300-400 tokens，USER.md 上限 1,375 chars ≈ 200 tokens  | **无 token budget**，只有字符级硬截断（4000 字符上限）；无优先级分配机制                                                                                                                                                                                                                                                                                           |
| **prefix cache 考量** | 不考虑——compose 每次返回内容可能不同                                                                                                                       | 不考虑                                                                                   | 不考虑——检索结果每轮可能不同                                                             | **核心设计考量**——冻结快照保证 system prompt 字节不变，保护 prefix cache                                 | 不考虑                                                                                                                                                                                                                                                                                                                                       |
| **意图感知**            | QueryPlanner 将 query 分类为 TypedQuery（memory/skill/resource）                                                                                    | 无意图感知                                                                                 | 无                                                                           | 无——全量注入不需要意图感知                                                                        | 有规则型意图感知：QueryRouter 用加权正则将 query 分类到 15 种 SearchType（如时间→TEMPORAL，推理→COT，精确→LEXICAL），但不区分"找偏好/找方法/找资料"语义意图；按记忆类型过滤通过 node_type/node_name 参数实现                                                                                                                                                                                            |
| **角色/场景过滤**         | slot 分区+TypedQuery 类型过滤                                                                                                                       | metadata filter 按 scope 过滤                                                            | tag 纯手动，dataset scope 限定                                                    | 全量注入——不需要过滤                                                                           | 支持 dataset_scope 限定检索范围（不同 dataset 存不同领域知识）；但没有"只检索与当前任务相关的记忆"的细粒度场景感知过滤                                                                                                                                                                                                                                                                  |

### 💡 可借鉴要点

1. **Hermes 的"策展即选择 + 全量注入 + prefix cache 保护"闭环**——完整的设计闭环：字符硬限制迫使写入时策展 → 已保存的都是高价值 → 全量注入不需要检索 → 全量注入保证 system prompt 稳定 → 稳定保护 prefix cache → 降低 token 成本。oG-Memory 可以考虑**对 identity_context slot（profile + 稳定偏好）采用 Hermes 式全量注入**——这些信息量很小且极其稳定，全量注入既保护 cache 又确保关键信息不遗漏。其余 slot 继续走检索流水线。

2. **memOS 的"fast/fine/mixed 三模式"**：不同场景对检索精度需求不同。oG-Memory 目前只有一种检索模式，可以考虑增加模式选择参数。

3. **Cognee 的 Triplet Search 检索范式**：Cognee 的检索不是简单的向量搜索，而是把知识图谱中的"三元组"（node1→edge→node2）作为检索单元，同时在 5 个向量集合中搜索，然后组合打分。这种图+向量混合检索在需要**关系推理**的场景下比纯向量检索更有效——比如"用户提到了 Alice，Alice 是华为的工程师"这样的关系型信息，纯向量检索可能只命中 Alice，而 Triplet Search 能命中 Alice→is_a→Engineer 和 Alice→works_at→Huawei 两个三元组，提供更完整的上下文。oG-Memory 的 HierarchicalSearcher 已经有类似理念（沿关系边展开），但只在层级内展开——可以考虑增加跨层级的关系边展开能力。

4. **oG-Memory 自身的优势：语义 slot 注入 + 优先级降级**。其他项目要么全量注入（Hermes）要么扁平拼接（mem0/memOS/Cognee），oG-Memory 的 5 个 slot + TokenBudget 优先级分配 + 降级策略是最精细的注入设计。**应保留并强化**。

---

## Q5：如何处理记忆的过期、冲突、合并、版本变化

| 维度       | **oG-Memory**                                                                                                                                                                                    | **mem0 2.0**                                           | **memOS 2.0**                                                                         | **Hermes-Agent**                                                                        | **Cognee**                                                                                                  |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------ | ------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| **写入策略** | **类型化策略**：profile 用 ProfilePolicy（固定 URI，替换式）；preference/entity/pattern 用 AggregateTopicPolicy（slug URI，追加式合并）；event/case 用 AppendOnlyPolicy（每次创建新节点）；skill/tool 用 SkillToolPolicy（固定 URI，累积式合并） | **无类型化策略**：LLM 判断新旧记忆关系后输出 add/update/delete/none 四种动作 | **状态三值**（activated/archived/deleted）+ **两套可选冲突策略**                                    | **字符硬限制驱动的策展压力**：空间满了必须删旧的，错误信息不只说"满了"还列出当前所有条目+容量百分比+新条目字符数                            | **remember() 是唯一入口**，写入有两种方案：Session Cache 或持久化到图谱。无类型化策略区分                                                 |
| **冲突检测** | **Safety Check**（提取阶段语义冲突预防）+ **乐观锁**（并发层面冲突控制）                                                                                                                                                  | **LLM 判断语义冲突**：新记忆写入时召回相似旧记忆，LLM 判断关系                  | **写入时自动语义冲突检测**（CONFLICT/DUPLICATE/COMPLEMENT/NEW 四种标签）                               | **无语义冲突检测**——内建记忆没有检索能力。**有并发写入保护**                                                     | **写入时预防冲突**：提取实体时通过模糊匹配将同一概念的不同写法映射为规范术语。**检索时低分知识排名靠后**（依据用户反馈），但矛盾的记忆是共存于图谱的                              |
| **冲突解决** | 乐观锁重试（最多 3 次）；类型化写入策略隐式解决语义冲突                                                                                                                                                                    | linked_memory_ids 关联新旧记忆——新旧记忆并存+链接                    | **两套可选策略**：latest_override（新胜旧，旧→archived）和 preserve_parallel（共存，标记 conflict_with）    | 漂移检测→拒绝修改+创建 `.bak.<timestamp>` 备份+返回修复指导                                               | 矛盾记忆共存于图谱，依赖用户反馈和频率权重过滤低质量节点排名靠后                                                                            |
| **过期机制** | **无 TTL**——依赖热度衰减（hotness_score 基于半衰期默认7天和访问计数衰减）影响排序权重，但不影响记忆生命周期                                                                                                                               | **无 TTL**——依赖 recency replacement 和 memory decay       | **状态三值 + 热度标记 + 衰减**：activated 正常召回 → archived 默认不召回 → deleted 永不召回（后台硬删除）。热度标记影响召回排序 | **无 TTL**——策展压力间接实现：空间满了必须删旧的                                                           | **没有过期机制**——知识图谱中的节点/边一旦写入即永久存在，除非被显式删除（`forget()`）                                                         |
| **碎片合并** | preference/entity 用 AggregateTopicPolicy（追加式合并，保留 provenance_ids）                                                                                                                                | **无后台合并机制**——只有写入时的 linked_memory_ids 关联               | **Dream 模块合并**：合并重复/相似记忆，旧记忆归档为 archived                                              | **去重**：加载时 dict.fromkeys() 保持顺序去重；写入时精确字符串匹配去重                                          | **实体描述整合**：周期性维护——获取实体及邻域→构建包含当前描述+所有关联边的 prompt→LLM 生成整合描述→写回图谱。**标签合并**：写入时做标签并运算（旧+新去重），删除时做标签剥离（只删目标标签） |
| **版本管理** | 扁平 version 数字 + provenance_ids 跨版本累积追踪来源                                                                                                                                                         | **无版本管理**——只有 linked_memory_ids 关联                     | **完整版本链**（latest_override 策略下）：旧 memory_id 通过 obsoleted_by 指向新记录                      | **"时间窗口"式版本管理**：磁盘文件=跨 session 持久版本；活跃状态=session 内 working copy；冻结快照=session start 发布版本 | **有记录版本但实际没作用**。节点创建后不可更改，更新实际上为删除再新增                                                                       |
| **并发安全** | 乐观锁 + Outbox 异步索引                                                                                                                                                                                | 无特殊并发控制                                                | 未提及                                                                                   | **最完整**：文件锁+重新读取+漂移检测+原子写入+威胁扫描（12种正则+10种不可见Unicode）                                    | 未提及                                                                                                         |

### 关键概念展开

**memOS 2.0 的写入时语义冲突检测**：新记忆写入前自动检索相似已有记忆，LLM 判断关系输出四种标签：

- **CONFLICT**：新旧互相矛盾
- **DUPLICATE**：同一件事的重复表述
- **COMPLEMENT**：互相补充
- **NEW**：无关

CONFLICT 时两套可选策略：
- **策略 A：latest_override**（默认）：旧→archived，新→activated，建立版本链
- **策略 B：preserve_parallel**：新旧均保留为 activated，标记 conflict_with，按时间+置信度排序

**Hermes-Agent 的并发写入保护**：文件锁+重新读取+漂移检测+原子写入+威胁扫描。姐妹会话并发写入和外部工具修改都有保护机制。

**Cognee 的冲突与合并机制**：

- **写入时预防冲突**：提取实体时通过模糊匹配将同一概念的不同写法映射为规范术语——比如"Python"和"python"会被映射为同一个 EntityType
- **检索时冲突处理**：矛盾的记忆共存于图谱，低分知识排名靠后（依据用户反馈）
- **实体描述整合**：周期性维护，LLM 生成整合描述写回图谱
- **标签合并**：同一实体被多个 dataset 共享时，写入时做标签并运算（旧+新去重），删除时做标签剥离（只删目标标签）——确保"归属关系"不被覆盖或误删

### 💡 可借鉴要点

1. **memOS 的"写入时语义冲突检测"**：oG-Memory 的冲突处理是写入层面的（乐观锁防并发），不是语义层面的。建议在 `ContextWriter.write_candidates()` 中增加写入前语义冲突检测步骤：对每个 CandidateMemory 检索已有相似节点，由 LLM 判断关系输出 CONFLICT/DUPLICATE/COMPLEMENT/NEW 标签，根据标签决定写入策略。

2. **memOS 的两套冲突策略可选**：latest_override（新胜旧）和 preserve_parallel（共存）可选。oG-Memory 目前只有类型化策略（profile 替换、preference 追加），没有"共存模式"——对于某些场景，preserve_parallel 保留两者让下游决策比强制覆盖更合理。

3. **Hermes 的威胁扫描**：12 种正则+10 种不可见 Unicode 检测覆盖 prompt injection / 角色劫持 / 秘密泄露 / SSH 后门等。oG-Memory 完全没有这个安全层——至少应该在用户手动写入时做威胁扫描。

4. **Hermes 的"策展压力间接实现过期"**：空间满了必须删旧的，关键是返回**策展上下文**（当前所有条目列表、容量百分比、新条目字符数），让 Agent 有足够信息做策展决策而非盲目删除。oG-Memory 可以借鉴"提供策展上下文而非二元拒绝"的理念。

5. **memOS 的"状态三值 + 容量上限"驱动记忆生命周期——oG-Memory 没有状态驱动召回也没有容量管理**：memOS 的 activated/archived/deleted 三值是**召回的硬开关**——recall.py 所有查询路径强制 `status="activated"`，archived/deleted 直接排除。配合（1）冲突归档→旧→archived；（2）容量淘汰→80%阈值时 DETACH DELETE 最旧的；（3）Dream 维护→长期未命中/低 usefulness/被反馈推翻→归档。oG-Memory 对比鲜明：hotness_score 衰减只影响排序权重不影响召回；archive_node() 和 ARCHIVE_CONTEXT 处理路径存在但无触发源；没有容量管理。可考虑三步渐进式改造：① 先加热度门槛（hotness<阈值直接跳过）；② 再激活自动归档触发源；③ 最后加 Dream 维护。

6. **Cognee 的标签并运算/标签剥离**：Cognee 在同一个知识实体被多个 dataset 共享时，写入时做标签并运算（旧+新去重），删除时做标签剥离（只删目标标签）——确保"归属关系"不被覆盖或误删。oG-Memory 的 AggregateTopicPolicy 做的是"追加式合并"，但没有"精确删除"能力——如果要删除某条偏好中的一部分内容，只能整体替换。可以考虑增加类似标签运算的精细删除能力。

---

## Q6：如何衡量记忆的好坏

| 维度        | **oG-Memory**                    | **mem0 2.0**                     | **memOS 2.0**                                                                                 | **Hermes-Agent** | **Cognee**                                                                                                                                                                    |
| --------- | -------------------------------- | -------------------------------- | --------------------------------------------------------------------------------------------- | ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **评估数据集** | LoCoMo（多轮对话）                     | LoCoMo                           | **4 个学术 + 1 个自建 benchmark**（LoCoMo / LongMemEval / LongBench v2 / PrefEval / PersonaMem）      | 未提及              | **5 个数据集**：HotPotQA（24题多跳推理，主要竞品对比基准）、TwoWikiMultiHop（双维基多跳）、**BEAM**（专为长期记忆评估设计，含10类探测问题，支持100K/500K/1M token）、LogisticsSystem（内部合成物流领域）、Musique（多跳推理QA）                     |
| **核心指标**  | 正确率提升+token消耗降低+提取质量+检索质量+组装质量   | BLEU+F1+LLM Judge（CORRECT/WRONG） | **三类指标**：词法（F1/ROUGE/BLEU/METEOR）+语义（BERT F1/Similarity）+LLM Judge                            | 未提及              | **6 类指标**：Correctness（DeepEval GEval，LLM-as-judge）、Exact Match（二值1.0/0.0）、F1（Token-level）、Contextual Relevancy（检索上下文相关性）、Context Coverage（覆盖黄金上下文比例）、Rubric（LLM 逐条评判，BEAM 专用） |
| **对比基线**  | 无记忆 baseline（数据待填）               | LoCoMo 上的多种 baseline             | **7 个对比基线**（mem0/mem0-graph/memobase/memu/supermemory/zep/RAG）                                | 未提及              | HotPotQA 24题上的竞品对比（结果在 `evals/benchmark_summary_competition.json`）                                                                                                            |
| **评估脚本**  | 公开：tests/e2e/ + tests/benchmark/ | 公开：evaluation/                   | **公开且可一键运行**（run_*.sh 五步流水线）                                                                  | 未提及              | 公开（evals/src/qa/ 下有4个系统的完整 benchmark 类），但**不能一键跑出结果**——需分别配置 Modal 云环境、Neo4j/各系统 API 等基础设施                                                                                    |
| **细分评估**  | **三维度分别评估**：提取/检索/组装             | 不细分——端到端评估                       | 按 benchmark 类型细分（LoCoMo 4类/LongMemEval 7场景/PrefEval 4偏好维度/PersonaMem 按主题/LongBench 按难度+长度+领域） | 未提及              | 按 Correctness/EM/F1/Relevancy/Coverage/Rubric 分别计算，但**没有记忆增长曲线、没有副作用评估、没有消融实验**                                                                                               |

### 💡 可借鉴要点

1. **Cognee 的 BEAM benchmark**——这是**专为长期记忆评估设计**的 benchmark，含 10 类探测问题，支持 100K/500K/1M token 长度。oG-Memory 目前只在 LoCoMo 上评估——可以考虑扩展到 BEAM（长期记忆是 oG-Memory 的核心场景）。

2. **memOS 的多 benchmark 评估体系**——4 个学术 benchmark + 1 个自建，覆盖不同能力维度。oG-Memory 可以扩展到 LongMemEval（跨会话记忆）和 PrefEval（偏好理解使用）。

3. **memOS 的 PrefEval 偏好维度评估**——4 个偏好维度（违反/承认/幻觉/有用）不是简单的"回答正确率"，而是偏好层面的细粒度评估。oG-Memory 的 preference 类型记忆目前没有专门的评估维度。

4. **细分评估是 oG-Memory 的优势**：提取/检索/组装三维度分别评估可以定位问题在哪个环节。memOS 和 Cognee 的评估是端到端的。**应保持并强化**。

---

## 优先级排序：oG-Memory 最应借鉴的改进

| 优先级 | 改进项 | 来源 | 预期收益 | 实现难度 |
|--------|--------|------|---------|---------|
| **P0** | **Dream/离线反思模块** | memOS | 消除碎片、提炼技能、生成洞察——长期记忆质量的关键瓶颈 | 高（新模块） |
| **P1** | **写入时语义冲突检测** | memOS | 防止矛盾记忆共存、自动识别 DUPLICATE/COMPLEMENT | 中 |
| **P1** | **identity slot 全量注入 + 冻结快照** | Hermes | 保护 prefix cache、确保关键信息不遗漏、降低检索成本 | 低 |
| **P2** | **记忆语义角色标注** | Hermes | 引导模型对两类信息产生不同行为倾向 | 低 |
| **P2** | **Nudge 智能触发** | Hermes | 降低提取成本、避免提取噪音 | 中 |
| **P2** | **反馈与修正类 YAML schema** | memOS | 用户纠正场景的显式标记和来源追踪 | 中 |
| **P2** | **SkillRun + SkillImprovementProposal 闭环** | Cognee | 技能执行的显式反馈闭环，从"最佳实践"到"执行→记录→改进" | 中 |
| **P2** | **sub-agent 深层检索机制** | Cognee | 深层/跨会话检索 offload 到子 agent，不阻塞主对话流、不消耗主 token budget | 中（Claude Code 插件增加 og-deep-search Skill） |
| **P2** | **LLM Judge 语义正确性评估** | mem0 | 评估语义层面的回答正确性 | 低 |
| **P3** | **Plugin Hook 点** | memOS | 开放第三方干预核心流程的能力 | 中 |
| **P3** | **preserve_parallel 冲突策略** | memOS | 某些场景保留矛盾记忆共存比强制覆盖更合理 | 低 |
| **P3** | **检索模式选择** | memOS | 不同场景不同精度需求 | 低 |
| **P3** | **标签运算式精细删除** | Cognee | AggregateTopicPolicy 增加精确删除部分内容的能力 | 中 |
| **P3** | **BEAM benchmark 评估** | Cognee | 专为长期记忆设计的评估数据集 | 低 |
| **P3** | **ParametricMemory 层** | memOS | "技能式"记忆——内化后无需检索的直觉能力 | 高 |

---

## 相关链接

- [[agentic-memory-design-comparison]] — V1 版本（四项目对比，不含 Cognee）
- [[ogmemory-deep-analysis]] — oG-Memory 六维度深度分析
- [[ogmemory]] — oG-Memory 项目概述