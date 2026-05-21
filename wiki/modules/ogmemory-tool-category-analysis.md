---
type: module
title: "oG-Memory Tool Category Analysis"
status: active
tags: [ogmemory, tools, extraction, architecture]
created: 2026-05-21
updated: 2026-05-21
---

# oG-Memory 工具分类分析：Extraction Tools vs 操作工具

> 日期: 2026-05-21 | 问题: Extraction tools 和其他工具（检索、写入、更新）是否属于同一类？是否应该整合？

---

## 一、现有工具分类

### 1.1 Extraction Tools（抽取工具）

```python
# extraction/tool_schemas.py 定义的工具

ExtractProfileInput      → extract_profile
ExtractPreferenceInput   → extract_preference
ExtractEntityInput       → extract_entity
ExtractEventInput        → extract_event
ExtractCaseInput         → extract_case
ExtractPatternInput      → extract_pattern
ExtractSkillInput        → extract_skill
ExtractToolInput         → extract_tool
```

**特点**：
- 被 LLM 调用
- 输入：对话内容 + 上下文
- 输出：CandidateMemory（待写入的结构化信息）
- 目的：**从非结构化文本抽取结构化信息**

### 1.2 操作工具（检索、写入、更新）

```python
# service/api.py 提供的 API（非 function calling 工具）

search_memory(query, top_k, category) → SearchMemoryResult
read_memory(uri) → RetrievedBlock
# 写入通过 commit_session() 完成，不是独立工具
```

**特点**：
- 被应用层调用（不是 LLM）
- 输入：查询/URI
- 输出：记忆内容
- 目的：**数据存储和检索**
- 目前**不是 function calling 工具**

---

## 二、本质区别分析

### 2.1 工具定位矩阵

| 维度 | Extraction Tools | 操作工具 |
|------|------------------|----------|
| **调用者** | LLM (Agent) | 应用层 |
| **输入来源** | 对话内容 | 用户/系统请求 |
| **输出目标** | CandidateMemory | 存储/检索结果 |
| **数据流向** | 对话 → 结构化记忆 | 存储系统 → Agent |
| **目的** | 信息抽取 | 数据操作 |
| **Schema 定义** | YAML schema 定义 | API 参数定义 |
| **当前形态** | Function calling 工具 | 服务 API |

### 2.2 核心差异

**Extraction Tools 是"输出工具"**：
```
Agent 看对话 → 调用 extract_* → 输出结构化信息 → 写入存储
            └───────────────────────────────────▶
                        Agent 主动输出
```

**操作工具是"输入工具"**：
```
Agent 需要信息 → 调用 search_memory → 获取记忆内容
            ▲───────────────────────────────
                        Agent 主动获取
```

### 2.3 形态差异的根源

**当前设计**：
- Extraction Tools → LLM function calling 工具（因为 Agent 需要**输出**结构化信息）
- 操作工具 → 服务 API（因为应用层需要**调用**存储系统）

**根本原因**：
- Extraction 是 Agent 的**输出动作**，需要 function calling
- 检索/写入是 Agent 的**输入动作**，目前通过应用层间接调用

---

## 三、是否应该整合？

### 3.1 整合条件

整合的前提是：**Agent 自管理**

如果 Agent 完全自主管理记忆，那么：
- Agent 需要**读取**记忆 → search_memory 应该是工具
- Agent 需要**写入**记忆 → write_memory 应该是工具
- Agent 需要**更新**记忆 → update_memory 应该是工具
- Agent 需要**删除**记忆 → delete_memory 应该是工具

此时，所有工具都变成 Agent 可调用的 function calling 工具。

### 3.2 整合方案

**如果整合为 Agent 工具体系**：

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Agent 工具体系                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   === 输入工具（Agent 获取信息）===                                   │
│   search_memory(query) → 搜索记忆                                    │
│   read_memory(uri) → 读取记忆                                        │
│   list_memories(category) → 列出记忆                                 │
│                                                                      │
│   === 输出工具（Agent 产生信息）===                                   │
│   extract_profile(...) → 抽取 profile                                │
│   extract_preference(...) → 抽取 preference                          │
│   extract_entity(...) → 抽取 entity                                  │
│   ...                                                                │
│                                                                      │
│   === 操作工具（Agent 管理记忆）===                                   │
│   write_memory(uri, content) → 写入记忆                              │
│   update_memory(uri, content) → 更新记忆                             │
│   archive_memory(uri) → 归档记忆                                     │
│   delete_memory(uri) → 删除记忆                                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.3 整合后的工作流

```
Agent 工作流（完全自管理）:

1. 用户对话 → Agent 调用 search_memory 获取相关记忆
                           ↓
2. Agent 分析对话 → 调用 extract_* 抽取新信息
                           ↓
3. Agent 判断 → 调用 write_memory 或 update_memory 写入
                           ↓
4. Agent 检查重复 → 调用 delete_memory 或 archive_memory 清理
```

---

## 四、整合利弊分析

### 4.1 整合优势

| 优势 | 说明 |
|------|------|
| **Agent 自管理** | Agent 完全控制记忆生命周期 |
| **工具一致性** | 所有工具都是 function calling，统一管理 |
| **灵活性** | Agent 动态决定何时读/写/删 |
| **自主性** | Agent 可以自主优化记忆（如 Deep Dream） |

### 4.2 整合劣势

| 劣势 | 说明 |
|------|------|
| **权限风险** | Agent 可能误删重要记忆 |
| **复杂度增加** | Agent 需要理解存储逻辑 |
| **失去分层保护** | 应用层无法过滤/校验操作 |
| **调试困难** | Agent 行为难以预测和调试 |

### 4.3 现有设计的保护层

```
现有架构（分层保护）:

Agent → Extraction Tools → CandidateMemory → 应用层校验 → 写入存储
                                    └───────────────────────┘
                                          应用层保护
```

- Extraction 输出的是 CandidateMemory（候选），不是最终写入
- 应用层通过 MergePolicy 决定实际动作
- 有去重、合并、校验等保护机制

---

## 五、推荐方案

### 5.1 不推荐完全整合

原因：
1. **分层保护有价值**：应用层校验防止 Agent 误操作
2. **Extraction 工具定位正确**：它是输出工具，已经 function calling
3. **操作工具需要保护**：检索/写入/删除不应直接暴露给 Agent

### 5.2 推荐渐进式整合

**阶段 1：Extraction 保持现状**
- Extraction Tools 继续作为 function calling 工具
- 输出 CandidateMemory，通过应用层写入

**阶段 2：检索工具化**
- search_memory 可以作为 Agent 工具（只读操作，风险低）
- 让 Agent 能主动获取相关记忆

**阶段 3：写入工具化（谨慎）**
- write_memory 作为 Agent 工具，但**受限**
- 只能写入特定 category（如 event/case）
- profile/preference 等关键记忆仍通过应用层

**阶段 4：删除工具化（最谨慎）**
- delete_memory 仅限特定场景
- 如 Deep Dream 清理重复记忆
- 需要明确授权和校验

### 5.3 工具分层建议

```
┌─────────────────────────────────────────────────────────────────────┐
│                       工具分层架构                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Layer 1: Agent 可自由调用                                         │
│   ─────────────────────────────                                     │
│   search_memory   → 只读，安全                                       │
│   read_memory     → 只读，安全                                       │
│   extract_*       → 输出结构化信息                                   │
│                                                                      │
│   Layer 2: Agent 受限调用                                           │
│   ─────────────────────────────                                     │
│   write_memory    → 仅限 event/case 等追加类型                       │
│   archive_memory  → 仅限 Deep Dream 等清理场景                       │
│                                                                      │
│   Layer 3: 应用层控制                                               │
│   ─────────────────────────────                                     │
│   update_memory   → profile/preference 等关键记忆                    │
│   delete_memory  → 高风险操作                                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 六、结论

### 6.1 本质区别确认

**Extraction Tools 和操作工具确实是不同类工具**：
- Extraction Tools = 输出工具（Agent 产生信息）
- 操作工具 = 输入/管理工具（Agent 获取/管理信息）

### 6.2 当前设计合理

现有设计保持分层保护：
- Extraction 输出候选 → 应用层校验 → 存储
- 检索/写入通过 API → 应用层控制

### 6.3 整合方向

**不是完全整合，而是渐进开放**：
1. 检索工具先开放（安全）
2. 写入工具受限开放（event/case）
3. 删除工具最谨慎开放（Deep Dream 专用）

### 6.4 Deep Dream 的影响

Deep Dream 正是 Layer 2 的典型场景：
- 需要读取记忆 → search_memory（Layer 1）
- 需要写入新记忆 → write_memory（Layer 2）
- 需要清理重复 → archive_memory（Layer 2）

---

## 参考

- [[ogmemory-tool-definition-refactor-analysis]] - 工具定义改造分析
- [[ogmemory-deep-dream-framework]] - Deep Dream 设计文档