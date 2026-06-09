---
type: concept
title: "Deep Dream 修改已有记忆的可行性分析：存储层接口与版本管理"
created: 2026-06-08
updated: 2026-06-08
tags:
  - research
  - ai/memory
  - dreaming
  - storage
  - agfs
  - modification
  - versioning
related:
  - "[[ogmemory-deep-dream-framework]]"
  - "[[agent-dream-consolidation-landscape-2025]]"
  - "[[oG-Memory Extraction and Storage Analysis]]"
---

# Deep Dream 修改已有记忆的可行性分析

> [!abstract]
> 分析了 oG-Memory 存储层是否支持 Dream 对已有记忆的修改/评分影响。**核心结论**：ContextFS 协议层面已预留 `write_node`（upsert）、`archive_node`（软删除）、`delete_node`（硬删除）接口，但当前 Dream 只走 `add_only` 路径，**未利用已有的 merge/update 能力**。版本管理采用乐观锁（optimistic locking），但**没有历史版本存储**——旧内容直接覆盖。Dream 想修改已有记忆，需要在现有 MergePolicy 框架内扩展，而非绕过它。

---

## 一、ContextFS 协议层：修改接口已预留

`core/interfaces.py:26-101` 定义了 `ContextFS` Protocol，包含以下操作：

| 方法 | 功能 | 当前 Dream 是否使用 |
|------|------|-------------------|
| `write_node(node, ctx)` | 写入/更新节点（atomic upsert） | ✅ Dream 用此写入新 dream 记忆 |
| `read_node(uri, ctx)` | 读取节点完整内容 | ✅ Deep Dream ReActLoop 用 `read` 工具读取 |
| `delete_node(uri, ctx)` | 永久删除节点 | ❌ 未使用 |
| `archive_node(uri, ctx)` | 软删除（标记 ARCHIVED） | ❌ 未使用 |
| `move_node(from_uri, to_uri, ctx)` | 重命名/迁移节点 | ❌ 未使用 |
| `exists(uri, ctx)` | 检查节点是否存在且 ACTIVE | ✅ MergePolicy 内部使用 |
| `list_children(uri, ctx)` | 列出子节点 | ❌ 未使用 |

**关键发现**：`write_node` 不是单纯的"创建"，而是**atomic upsert**——同一个 URI 的节点会被覆盖更新。这意味着如果 Dream 想修改已有记忆的内容，**接口层面已经支持**——只需对同一 URI 调用 `write_node` 即可。

但当前 `DreamPolicy` 通过 content hash 在 routing_key 中保证唯一性，**永远不会命中已有 URI**，因此 `write_node` 在 Dream 路径上实际只执行 INSERT。

---

## 二、版本管理：乐观锁存在，但无历史版本

### 2.1 乐观锁机制

`fs/agfs_adapter/agfs_context_fs.py:540-558` 实现了乐观锁：

```python
# Step 0.5: Optimistic locking check for merge operations
expected_version = node.metadata.get("expected_version")
if expected_version is not None:
    # Read existing .meta.json
    actual_version = existing_meta.get("version", 0)
    if actual_version != expected_version:
        raise ConcurrentModificationError(node.uri, expected_version, actual_version)
```

`commit/context_writer.py:107-209` 实现了重试机制：
- 最多 3 次重试，指数退避（0.1s, 0.2s, 0.4s）
- 每次重试会重新读取最新版本再 plan

### 2.2 版本号的处理

`agfs_context_fs.py:595-612`：

```python
if expected_version is not None:
    new_version = expected_version + 1  # merge: 从 expected 版本递增
else:
    new_version = node.metadata.get("version", 1)  # create: 默认版本 1
```

### 2.3 没有历史版本存储

**关键限制**：版本号只用于乐观锁，**不保存旧版本内容**。写入新版本时，旧内容的 5 个文件（content.md, .relations.json, .abstract.md, .overview.md, .meta.json）直接覆盖。

这意味着：
- ✅ 可以安全地并发修改同一节点（乐观锁保证）
- ❌ 无法回溯"修改前是什么"（没有历史快照）
- ❌ 无法对比"dream 修改前后差异"

**对 Dream 的影响**：如果 Dream 修改了已有记忆，修改是不可逆的——除非自行在 dream 记忆中保存原始内容的 provenance 引用。

---

## 三、Write Pipeline：MergePolicy 是修改的唯一合法路径

### 3.1 当前 5 种 MergePolicy

| Policy | 适用类型 | operation_mode | 能否修改已有节点 |
|--------|---------|---------------|--------------|
| `ProfilePolicy` | profile | upsert + single_file | ✅ 总是 merge（新覆盖旧） |
| `AggregateTopicPolicy` | preference/entity/pattern | upsert + multi_file | ✅ 存在时 merge，不存在时 create |
| `AppendOnlyPolicy` | event/case | add_only | ❌ 总是 create 新节点 |
| `DreamPolicy` | dream | add_only | ❌ 总是 create 新节点 |
| `SkillToolPolicy` | skill/tool | upsert + agent scope | ✅ 存在时 merge（累积式） |

### 3.2 MergePolicy 的 plan → build → write 三步流程

1. **plan**: MergePolicy 生成 `WritePlan(action, target_uri, merged_fields)`
2. **build**: ArchiveBuilder 从 CandidateMemory + WritePlan 构建 ContextNode
   - merge action 时，ArchiveBuilder 调用 `_llm_merge_content()` 或 `_merge_overview()` 合并新旧内容
3. **write**: ContextWriter 调用 `fs.write_node(node, ctx)` 写入

### 3.3 PolicyRouter 路由逻辑

`commit/policy_router.py:74-145` 根据 schema 属性路由：

```python
# add_only → AppendOnlyPolicy (event/case/dream)
# upsert + single_file → ProfilePolicy (profile)
# upsert + multi_file → AggregateTopicPolicy (preference/entity/pattern)
# agent scope + skill/tool → SkillToolPolicy
```

**关键设计**：路由由 schema 的 `operation_mode` 和 `is_single_file` 驱动，**不是硬编码 category 映射**。这意味着新增 Dream 修改行为，可以**通过新增 schema 配置**而非修改代码。

---

## 四、Node 状态生命周期

`core/enums.py` 定义了 NodeStatus：

| 状态 | 含义 | 对检索可见 |
|------|------|----------|
| PENDING | 写入未完成（没有 .meta.json） | ❌ 不可见 |
| ACTIVE | 正常可用 | ✅ 可见 |
| BROKEN | 需人工干预 | ❌ read_node 会抛 NodeBrokenError |
| ARCHIVED | 软删除 | ❌ read_node 会抛 NodeNotFoundError |

**缺失的状态**：没有 `DEPRECATED` 或 `SUPERSEDED` 状态。如果 Dream 想标记已有记忆为"过时/已被晋升替代"，需要新增此状态。

---

## 五、RelationEdge：已有但未充分利用的关系机制

`core/models.py:130-141`：

```python
@dataclass
class RelationEdge:
    from_uri: str
    to_uri: str
    relation_type: str  # e.g. "related_to" | "derived_from" | "contradicts" | "SEQUENCE"
    weight: float  # 0.0-1.0
    reason: str
```

`RelationStore` 协议 (`core/interfaces.py:103-130`) 提供：
- `get_edges(uri, ctx)` — 获取出边
- `upsert_edges(edges, ctx)` — 添加/更新边
- `get_one_hop(uri, ctx, limit)` — 获取 top-N 高权重出边

**当前 Dream 的使用**：DreamPolicy 的 WritePlan 不包含 `relation_edges`（`relation_edges=[]`）。Dream 记忆与源记忆之间只有 `provenance_ids`（纯文本 URI 列表），没有结构化的 `RelationEdge`。

**可利用的设计**：如果 Dream 修改了已有记忆的评分，可以通过 `RelationEdge` 建立：
- `dream_node → source_node: SUPERSIDES`（替代关系，weight=confidence）
- `dream_node → source_node: DERIVED_FROM`（已有，通过 provenance_ids）
- `source_node → dream_node: PROMOTED_BY`（反向引用）

---

## 六、MemoryWriteAPI：公开接口层的限制

`service/api.py:24-420` 的 `MemoryWriteAPI` 只暴露了写入相关方法：

| 方法 | 功能 |
|------|------|
| `commit_session(messages, ctx)` | 从对话抽取并写入 |
| `write_raw_chunk(messages, ctx)` | 写入原始对话存档 |
| `write_memory(candidate, ctx)` | 写入单个预构建的 CandidateMemory |
| `write_memories(candidates, ctx)` | 写入多个 CandidateMemory |

**未暴露的方法**：
- ❌ `archive_node(uri, ctx)` — 存在但只在 ContextWriter 内部使用
- ❌ `delete_node(uri, ctx)` — 同上
- ❌ `update_metadata(uri, metadata_updates, ctx)` — 不存在
- ❌ `add_relation(edges, ctx)` — 不存在（RelationStore.upsert_edges 是内部接口）

---

## 七、Dream 修改已有记忆的可行路径分析

假设 Dream 的产出会影响已有记忆（如：降低评分、标记过时、修正矛盾），有以下可行路径：

### 路径 A：通过 CandidateMemory + MergePolicy（最符合现有架构）

**思路**：Dream 产出 `CandidateMemory`，category 设为已有记忆的 category（而非 "dream"），走对应的 MergePolicy merge 路径。

| 修改类型 | category | 走哪个 Policy | 修改方式 |
|---------|----------|-------------|---------|
| 修正 preference 矛盾 | preference | AggregateTopicPolicy | merge → LLM 合并，新覆盖旧 |
| 修正 entity 信息 | entity | AggregateTopicPolicy | merge → LLM 合并 |
| 更新 profile | profile | ProfilePolicy | merge → 新覆盖旧 |

**优势**：
- 完全符合现有 pipeline（plan → build → write）
- 乐观锁保证并发安全
- provenance_ids 自然合并（源记忆 URI + dream URI 都保留）
- 向量索引自动同步（outbox event）

**限制**：
- 只能 merge 内容，不能只修改 metadata（如只改 confidence score）
- merge 的语义是"新旧合并"，不是"旧被标记为过时"
- LLM 合并可能产生不可控的结果

### 路径 B：新增 WriteAction.ARCHIVE + 源记忆 URI（利用已有但未暴露的 archive）

**思路**：Dream 产出包含 `action=archive` 的指令，让被晋升替代的源记忆标记为 ARCHIVED。

| 步骤 | 代码路径 |
|------|---------|
| 1. Dream ReActLoop 收集 archive 指令 | `_dream_outputs.append(ArchiveDirective(uri=...))` |
| 2. 转换为 WritePlan(action=archive) | 类似 ContextWriter._handle_archive() |
| 3. 执行 archive | fs.archive_node(uri, ctx) |
| 4. 注册 outbox 事件 | 删除向量索引记录 |

**优势**：
- archive 已有完整实现（AGFS + SQL adapter 都支持）
- 软删除后节点不再出现在检索中
- 可以恢复（理论上可以把 ARCHIVED 改回 ACTIVE）

**限制**：
- MemoryWriteAPI 未暴露 archive 操作，需要新增 API 方法
- archive 是不可逆的检索影响——源记忆从此不再被检索到
- 没有保留"为什么被 archive"的理由

### 路径 C：新增 metadata-only update + DEPRECATED 状态（需扩展设计）

**思路**：新增 `NodeStatus.DEPRECATED` 和 metadata-only update 机制，让 Dream 可以只修改已有记忆的 metadata（添加 deprecated_at、superseded_by、dream_confidence_adjustment 等），不改变内容。

**需要新增的代码**：

```
1. core/enums.py: NodeStatus 新增 DEPRECATED
2. core/interfaces.py: ContextFS 新增 update_metadata(uri, updates, ctx)
3. service/api.py: MemoryWriteAPI 新增 update_node_metadata(uri, metadata, ctx)
4. 检索层：检索过滤排除 DEPRECATED 状态节点（类似 ARCHIVED 的处理）
5. RelationStore: 新增 SUPERSIDES_BY 关系类型
```

**优势**：
- 最精确的控制——只改 metadata，不改内容
- 源记忆仍然存在于存储中，但检索时降权或排除
- 可以追溯"被什么 dream 替代"（superseded_by URI）
- 最符合"Sleep Rewrites Rules"的理念——记忆不是消失，而是被整合升级

**限制**：
- 需要较多新增代码（新状态、新接口、新 API 方法、检索层适配）
- metadata 变化不会触发向量索引更新（需要设计：是否重新 embed？）
- 需要在 read_node 中处理 DEPRECATED 状态（返回还是拒绝？）

### 路径 D：Dream 写入新节点 + 关系指向旧节点（最小改动方案）

**思路**：Dream 不修改已有记忆本身，只写入新的 dream 记忆，并通过 `RelationEdge` 建立关系。检索层在遇到 `SUPERSEDED_BY` 关系时自动降权源记忆。

```
已有记忆: ctx://acme/users/alice/memories/preference/coding_style
  → RelationEdge(to=dream_node, type=SUPERSEDED_BY, weight=0.85)

Dream 记忆: ctx://acme/users/alice/memories/dream/promotion_coding_style_xxxx
  → provenance_ids 包含源记忆 URI
```

**优势**：
- **零改动现有存储层**——完全利用已有的 RelationEdge + DreamPolicy
- 源记忆内容不变，只是检索时通过关系降权
- 溯源链天然保留（源记忆 → dream 记忆，双向可查）

**限制**：
- 检索层需要新增逻辑：遇到 SUPERSEDED_BY 关系时降权/排除
- 源记忆和 dream 记忆内容可能重复（源记忆的 coding_style 信息 vs dream 记忆的晋升版 coding_style）
- 需要新增一个 prompt-driven tool 让 LLM 建立 SUPERSEDED_BY 关系

---

## 八、推荐方案

### 短期（最小改动）：路径 D

Dream 不修改已有记忆，只通过 `RelationEdge` 建立关系，检索层根据关系降权。

**具体实现**：
1. `process_promotion` validator 修改：当产出被判定为 valid 时，额外返回 `superseded_uris`（被此晋升替代的源记忆 URI 列表）
2. DeepDreamReActLoop 收集 superseded 关系
3. 写入时在 `WritePlan.relation_edges` 中加入 `SUPERSEDED_BY` 关系
4. 检索层（HierarchicalSearcher 或 SeedRetriever）新增过滤逻辑：遇到 SUPERSEDED_BY 出边时降低源记忆权重

### 中期（架构扩展）：路径 C

新增 `NodeStatus.DEPRECATED` 和 metadata-only update。

**具体实现**：
1. `core/enums.py` 新增 `DEPRECATED = "DEPRECATED"`
2. `ContextFS` 协议新增 `update_node_metadata(uri, metadata_updates, ctx)` → 只修改 `.meta.json`
3. 新增 prompt-driven tool `process_deprecation`（标记已有记忆为过时）
4. 检索层在 `exists()` 和 `read_node()` 中排除 DEPRECATED
5. 新增 `SUPERSIDES_BY` RelationEdge 类型

### 长期（完整架构）：路径 A + C 混合

Dream 产出既包含新的 dream 记忆（走 DreamPolicy add_only），也包含对已有记忆的修改指令（走对应 category 的 MergePolicy merge 或新 deprecation 逻辑）。

---

## 九、关键代码索引

| 功能 | 文件 | 行号 |
|------|------|------|
| ContextFS Protocol | `core/interfaces.py` | 26-101 |
| ContextNode 模型 | `core/models.py` | 86-103 |
| RelationEdge 模型 | `core/models.py` | 130-141 |
| CandidateMemory 模型 | `core/models.py` | 144-167 |
| WriteAction/WritePlan | `core/models.py` | 169-191 |
| NodeStatus Enum | `core/enums.py` | 9-20 |
| AGFS write_node 实现 | `fs/agfs_adapter/agfs_context_fs.py` | 521-620 |
| AGFS 乐观锁检查 | `fs/agfs_adapter/agfs_context_fs.py` | 540-558 |
| AGFS 版本号处理 | `fs/agfs_adapter/agfs_context_fs.py` | 595-612 |
| AGFS archive_node | `fs/agfs_adapter/agfs_context_fs.py` | 771+ (通过 ContextWriter._handle_archive) |
| ProfilePolicy | `commit/merge_policies.py` | 31-94 |
| AggregateTopicPolicy | `commit/merge_policies.py` | 97-172 |
| DreamPolicy | `commit/merge_policies.py` | 233-258 |
| PolicyRouter | `commit/policy_router.py` | 29-225 |
| ContextWriter（写入编排） | `commit/context_writer.py` | 22-360 |
| ArchiveBuilder（节点构建） | `commit/archive_builder.py` | 21-252 |
| MemoryWriteAPI（公开接口） | `service/api.py` | 24-420 |

---

## 参考

- [[ogmemory-deep-dream-framework]] — Deep Dream 框架 v3.0，当前 DreamPolicy add_only 设计
- [[agent-dream-consolidation-landscape-2025]] — 2025 研究全景与改进路线图
- [[oG-Memory Extraction and Storage Analysis]] — 抽取与存储架构分析