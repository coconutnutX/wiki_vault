---
name: ogmemory-deep-analysis
description: oG-Memory (ContextEngine) 六维度深度分析：北向对接、记忆模型、记忆产生、检索注入、生命周期、效果衡量——作为 agentic memory 项目调研比较的基准示例
tags: [ogmemory, agentic-memory, comparison, analysis]
created: 2026-05-27
updated: 2026-05-27
---

# Agentic Memory 项目调研比较

随着 Agent 应用从单次对话走向长期协作，**记忆系统**正在成为决定 Agent 智能水平的关键基础设施——它决定了 Agent 能否跨会话积累知识、能否在推理前拿到正确的上下文、能否随用户变化自我更新。当前 agentic memory 领域尚无统一范式，各项目在设计取舍上差异显著：有的把记忆内嵌在 Agent 中，有的独立为外部服务；有的只存用户偏好，有的同时追踪 Agent 自身技能；有的纯向量检索，有的向图混合；有的只做对话后提取，有的还支持离线反思优化。

我们团队正在开发 **oG-Memory（ContextEngine）**，已有初步的设计和实现。为了深化对记忆系统设计的理解，我们需要对当前主流 agentic memory 项目进行系统调研：**比较不同项目的设计思路，借鉴可取之处，识别 oG-Memory 的不足与改进方向，最终优化 oG-Memory 的设计**。本调研不是简单的功能清单对比，而是聚焦于每个设计决策背后的"为什么"——为什么这样建模记忆、为什么这样触发提取、为什么这样检索注入——因为这些取舍才是跨项目可借鉴的真正内核。

本文档包含两部分：
- **第 0 章**：提出六个核心问题及其详细描述、调研侧重点（设计思路优先）和追问方向，供调研人员快速搞清楚需要调查什么、怎么看。
- **第 1–6 章**：以 oG-Memory 为例给出具体答案，作为其他同事梳理对应项目的**基准示例**——照此格式填充即可。

> **关于调研粒度的说明**：第 0 章列出的侧重点和追问方向是**可选参考**，不是逐一回答的检查清单。每个记忆体项目都有自己的设计重心和亮点——调研时应**抓住该项目最独特、最有启发性的设计决策**来回答六个核心问题，从设计思路上整体阐述，而非逐条回应每个细粒度小点。换句话说：六个核心问题必须回答，但回答的方式是从整体设计脉络出发，挑重点讲清楚"为什么这样做"，而不是零碎罗列"是否支持 X、是否支持 Y"。oG-Memory 示例（第 1–6 章）的写法就是这种风格——围绕核心设计逻辑组织内容，细节和代码引用服务于思路说明，而非独立成段。

---

## 0. 调研框架：六个核心问题

对每个 agentic memory 项目，我们需要回答以下六个问题。每个问题给出了**详细描述**、**调研侧重点（设计思路优先）**，以及**值得追问的方向**。调研时应重点关注"为什么这样设计"而非"代码怎么写"——后者可以从代码中直接读到，前者才是跨项目比较的真正价值。

### Q1：如何对接不同的 Agent

**问题描述**：记忆系统不是孤立运行的，它必须和 Agent 产生交互——Agent 需要在推理前拿到记忆（读），也需要在对话后把新信息送入记忆（写）。这个"接口"怎么设计？是记忆内嵌在 Agent 里（in-process），还是独立服务外挂（out-of-process）？Agent 和记忆之间的数据流走向如何？

**调研侧重点**：
- **架构拓扑**：记忆是 Agent 的一部分还是独立进程/服务？这决定了部署复杂度和多 Agent 共享的可行性
- **拦截机制**：Agent 的哪些生命周期节点被拦截？是在消息输入时、推理前、工具调用后、轮次结束时、还是压缩/退出时？拦截点越多越精细，但侵入性也越大
- **数据流方向**：读侧（记忆注入 Agent）和写侧（Agent 输出沉淀到记忆）是否对称？是否还有"旁路"数据（如工具 I/O 直接入记忆，不经 Agent 主循环）
- **耦合程度**：Agent 是否需要知道记忆的内部结构（如记忆类型、URI）？还是只看到一个扁平的 text block？前者灵活但耦合重，后者简单但可控性差

**追问方向**：
- 是否支持多 Agent 共享同一记忆实例？共享时的可见性/隔离策略？
- 是否支持同步和异步两种写入模式？（实时 vs 批量）
- 接口协议：HTTP REST / gRPC / SDK callback / 共享文件 / 其他？

### Q2：记忆对象模型如何构建

**问题描述**：把什么当成"记忆"？是用户画像、偏好、任务、事件、实体、技能、工具经验、代码轨迹……还是更抽象的东西？记忆粒度如何——一条一条偏好 vs 一个主题下的所有偏好？记忆有无层级结构？

**调研侧重点**：
- **记忆分类体系**：项目定义了哪些记忆类型？类型之间有无边界约束（如"偏好只限用户自己说的，他人偏好归实体"）？这是区分项目设计哲学的关键
- **粒度与结构**：记忆粒度是原子级（一条事实）还是主题级（一个话题下的所有信息）？是否有层级（如 profile → preference → 具体偏好）或图结构（关系边）？
- **Scope 区分**：是否区分 user-scope（用户画像）和 agent-scope（Agent 学到的技能/案例）？这影响记忆归属和多 Agent 场景下的共享
- **Schema 定义方式**：记忆结构是硬编码还是动态定义（YAML/JSON schema）？后者允许无代码扩展新类型
- **索引分层**：记忆是否有多个摘要层级（如 abstract / overview / content）用于不同检索精度？

**追问方向**：
- 是否有"反思型"记忆（Agent 对自身行为的评价）？
- 是否有"目标型"记忆（当前任务、计划、待办）？
- 记忆节点之间是否有语义关系（related_to / contradicts / derived_from）还是纯扁平？
- 记忆是否带时间戳和来源溯源（provenance）？

### Q3：记忆怎么产生

**问题描述**：记忆从哪里来？用户手动写入？对话后自动抽取？工具结果沉淀？任务结束后总结？后台整理合并？离线 dreaming 优化？不同的产生方式决定了记忆的丰富度、及时性和成本。

**调研侧重点**：
- **自动化程度**：记忆产生是完全自动（LLM 提取）还是需要用户干预？自动提取的触发时机是什么（turn end / session end / 实时）？
- **提取方法论**：如果是 LLM 提取，是单次调用还是多轮迭代（ReAct loop）？是否有 span 划分（先找可提取片段再结构化）？是否有去重/去噪机制？
- **多来源整合**：是否整合了工具 I/O、代码轨迹、搜索结果等非对话来源？还是只从 user/assistant 消息中提取？
- **离线处理**：是否有后台整理（合并碎片、消除矛盾、生成摘要）或 dreaming（离线反思优化）？这是长期记忆质量的关键但多数项目缺失

**追问方向**：
- 提取是否支持增量（只处理新消息）还是全量重跑？
- 提取结果的置信度如何评估和过滤？
- 是否有"双重运行"策略（同一段对话跑两次提取取交集）来提升鲁棒性？
- 用户能否手动写入/修改/删除记忆？

### Q4：推理前如何选择正确记忆并注入上下文

**问题描述**：Agent 推理前，记忆系统要从海量历史中选出"对的"记忆，并以正确格式注入到 Agent 的上下文中。检索策略是什么（纯向量 / 向量+图 / 向量+关键词混合 / 直接读文件）？注入方式是什么（system prompt 后缀 / 语义 slot / 消息拼接）？如何控制注入量（token budget）？

**调研侧重点**：
- **检索策略**：是纯向量检索、向量+BM25 混合、还是图遍历？是否有多阶段检索（先粗筛后精排）？是否支持层级展开（从摘要找到全文）？
- **意图感知**：检索是否考虑查询意图（"我要找偏好"vs"我要找怎么做事"vs"我要找资料"）？是否按记忆类型过滤？
- **注入方式**：记忆注入到 Agent 的什么位置——system prompt、user message 后缀、还是独立消息？是否按语义 slot 分区注入（身份层 / 会话层 / 搜索层）？
- **预算控制**：是否有 token budget 机制？预算分配是否按优先级（身份信息不可删减，搜索结果可降级）？降级策略是什么（content → overview → abstract）？

**追问方向**：
- 是否支持推理过程中动态补充检索（不是只在 turn 开始时）？
- 是否考虑记忆时效性（新记忆优先 / 热度衰减）？
- 是否支持角色/场景过滤（"只检索与当前任务相关的记忆"）？
- 检索结果是否带关系扩展（找到一条记忆后沿关系边扩展相关记忆）？

### Q5：如何处理记忆的过期、冲突、合并、版本变化

**问题描述**：记忆不是静态的——用户偏好会变，事实会过期，同一主题可能有多条碎片需要合并，并发写入会产生冲突。项目如何处理这些动态性？

**调研侧重点**：
- **写入策略**：不同类型的记忆是否有不同的写入策略（替换 / 追加 / 累积 / 只增不改）？这是设计哲学的核心：profile 该替换还是累积？event 该追加还是覆盖？
- **冲突检测与解决**：并发写入时是否有乐观锁 / CAS / 版本号机制？新旧信息矛盾时如何处理——新胜旧？保留两者？标记为矛盾？
- **过期机制**：是否有显式 TTL / 过期时间？还是依赖热度衰减（访问越多越活，越久不用越沉）？是否有归档/软删除而非物理删除？
- **碎片合并**：是否有后台定期合并机制（将多条碎片偏好合并为一条综述）？还是只在写入时增量合并？

**追问方向**：
- 是否有"矛盾标记"机制（主动检测并标记互相矛盾的记忆）？
- 是否支持记忆版本历史（回溯到旧版本）还是只有当前版本？
- 归档/删除是由系统自动触发还是需要用户/Agent 决策？
- 是否有数据一致性保障（如 Outbox pattern 保证写入和索引同步）？

### Q6：如何衡量记忆的好坏

**问题描述**：记忆系统是否真的有帮助？用什么指标衡量？是提升问答正确率、任务成功率、减少 token 消耗、还是用户满意度？评估用的数据集是什么？

**调研侧重点**：
- **评估数据集**：是否在标准数据集上评估（LoCoMo / LongMemEval / 其他）？是否自建数据集？数据集覆盖哪些场景（单轮问答 / 多轮对话 / 长期遗忘 / 跨会话）？
- **核心指标**：正确率 / recall / precision / F1？token 消耗对比（有记忆 vs 无记忆）？任务完成率？记忆命中率（多少对话被记忆增强）？
- **对比基线**：和什么 baseline 对比？无记忆？简单 RAG？其他记忆系统？
- **可复现性**：评估脚本是否公开？能否一键跑出结果？

**追问方向**：
- 是否有"记忆增长曲线"评估（随对话轮次增加，记忆数量和存储大小如何增长）？
- 是否有 A/B 测试或用户满意度数据？
- 是否评估了记忆系统的副作用（如注入错误记忆导致的性能下降）？
- 评估是否区分了不同记忆类型的贡献（profile vs preference vs event 分别提升多少）？

---

## 1. 如何对接不同的 Agent

oG-Memory 的设计是**记忆服务独立于 Agent 运行**，核心逻辑和 Agent 对接代码完全分离——Agent 侧对接代码是独立文件夹（`claude-plugin/`、`openclaw_context_engine_plugin/`），通过中间脚本调用 HTTP API，不依赖项目核心 Python 包。

### 整体架构

```
Agent (Claude Code / OpenClaw)
  │
  │  hook（自动触发）或 skill（Agent 自主调用）
  v
中间脚本（独立文件夹，纯 stdlib bash/python/node）
  │
  │  HTTP POST/GET → /api/v1/compose, after_turn ...
  v
oG-Memory 服务 (Flask, :8090)
  │
  v
MemoryService → Extraction / Retrieval / Commit
```

### 插件分离设计

oG-Memory 的 Agent 对接代码与核心业务逻辑物理分离：

| 插件文件夹                             | 语言            | 入口                                                                                                                          | 中间脚本/模块                                                                                                                                                                                                                                                                                                                                   | HTTP 调用方式                             |
| --------------------------------- | ------------- | --------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| `claude-plugin/`                  | Python + Bash | [hooks/hooks.json](https://gitcode.com/opengauss/oGMemory/blob/dev/claude-plugin/hooks/hooks.json)                          | [scripts/](https://gitcode.com/opengauss/oGMemory/blob/dev/claude-plugin/scripts/) 下 4 个脚本：`call_compose.py`、`call_after_turn.py`、`call_add_session_message.py`、`stop_detach.sh`，共享 [ogm_plugin_request.py](https://gitcode.com/opengauss/oGMemory/blob/dev/claude-plugin/scripts/ogm_plugin_request.py) 封装 base_url / headers / identity | Python stdlib `urllib.request`，无第三方依赖 |
| `openclaw_context_engine_plugin/` | Node.js       | [openclaw.plugin.json](https://gitcode.com/opengauss/oGMemory/blob/dev/openclaw_context_engine_plugin/openclaw.plugin.json) | [index.js](https://gitcode.com/opengauss/oGMemory/blob/dev/openclaw_context_engine_plugin/index.js) 统一处理所有 lifecycle method                                                                                                                                                                                                               | Node `fetch` / `http`                 |

中间脚本的设计原则：
- **纯 stdlib，无项目依赖**：Claude Code 脚本只用 `urllib.request`（Python stdlib），不 import oG-Memory 核心包；OpenClaw 插件是纯 Node.js。这样 hook 脚本可以在任意环境跑，不要求 oG-Memory 包在 sys.path 上
- **身份与配置统一**：[ogm_plugin_request.py](https://gitcode.com/opengauss/oGMemory/blob/dev/claude-plugin/scripts/ogm_plugin_request.py) 封装了 `OG_MEMORY_URL`、`OG_AUTH_API_KEY`、account/user/agent ID 的解析——先尝试加载 `OgMemConfig`（monorepo 在 sys.path 时），否则回退环境变量

### Hook（自动触发）vs Skill（Agent 自主决定）

oG-Memory 的 Claude Code 插件同时注册了 **hook** 和 **skill**，两者的触发方式截然不同：

**Hook — 一定会跑**：[hooks/hooks.json](https://gitcode.com/opengauss/oGMemory/blob/dev/claude-plugin/hooks/hooks.json) 定义了 4 个 hook，Claude Code 在对应生命周期事件时**自动调用**中间脚本，不需要 Agent 主动触发：

| Hook               | 脚本                            | 调用接口                         | 说明                                       |      |           |      |                                  |
| ------------------ | ----------------------------- | ---------------------------- | ---------------------------------------- | ---- | --------- | ---- | -------------------------------- |
| `PostToolUse`      | `call_add_session_message.py` | `/sessions/{id}/messages`    | 仅**有副作用**的工具（matcher: `Write             | Edit | MultiEdit | Bash | NotebookEdit`）写入，避免 Read/Glob 刷屏 |
| `UserPromptSubmit` | `call_compose.py`             | `/api/v1/compose`            | 用户每次提问时**自动**检索记忆，注入 `additionalContext` |      |           |      |                                  |
| `Stop`             | `stop_detach.sh`              | `/api/v1/after_turn` (async) | 对话结束**自动**增量同步 transcript                |      |           |      |                                  |
| `PreCompact`       | `call_after_turn.py`          | `/api/v1/after_turn`         | 压缩前**自动**同步防丢失                           |      |           |      |                                  |

**Skill — Agent 自主决定是否用**：[plugin.json](https://gitcode.com/opengauss/oGMemory/blob/dev/claude-plugin/.claude-plugin/plugin.json) 的 `skills` 字段注册了两个 skill，Agent 在对话中根据需要**自主调用**（用户也可以通过 `/og-compose`、`/og-add-history` 显式触发）：

| Skill | 说明 |
|-------|------|
| `/og-compose <关键词>` | Agent 觉得需要检索记忆时主动用；内联 bash curl 调用 `/api/v1/compose`，返回完整记忆上下文（profile + archives + session + working set） |
| `/og-add-history` | Agent 或用户决定导入本仓库历史对话；有成本提示与 `.ingest-offset` 去重 |

这种 hook + skill 双轨设计的好处：hook 保证核心读写流程**不遗漏**（每次提问自动检索、每次结束自动写入），skill 提供**按需深挖**能力（Agent 可以主动搜更多上下文、主动导入历史）。

### OpenClaw 插件

[openclaw_context_engine_plugin/README.md](https://gitcode.com/opengauss/oGMemory/blob/dev/openclaw_context_engine_plugin/README.md) 通过 OpenClaw 的 `contextEngine` slot 接入，映射 OpenClaw lifecycle method 到 oG-Memory API。与 Claude Code 插件不同，OpenClaw 没有独立的中间脚本文件——所有逻辑在 [index.js](https://gitcode.com/opengauss/oGMemory/blob/dev/openclaw_context_engine_plugin/index.js) 中统一处理，因为它只需要实现 OpenClaw 插件接口的 4 个方法（assemble / afterTurn / compact / dispose），每个方法内部直接 HTTP 调用 oG-Memory API：

| OpenClaw 方法 | oG-Memory 调用 | 行为 |
|---|---|---|
| `assemble()` | `/prefetch` → `/compose` | 分层返回 profile/archive/session/working set |
| `afterTurn()` | `/after_turn` | 增量提取并持久化 |
| `compact()` | `/prepare_compaction` → `/compact` | 压缩接管 |
| `dispose()` | `/dispose` | 释放资源 |

### 关键设计决策

- **Agent 不知道记忆内部结构**：Agent 只看到 compose 返回的 `systemPromptAddition` / `additionalContext`，不直接操作记忆存储
- **双向拦截**：读侧（compose）在推理前注入；写侧（after_turn）在 turn 结束后抽取
- **多租户隔离**：所有 API 要求 `RequestContext`（account_id + owner_space），[core/models.py:56-82](https://gitcode.com/opengauss/oGMemory/blob/dev/core/models.py#L56-L82) 的 `RequestContext` 强制携带 tenant 信息

### 调研方向

- 其他项目是否直接将记忆嵌入 Agent 内部（而非独立服务）？
- 是否支持多 Agent 共享记忆？Agent 间记忆可见性策略？
- 除 HTTP 外是否支持 gRPC / WebSocket / in-process 调用？
- 是否支持 Agent 自定义拦截点（如 TOOL_RESULT、TASK_END）？

---

## 2. 记忆对象模型如何构建

oG-Memory 的记忆存储设计思路是**以 URI 为核心寻址，以 ContextNode 为统一存储单元**。URI 是记忆的身份标识（如 `ctx://acme/users/alice/memories/preferences/coding_style`），ContextNode 是 URI 对应的完整数据结构。上层所有操作（读写、检索、合并）都通过 URI 寻址，不关心底层物理存储。

### URI-centric 架构与存储演进

oG-Memory 的核心接口 [ContextFS](https://gitcode.com/opengauss/oGMemory/blob/dev/core/interfaces.py) 定义了 `read_node(uri)`、`write_node(node)`、`exists(uri)`、`list_children(uri)` 等方法——**操作单位始终是 URI，而非文件路径或数据库行**。这让底层存储可以替换而不影响上层逻辑。

目前有两套存储实现，处于**从文件系统（AGFS）向数据库（DBFirst）迁移**的阶段：

| 适配器             | 文件夹                                                                                  | 存储方式                                                                                                                     | 状态                    | 说明                                                                                                                           |
| --------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ | --------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `AGFSContextFS` | [fs/agfs_adapter/](https://gitcode.com/opengauss/oGMemory/blob/dev/fs/agfs_adapter/) | AGFS（Go 实现的分布式文件系统），每个 URI 对应一个目录，含 `content.md`、`.abstract.md`、`.overview.md`、`.meta.json`、`.relations.json`            | **旧方案**，逐步退出          | 文件模型直观、可人工检查，但原子写入需要 4 步（写 content → abstract → overview → meta），并发控制靠乐观锁+版本号                                                |
| `SQLContextFS`  | [fs/sql_adapter/](https://gitcode.com/opengauss/oGMemory/blob/dev/fs/sql_adapter/)   | PostgreSQL / openGauss 单表 `context_nodes`，一条行 = 一个 URI 的全部字段（content / abstract / overview / relations / metadata JSONB） | **新方案（DBFirst）**，正在迁移 | 一条 `INSERT ON CONFLICT` 替代 AGFS 的 4 步原子写入，事务天然保证一致性；RLS（Row-Level Security）实现多租户隔离；outbox_events 也从 `.outbox/*.json` 迁移到数据库表 |

DBFirst 迁移的核心动机：AGFS 的文件模型在并发写入、事务一致性、运维复杂度上有天然劣势（4 步写入无法在一个事务内完成，需要乐观锁重试）；数据库模型用一条 SQL 替代，PostgreSQL/openGauss 事务直接保证原子性。同时 [db_schema.sql](https://gitcode.com/opengauss/oGMemory/blob/dev/fs/sql_adapter/db_schema.sql) 还定义了 `relation_edges`（关系边独立表）、`session_archives`（会话归档）、`outbox_events`（异步索引事件）、`dream_recalls`（梦境召回追踪）等表，以及 RLS 策略保证租户隔离。

### 七类记忆模型

| 类别             | 语义             | owner_scope | URI 示例                                                                 | operation_mode | 提取方式（LLM function call 工具）                                                                                      | 提取提示词差异                                                                                                                         |
| -------------- | -------------- | ----------- | ---------------------------------------------------------------------- | -------------- | ------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| **profile**    | 用户属性：姓名、职业、背景  | user        | `ctx://acme/users/alice/memories/profile/name`                         | upsert（替换式合并）  | `extract_profile`：**逐属性调用**（每次提取一个属性，如 name、location）                                                          | **必须提供** `evidence_quote`、`attributed_speaker`、`attribution_basis`（enum: self_first_person / self_named / other_named），用于区分"用户自己说"和"别人提到" |
| **preference** | 用户偏好：咖啡口味、编码风格 | user        | `ctx://acme/users/alice/memories/preferences/coding_style`             | upsert         | `extract_preference`：**逐主题调用**（每次提取一个偏好主题）                                                               | **仅限用户自己的偏好**（"ONLY for preferences stated BY the user ABOUT the user"），他人偏好 → entity；overview 须包含 topic + 偏好 + evidence                 |
| **entity**     | 实体：人、地点、组织     | user        | `ctx://acme/users/alice/memories/entities/alice`                       | upsert         | `extract_entity`：**逐实体调用**（每次提取一个实体）                                                                   | overview 须包含 entity type（person/place/org/object）、关键属性、特征、关系                                                                   |
| **event**      | 时间事件：会议、旅行     | user        | `ctx://acme/users/alice/memories/events/meeting_with_team_20260527_x8` | add_only（追加）   | `extract_event`：**逐事件调用**                                                                                      | overview 须包含 what happened、绝对日期、参与者、情感语境；routing_key 加 timestamp_uuid 后缀保证唯一                                          |
| **case**       | 问题解决案例         | agent       | `ctx://acme/agents/ogmem/memories/cases/debug_api_timeout_20260527_x8` | add_only       | `extract_case`：**逐案例调用**                                                                                       | **必须包含三要素**：问题描述、解决步骤、最终结果；routing_key 加 timestamp_uuid 后缀                                                |
| **pattern**    | 行为模式观察         | user        | `ctx://acme/users/alice/memories/patterns/friday_followup`             | upsert         | `extract_pattern`：**逐模式调用**                                                                                    | overview 须包含 trigger condition、observation、frequency/context；**明确排除**完整可复用工作流（→ skill）                                |
| **skill**      | 可复用技能/工作流      | agent       | `ctx://acme/agents/ogmem/skills/debug_protocol`                        | upsert（累积式合并）  | `extract_skill`：**逐技能调用**                                                                                      | **必须满足三条件**：clear trigger condition、specific step sequence、verifiable completion criteria                                 |
| **tool**       | 工具使用洞察         | agent       | `ctx://acme/agents/ogmem/memories/tools/bash`                          | upsert（累积式合并）  | `extract_tool`：**逐工具调用**                                                                                       | 独有字段：`best_for`、`optimal_params`、`common_failures`、`recommendation`；提取后存入 `tool_stats` dict                                |

> **机制说明**：七类记忆共享同一个 Phase 1 span identification 步骤（找出哪些对话片段值得提取），但在 Phase 2 时 LLM 通过不同的 `extract_*` 工具调用分别提取。每个工具的 `description`（即 YAML 中的 `description` 字段）就是给 LLM 的提示词核心——它规定了什么该提取、什么不该提取、overview 必须包含什么结构。工具的 `input_schema`（即 YAML 中的 `fields`）规定了输出格式。`SchemaRegistry` + `tool_builder` 将 YAML 动态转换为 OpenAI function calling 工具定义，因此**增加新记忆类型只需添加 YAML 文件，无需改代码**。

### Schema 定义机制

每种记忆类型由 [extraction/schemas/definitions/](https://gitcode.com/opengauss/oGMemory/blob/dev/extraction/schemas/definitions/) 下的 YAML 文件定义。例如 [profile.yaml](https://gitcode.com/opengauss/oGMemory/blob/dev/extraction/schemas/definitions/profile.yaml) 定义了字段 `routing_key`、`abstract`、`overview`、`content`、`confidence`、`evidence_quote`、`attributed_speaker` 等。

YAML schema 同时驱动：
1. **LLM 提取工具生成**：`SchemaRegistry` + `tool_builder` 将 YAML 转为 `extract_profile`、`extract_preference` 等工具定义，供 LLM function calling
2. **URI 解析**：`directory` + `filename_template` 决定记忆在存储中的路径
3. **写入策略选择**：`operation_mode` 决定 upsert / add_only，映射到 `MergePolicy`

### 记忆节点结构

[core/models.py:86-103](https://gitcode.com/opengauss/oGMemory/blob/dev/core/models.py#L86-L103) 的 `ContextNode` 是统一的持久化单元，包含 `uri`、`abstract`、`overview`、`content`、`metadata`、`relations` 等字段。底层存储映射不同：

- **AGFS 旧方案**：每个 URI 对应一个目录，拆为 5 个文件（`content.md`、`.abstract.md`、`.overview.md`、`.meta.json`、`.relations.json`）
- **DBFirst 新方案**：每个 URI 对应 `context_nodes` 表的一行，所有字段（content / abstract / overview / relations / metadata）为同一行的不同列（relations 和 metadata 用 JSONB），一条 SQL 即完成原子读写

### 三级索引

每个 ContextNode 对应 3 个向量索引记录（L0/L1/L2），[core/models.py:194-217](https://gitcode.com/opengauss/oGMemory/blob/dev/core/models.py#L194-L217) 的 `IndexRecord`：
- **L0 (abstract)** — ~100 tokens，作为路标式嵌入
- **L1 (overview)** — ~500 tokens，平衡信号
- **L2 (content)** — ~5000 tokens，全面细节

### 关系边

[core/models.py:131-137](https://gitcode.com/opengauss/oGMemory/blob/dev/core/models.py#L131-L137) 的 `RelationEdge` 支持有向关系：`related_to`、`derived_from`、`contradicts`、`SEQUENCE`，带权重 0.0-1.0。

### 调研方向

- 其他项目是否区分 user-scope 和 agent-scope 记忆？
- 是否有更多记忆类型（如 emotion、goal、plan、reflection）？
- 记忆粒度：oG-Memory 以 topic 为粒度（一个 preference 节点 = 一个主题）；其他项目是否更细（一条一条偏好）或更粗？
- 是否有层级/图结构记忆？oG-Memory 有 L0/L1/L2 但本质是扁平节点 + 关系边

---

## 3. 记忆怎么产生

oG-Memory 的记忆产生完全依赖**对话后自动抽取**，不支持用户手动写入或离线优化。

### 两阶段提取（Eager 模式）

[extraction/tools.py](https://gitcode.com/opengauss/oGMemory/blob/dev/extraction/tools.py) 的 `Extractor` 实现了两阶段提取：

**Phase 1 — Span Identification**（轻量 JSON 模式）：
- 用便宜的 LLM 调用识别哪些消息范围包含可提取信息
- 输出 `[{"start": N, "end": M, "reason": "...", "categories": ["profile", "preference"]}]`
- 已提取的内容通过 `session_summary_block` 传入，避免重复提取

**Phase 2 — Span Structuring**（工具调用模式）：
- 对每个 span，用 LLM function calling 调用 `extract_profile` / `extract_preference` 等工具
- 每个提取结果是一个 `CandidateMemory`（带 confidence、evidence_quote、attributed_speaker）
- **双重运行（dual-run）**：每个 span 跑两次（正常温度 + temperature=0），取置信度高/内容丰富的合并
- confidence < 0.5 的候选直接丢弃

### ReAct 循环提取（Lazy 模式）

[extraction/extraction_react_loop.py](https://gitcode.com/opengauss/oGMemory/blob/dev/extraction/extraction_react_loop.py) 的 `ExtractionReActLoop` 实现了迭代式提取：

- LLM 可在提取过程中调用 `read`、`list`、`get_relations`、`get_access_stats` 工具读取已有记忆
- **Safety Check**：如果 LLM 决定写入一个未读过且已存在的记忆节点，自动重读该节点（防止覆盖式冲突）
- 最大迭代 3 次，超时 30 秒

### 触发时机

记忆提取在以下时刻触发：

| 时机 | 触发方式 | 说明 |
|------|---------|------|
| **Turn End (after_turn)** | Agent hook 调用 API | 对话轮次结束，增量提取 |
| **Pre-Compact** | Agent hook 调用 API | 压缩前提取未落盘内容 |
| **Session Commit** | `/sessions/{id}/commit` | 会话结束时归档+提取 |
| **Bootstrap** | `/api/v1/bootstrap` | 初始化用户空间 |

### 工具结果沉淀

Claude Code 的 `PostToolUse` hook 仅将有副作用工具（Write/Edit/Bash）的 I/O 作为 `role: tool` 补录到 session，与 `after_turn` 的 transcript 互补。这些 tool 消息也会被纳入提取范围——tool 类型记忆有专门的 schema（[tool.yaml](https://gitcode.com/opengauss/oGMemory/blob/dev/extraction/schemas/definitions/tool.yaml)），包含 `best_for`、`optimal_params`、`common_failures` 等字段。

### 提取流水线完整流程

```
after_turn → SessionManager.commit()
  → 消息缓冲区 → Extractor.extract()
    → Phase 1: 识别 spans
    → Phase 2: 结构化 spans → CandidateMemory[]
  → ContextWriter.write_candidates()
    → PolicyRouter.plan() → WritePlan
    → ArchiveBuilder.build() → ContextNode
    → ContextFS.write_node() → AGFS
    → OutboxStore.register_write() → 异步索引更新
```

### 调研方向

- 其他项目是否支持**用户手动写入**记忆？（如 Claude Code 的 MEMORY.md）
- 是否有**离线优化/Dreaming**机制？（如 Letta 的 archival memory re-processing）
- 是否有**任务结束后总结**？（oG-Memory 的 session_archive 是压缩产物而非反思总结）
- 是否在推理过程中实时更新记忆？（而非只在 turn end）
- 是否利用工具结果的 I/O 做提取？（oG-Memory 有，但只是补录到 transcript）

---

## 4. 如何在推理前选择正确记忆并注入上下文

oG-Memory 的检索+组装流程是**向量为主、层级递归展开、语义 slot 注入**。

### 四阶段检索流水线

[retrieval/pipeline.py](https://gitcode.com/opengauss/oGMemory/blob/dev/retrieval/pipeline.py) 的 `RetrievalPipeline.run()`：

1. **Stage 1 — QueryPlanner**：将自然语言查询分类为 `TypedQuery[]`（memory/skill/resource），推断 categories
   - [retrieval/query_planner.py](https://gitcode.com/opengauss/oGMemory/blob/dev/retrieval/query_planner.py) 使用正则匹配（`_MEMORY_PATTERNS`、`_SKILL_PATTERNS`、`_RESOURCE_PATTERNS`）
   - 支持 intent 分类（`RetrievalIntentClassifier`）

2. **Stage 2 — SeedRetriever**：全局向量搜索 L0+L2，可选 BM25 混合融合
   - [retrieval/seed_retriever.py](https://gitcode.com/opengauss/oGMemory/blob/dev/retrieval/seed_retriever.py) 支持 Vector-Anchored Fusion
   - 按租户隔离过滤（account_id + owner_space）

3. **Stage 3 — HierarchicalSearcher**：L0/L1 起始点递归展开到 L2 叶节点
   - [retrieval/hierarchical_searcher.py](https://gitcode.com/opengauss/oGMemory/blob/dev/retrieval/hierarchical_searcher.py) 优先队列驱动，分数传播公式：`final = alpha * child + (1-alpha) * parent`
   - 收敛检测：top-k URI 集连续 N 轮不变则停止
   - **热度衰减**：`blended = (1-hotness_alpha) * semantic + hotness_alpha * hotness_score`，半衰期 7 天

4. **Stage 4 — ResultRanker**：过滤 L2 叶节点、去重、排序、top-k → `RetrievedBlock[]`
   - [retrieval/result_ranker.py](https://gitcode.com/opengauss/oGMemory/blob/dev/retrieval/result_ranker.py)

### 组装与注入

[server/memory_service.py](https://gitcode.com/opengauss/oGMemory/blob/dev/server/memory_service.py) 的 `compose()` 方法将检索结果组装为 `ComposedContext`，按**语义 slot** 注入：

[core/models.py:496-520](https://gitcode.com/opengauss/oGMemory/blob/dev/core/models.py#L496-L520) 的语义 slot：

| Slot | 优先级 | 内容 | 是否可降级 |
|------|--------|------|-----------|
| **identity_context** | 0（最高） | Profile + 稳定偏好 | 不可降级 |
| **session_context** | 1 | 结构化 session 状态 | 不可降级 |
| **episodic_context** | 5 | Archive 历史 | 可降级（content → overview → abstract） |
| **task_context** | 5 | 当前任务描述 | 可降级 |
| **retrieved_evidence** | 8 | 搜索结果 working set | 可降级 + 可扩展 |

### Token 预算分配

[core/models.py:381-465](https://gitcode.com/opengauss/oGMemory/blob/dev/core/models.py#L381-L465) 的 `TokenBudget` 实现了优先级层级分配：
- 默认总预算 128,000 tokens
- 优先级 0（identity）→ 优先级 1（session）→ 优先级 5（archive）→ 优先级 8（working set）
- 预算不足时低优先级 tier 降级到 min_tokens

### 注入方式

- **Claude Code**：通过 `UserPromptSubmit` hook 返回 `additionalContext`，注入到用户 prompt
- **OpenClaw**：通过 `assemble()` 返回分层 messages，作为 system prompt 后缀

### 调研方向

- 其他项目是否使用**知识图谱检索**？（oG-Memory 有 `RelationEdge` 但检索以向量为主）
- 是否有**直接读 .md 文件**的方式？（oG-Memory 的 AGFS 底层是文件系统，但上层不直接暴露路径）
- 是否支持**角色过滤**？（如 "只检索与当前角色相关的记忆"）
- 是否支持**推理过程中动态补充检索**？（而非仅在 turn 开始时一次注入）
- 检索是否考虑**时效性**？（oG-Memory 的 hotness 衰减是一种，但非显式过期）

---

## 5. 如何处理记忆的过期、冲突、合并、版本变化

oG-Memory 通过**类型化写入策略 + 乐观锁 + Outbox 异步** 来处理这些情况。

### 写入策略（MergePolicy）

[commit/merge_policies.py](https://gitcode.com/opengauss/oGMemory/blob/dev/commit/merge_policies.py) 为每类记忆实现了不同策略：

| 策略类 | 适用类别 | 行为 | 冲突处理 |
|--------|---------|------|---------|
| `ProfilePolicy` | profile | 固定 URI，always merge | **新信息替换旧信息**（用户状态会变化） |
| `AggregateTopicPolicy` | preference, entity, pattern | slug URI，create or merge | **追加式合并**：existing + incoming，保留 provenance_ids |
| `AppendOnlyPolicy` | event, case | 每次创建新节点 | **不覆盖历史**：timestamp_uuid 保证唯一 |
| `SkillToolPolicy` | skill, tool | 固定 URI，累积式 merge | **累积**：最佳实践追加，usage_count++ |

### 冲突处理细节

- **乐观锁**：[commit/context_writer.py:106-209](https://gitcode.com/opengauss/oGMemory/blob/dev/commit/context_writer.py#L106-L209) 使用 `expected_version` 机制，并发写入时最多重试 3 次（指数退避）
- **幂等性**：每次操作生成 SHA256 fingerprint（URI + action + session_id + content_hash），防止重复写入
- **Safety Check**：在提取阶段，如果 LLM 要写一个已存在但未读过的节点，ReAct 循环会**自动重读**该节点内容并提醒 LLM 考虑现有内容

### 过期与归档

- **Archive 操作**：`WritePlan.action = "archive"` 时标记 `status: ARCHIVED` + `archived_at`，软删除而非物理删除
- **Delete 操作**：`WritePlan.action = "delete"` 时物理删除节点
- **热度衰减**：`hotness_score` 基于半衰期（默认 7 天）和访问计数衰减，影响检索排序而非记忆生命周期

### 版本变化

- 每个 `ContextNode` 的 metadata 中维护 `version` 字段
- 写入时 `expected_version` 与实际版本比对，不匹配则触发 `ConcurrentModificationError` → 重试
- `provenance_ids` 跨版本累积，追踪来源

### Outbox 异步索引

[commit/outbox_store.py](https://gitcode.com/opengauss/oGMemory/blob/dev/commit/outbox_store.py) 实现了 Outbox Pattern：
- 写入节点后注册 `OutboxEvent`（UPSERT/DELETE/MOVE 等类型）
- 异步 Worker 处理事件，更新向量索引
- 事件持久化在 AGFS，可跨进程重启恢复

### 调研方向

- 其他项目是否有显式的**过期时间**（TTL）？（oG-Memory 依赖热度衰减而非 TTL）
- 是否有**矛盾检测与修复**？（oG-Memory 有 `contradicts` 关系类型但未见主动检测机制）
- 是否有**离线合并优化**？（如定期将碎片化偏好合并为综述）
- 是否有**记忆版本树**？（oG-Memory 是扁平的 version 数字，非版本链）

---

## 6. 如何衡量记忆的好坏

oG-Memory 在 **LoCoMo 数据集** 上评估效果。

### 评估维度

| 维度 | 说明 | oG-Memory 数据 |
|------|------|---------------|
| **正确率提升** | 在 LoCoMo 数据集上的问答正确率 | 较 baseline 提升 __% （待填） |
| **Token 消耗降低** | 带记忆 vs 无记忆的 token 开销对比 | 降低 __% （待填） |
| **提取质量** | 命中率、置信度分布 | [tests/benchmark/benchmark_extraction_quality.py](https://gitcode.com/opengauss/oGMemory/blob/dev/tests/benchmark/benchmark_extraction_quality.py) |
| **检索质量** | recall、precision、MRR | [tests/e2e/eval_retrieval.py](https://gitcode.com/opengauss/oGMemory/blob/dev/tests/e2e/eval_retrieval.py) |
| **组装质量** | session 组装的 token 预算利用率 | [tests/benchmark/benchmark_assemble_session.py](https://gitcode.com/opengauss/oGMemory/blob/dev/tests/benchmark/benchmark_assemble_session.py) |

### 评估代码

- [tests/e2e/eval_quality.py](https://gitcode.com/opengauss/oGMemory/blob/dev/tests/e2e/eval_quality.py) — 统计总测试数、错误数、"no info" 回答占比
- [tests/e2e/eval_retrieval.py](https://gitcode.com/opengauss/oGMemory/blob/dev/tests/e2e/eval_retrieval.py) — 检索召回评估
- [tests/e2e/eval.py](https://gitcode.com/opengauss/oGMemory/blob/dev/tests/e2e/eval.py) — 综合评估入口
- [tests/benchmark/](https://gitcode.com/opengauss/oGMemory/blob/dev/tests/benchmark/) — 性能 benchmark

### Token 统计

HTTP API 提供 `/api/v1/token_stats` 端点，追踪跨所有 `after_turn` 调用的累计 LLM & embedding token 使用量，支持 reset。

### 调研方向

- 其他项目是否在同一数据集（LoCoMo / LongMemEval）上评估？
- 是否有**任务成功率**指标？（而非仅问答正确率）
- 是否有**用户满意度 / A/B 测试**数据？
- 是否追踪**记忆命中率**（多少比例的对话被记忆增强）？
- 是否有**记忆增长曲线**（随对话轮次增加的记忆数量和存储大小）？

---

## 总结对比要点

| 维度 | oG-Memory | 调研关注点 |
|------|-----------|-----------|
| 北向对接 | HTTP 服务 + Agent hook/plugin 双向拦截 | 其他项目是否内嵌 vs 外挂？是否支持多 Agent？ |
| 记忆模型 | 7 类 YAML schema + 三级索引 + 关系边 | 其他项目记忆类型多少？粒度？是否区分 user/agent scope？ |
| 记忆产生 | 对话后自动两阶段提取（span → tool call） | 是否有手动写入、离线优化、任务总结、推理中更新？ |
| 检索注入 | 4 阶段向量检索 + 优先级 slot 注入 + token 预算 | 是否用图检索？是否动态补充？是否角色过滤？ |
| 生命周期 | 类型化写入策略 + 乐观锁 + Outbox 异步 | 是否有 TTL？矛盾检测？版本树？离线合并？ |
| 效果衡量 | LoCoMo 数据集正确率 + token 消耗 | 是否有更多评估维度？是否跨项目可比？ |

---

## 相关链接

- [[ogmemory]] — oG-Memory 项目概述
- [[agentic-memory-evaluation]] — 记忆评估方法论
- [[locomo]] — LoCoMo 数据集