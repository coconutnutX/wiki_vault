---
type: module
title: "LongMemEval Dream_36 测试报告"
created: 2026-06-11
updated: 2026-06-11
tags: [longmemeval, deepdream, test, owner_space-bug, benchmark]
related:
  - ogmemory-deep-dream-framework
  - ogmemory-deep-dream-e2e-test-guide
  - ogmemory-recalllogger-bug-retrospective
---

# LongMemEval Dream_36 测试报告

2026-06-11 执行的 dream_36 端到端测试完整记录，包括 owner_space 跨用户泄漏 bug 的发现与修复，以及 before/after-dream 对比分析。

关联：[[ogmemory-deep-dream-framework]]、[[ogmemory-deep-dream-e2e-test-guide]]

## 1. 测试配置

| 参数 | 值 |
|------|-----|
| 数据集 | `longmemeval_s_dream_36.json`（36题：12 multi-session + 12 temporal-reasoning + 12 knowledge-update） |
| run_id | `84325a8b` |
| user_prefix | `dream-36` |
| user_id 格式 | `dream-36-84325a8b-{qid}` |
| account_id | `longmemeval` |
| agent_id | `openclaw-longmemeval` |
| compose API | `http://127.0.0.1:8090` |
| reader/judge model | `doubao-seed-2-0-code-preview-260215` |
| reader/judge API | `https://ark.cn-beijing.volces.com/api/coding/v3` |

## 2. 文件目录

```
/data/Workspace2/repos/longmemeval/output/
├── dream-36-ingest/                    # Step 3: 原始记忆注入
│   ├── run_config.json                 # 运行配置
│   ├── details.jsonl                   # 注入详情
│   ├── pipeline.log                    # 注入日志
│   └── deepdream_results.json          # Step 5: DeepDream 结果
│
├── before-dream-84325a8b/              # Step 4: 旧版 before-dream QA（有 bug，中断）
│   ├── details.jsonl                   # 11题结果（5成功 + 6超时）
│   └── ogmemory-compose.hypotheses.jsonl
│
├── before-dream-v2-84325a8b/           # Step 4: 修复后 before-dream QA
│   ├── details.jsonl                   # 36题完整结果
│   ├── results.json                    # 统计结果
│   └── ogmemory-compose.hypotheses.jsonl
│
├── after-dream-84325a8b/               # Step 6: after-dream QA
│   ├── details.jsonl                   # 36题完整结果
│   ├── results.json                    # 统计结果
│   └── ogmemory-compose.hypotheses.jsonl
│
├── before-dream-9996050a/              # 旧版 dream_9 before-dream（9题，无 owner_space 修复）
├── after-dream-9996050a/               # 旧版 dream_9 after-dream（9题，无 owner_space 修复）
├── dream-smoke9-ingest/                # 旧版 dream_9 注入
```

## 3. 数据库写入量

### vector_index 表（记忆向量）

| 指标 | 值 |
|------|-----|
| 总记录数 | 30,730 |
| distinct owner_spaces | 37（36 user + 1 agent） |
| distinct accounts | 1（longmemeval） |

### 各 owner_space 记忆分布

| owner_space                       | 记忆数      | 说明                     |
| --------------------------------- | -------- | ---------------------- |
| `user:dream-36-84325a8b-b3c15d39` | 1240     | 最高                     |
| `user:dream-36-84325a8b-9ea5eabc` | 1218     |                        |
| ... (36 个 user)                   | 710-1240 | 平均 ~800                |
| `agent:openclaw-longmemeval`      | 462      | agent 级别（含跨用户 case 记忆） |
| `user:dream-36-84325a8b-6a1eabeb` | 256      | 最低（DeepDream 新增记忆稀释）   |

### DeepDream 新增记忆

| 指标 | 值 |
|------|-----|
| 成功/失败 | 36/36 |
| 新增 dream URI 总数 | 79 |
| 平均 iterations | 10.2 |
| 平均耗时 | 77.1s |
| 状态 | 全部 `partial`（未达到 complete 条件） |

## 4. owner_space 跨用户泄漏 Bug

### 发现过程

Step 4 before-dream QA 评测时，11题中只有5题成功完成，6题超时。成功的5题准确率 0%，且检索结果中混入了其他 user_id 的记忆。

### Bug 根因

**`retrieval/seed_retriever.py` 的 `_global_vector_search()` 向量搜索只按 `account_id` 过滤，不按 `owner_space` 过滤。**

具体代码路径：

1. QA eval 调用 compose，传入 `userId=dream-36-84325a8b-4f54b7c9`
2. `build_context()` 构建 ctx，但 auth 未启用时 `visible_owner_spaces=()`（空 tuple）
3. `_global_vector_search()` 构建 filters: `{account_id: "longmemeval", level: [0,1,2]}` — **没有 owner_space**
4. 向量搜索在 27,234 条记忆中跨 36 个 user 全局检索
5. 结果中大量来自其他 user 的记忆，正确用户的记忆被稀释

同时，`_get_root_uris()` 正确用 `ctx.user_id` 构建了 `ctx://longmemeval/users/{uid}/memories/`，但这只用于 `_merge_starting_points()` 合合逻辑，不影响 SQL WHERE 条件。

### Bug 影响

- **跨用户泄漏**：5题/5题都有其他 user 记忆混入
- **性能**：compose 耗时 158-177s（27k 全量搜索），55% 超时
- **准确率**：0/5（0%）

### 修复方案（OpenCode 执行）

**文件1：`server/memory_service.py`**

修复 `build_context()` — auth 未启用时不再留空 `visible_owner_spaces`：

```python
# 修复前
visible_owner_spaces=()

# 修复后
visible_owner_spaces: list[str] = [f"user:{user_id}"]
if requested_agent_id:
    visible_owner_spaces.append(f"agent:{requested_agent_id}")
visible_owner_spaces=tuple(dict.fromkeys(visible_owner_spaces))
```

**文件2：`retrieval/seed_retriever.py`**

新增 `_resolve_owner_filter()` 静态方法，强制加入 owner_space 过滤：

- `typed_query.owner_space` 有值 → 验证权限（`PermissionError`）
- `ctx.visible_owner_spaces` 有值 → 直接用
- **两者都无 → `raise ValueError`**，明确报错

`_global_vector_search()` 和 `_bm25_search()` 都改用 `_resolve_owner_filter()`。

### 设计原则

遵循 [[No Fallback / No Silent Error Handling]] 原则：`visible_owner_spaces` 缺失不是防御性编程场景，而是必须设置的值。没有 → 报错，而不是空 tuple 静默跳过。

### 残留问题：agent 级别 case 记忆

修复后仍有 3 题（07741c45、10e09553、6a1eabeb）存在跨用户泄漏，来源是 `owner_space=agent:openclaw-longmemeval` 的 `case` 类别记忆。这些记忆属于 agent 空间，在 ingest 时通过 agent 写入，内容里嵌入了不同 user_id 的信息。compose 的 `_get_root_uris()` 将 agent 空间视为可见空间，所以这些 case 记忆被合法检索到——但内容层面引用了其他用户的信息。

这需要在 ingest 或 compose 层面对 case 记忆做进一步的 user 关联过滤。

## 5. 准确率结果

### 总览对比

| 阶段 | 完成题数 | 超时 | 准确率 | compose avg | 跨用户泄漏 |
|------|---------|------|--------|------------|-----------|
| 旧版 before-dream（bug版） | 11/36 | 6 | 0/5 (0%) | 170s | 5/5题 |
| 修复后 before-dream-v2 | 36/36 | 0 | 22/36 (61.1%) | 32s | 3题 |
| after-dream | 36/36 | 0 | 22/36 (61.1%) | 23s | 1题 |
| 旧版 dream_9 before-dream | 9/9 | 0 | 2/9 (22.2%) | — | — |
| 旧版 dream_9 after-dream | 9/9 | 0 | 2/9 (22.2%) | — | — |

### 按题型分解

| 题型 | before-dream-v2 | after-dream | 变化 |
|------|-----------------|-------------|------|
| multi-session | 7/12 (58.3%) | 6/12 (50.0%) | -1 |
| temporal-reasoning | 8/12 (66.7%) | 9/12 (75.0%) | +1 |
| knowledge-update | 7/12 (58.3%) | 7/12 (58.3%) | 0 |
| **总计** | **22/36 (61.1%)** | **22/36 (61.1%)** | **0** |

### 每题对比

| QID           | 题型                 | Before  | After   | compose(前→后) | 变化     |
| ------------- | ------------------ | ------- | ------- | ------------ | ------ |
| 07741c45      | knowledge-update   | CORRECT | WRONG   | 45→16        | LOST   |
| 0db4c65d      | temporal-reasoning | CORRECT | CORRECT | 15→16        |        |
| 0ddfec37      | knowledge-update   | WRONG   | WRONG   | 24→18        |        |
| 10e09553      | knowledge-update   | WRONG   | CORRECT | 26→17        | GAINED |
| 18bc8abd      | knowledge-update   | CORRECT | CORRECT | 27→31        |        |
| 2318644b      | multi-session      | WRONG   | WRONG   | 15→18        |        |
| 28dc39ac      | multi-session      | WRONG   | WRONG   | 15→45        |        |
| 3fdac837      | multi-session      | CORRECT | CORRECT | 36→22        |        |
| 41698283      | knowledge-update   | WRONG   | WRONG   | 38→18        |        |
| 45dc21b6      | knowledge-update   | CORRECT | WRONG   | 15→16        | LOST   |
| 4f54b7c9      | multi-session      | CORRECT | CORRECT | 31→17        |        |
| 5025383b      | multi-session      | CORRECT | CORRECT | 62→16        |        |
| 5a7937c8      | multi-session      | WRONG   | WRONG   | 16→17        |        |
| 60472f9c      | multi-session      | CORRECT | CORRECT | 25→17        |        |
| 6a1eabeb      | knowledge-update   | WRONG   | CORRECT | 41→39        | GAINED |
| 6aeb4375      | knowledge-update   | CORRECT | CORRECT | 16→16        |        |
| 6cb6f249      | multi-session      | CORRECT | CORRECT | 16→16        |        |
| 6d550036      | multi-session      | CORRECT | WRONG   | 47→25        | LOST   |
| 7401057b      | knowledge-update   | WRONG   | WRONG   | 19→19        |        |
| 7e974930      | knowledge-update   | CORRECT | CORRECT | 37→16        |        |
| 9ea5eabc      | knowledge-update   | CORRECT | CORRECT | 38→37        |        |
| aae3761f      | multi-session      | WRONG   | WRONG   | 15→46        |        |
| b3c15d39      | multi-session      | CORRECT | CORRECT | 44→18        |        |
| bbf86515      | temporal-reasoning | WRONG   | CORRECT | 57→18        | GAINED |
| c8090214      | temporal-reasoning | CORRECT | CORRECT | 62→28        |        |
| cc5ded98      | knowledge-update   | CORRECT | CORRECT | 15→16        |        |
| dcfa8644      | temporal-reasoning | CORRECT | CORRECT | 45→16        |        |
| gpt4_0b2f1d21 | temporal-reasoning | CORRECT | CORRECT | 15→15        |        |
| gpt4_2d58bcd6 | temporal-reasoning | CORRECT | CORRECT | 75→15        |        |
| gpt4_31ff4165 | multi-session      | WRONG   | WRONG   | 37→47        |        |
| gpt4_7abb270c | temporal-reasoning | WRONG   | WRONG   | 48→17        |        |
| gpt4_7f6b06db | temporal-reasoning | WRONG   | WRONG   | 45→15        |        |
| gpt4_d31cdae3 | temporal-reasoning | CORRECT | CORRECT | 15→45        |        |
| gpt4_e061b84f | temporal-reasoning | WRONG   | WRONG   | 15→48        |        |
| gpt4_fa19884c | temporal-reasoning | CORRECT | CORRECT | 45→16        |        |
| gpt4_fe651585 | temporal-reasoning | CORRECT | CORRECT | 17→18        |        |

**Gained: 3 / Lost: 3 / Same: 30**

## 6. 分析与发现

### 6.1 owner_space 修复的效果

修复前后对比清晰：

| 指标 | 修复前 | 修复后 | 提升 |
|------|--------|--------|------|
| compose 耗时 | ~170s | ~32s | **5倍提速** |
| 超时率 | 55% (6/11) | 0% (0/36) | **完全消除** |
| 跨用户泄漏（vector层） | 严重 | 基本消除 | |
| 准确率 | 0% | 61.1% | **+61.1%** |

提速的原因：按 owner_space 过滤后，搜索范围从 27k 条缩小到每个 user ~800 条，PG 向量搜索效率大幅提升。

### 6.2 DeepDream 的 dilution 效果

**Dream 注入对准确率没有产生预期的干扰效果。**

- 总准确率不变：61.1% → 61.1%
- Gained 3 / Lost 3，互相抵消
- compose 耗时反而更快（32s → 23s），可能因为 consolidation 合并了部分记忆，结构更紧凑

可能原因：

1. **Dream 记忆量太少**：79 条 URI 相对 27k 原始记忆比例极低（0.3%），不足以产生显著稀释
2. **owner_space 隔离生效**：dream 记忆的 owner_space 与原始记忆分开，compose 按 owner_space 过滤后 dream 记忆可能被自然排除
3. **Dream 状态为 partial**：所有 36 题 DeepDream 返回 `status=partial`，未达到 complete 条件，可能 dream 记忆的整合深度不够

### 6.3 原始准确率分析

Before-dream 61.1% 的 14 题 WRONG 中，有几类失败模式：

- **数值统计错误**：如"5个古董"答成6（混入无关项），"cooking+hiking"答成 "photography+hiking"
- **完全 miss**：compose 返回 0 条相关记忆（identityContext 太简），如健康设备、宗教活动、自驾时间
- **推理能力不足**：检索到了正确记忆但 reader 模型（doubao-seed）推理出错

### 6.4 需要后续改进的方向

1. **agent 级别 case 记忆的 user 关联过滤** — 3题跨用户泄漏的残留问题
2. **DeepDream dilution 效果增强** — 增加注入量、改善 dream 记忆与原始记忆的交错
3. **compose 0-hit 问题的 profile 增强** — 提升低命中题的记忆覆盖
4. **reader 模型推理质量** — doubao-seed 作为 reader/judge 的推理能力可能不如 gpt-4o-mini

## 7. 数据查看指南

### 7.1 从 output 文件查看某一题的检索和结果

QA eval 的 `details.jsonl` 记录了每题的完整过程，包含 compose 检索结果、reader 回答、judge 判定。

```python
import json
with open('output/before-dream-v2-84325a8b/details.jsonl') as f:
    for line in f:
        d = json.loads(line)
        if d['question_id'] == '4f54b7c9':   # 替换为目标 qid
            # compose 检索结果
            print(d['memory_context']['identityContext'])    # Profile 摘要（一句话）
            print(d['memory_context']['episodicContext'])    # Archive 历史
            print(d['memory_context']['sessionContext'])     # Session 状态摘要
            print(d['memory_context']['retrievedEvidence'])  # 检索到的记忆原文（核心）
            # 最终结果
            print(d['hypothesis'])   # reader 模型基于检索结果生成的回答
            print(d['result'])       # judge 判定 (CORRECT/WRONG)
            print(d['expected'])     # 正确答案
            print(d['compose_stats']) # compose 检索统计
```

关键字段说明：

| 字段 | 含义 |
|------|------|
| `memory_context.retrievedEvidence` | compose 检索到的记忆原文（15条，按得分排序） |
| `memory_context.identityContext` | Profile 层（一句话身份摘要） |
| `memory_context.sessionContext` | Session 摘要（结构化的 session 状态） |
| `hypothesis` | reader 模型基于检索结果生成的回答 |
| `result` | judge 判定（CORRECT/WRONG） |
| `expected` | 正确答案 |
| `compose_stats.hit_categories` | 检索到的记忆类别列表 |
| `compose_stats.hit_count` | 检索到的记忆数 |
| `compose_elapsed_ms` | compose 耗时（ms） |

### 7.2 从 SQL 查看某个 user 的抽取内容

数据库中 `context_nodes`、`session_archives`、`relation_edges` 表均为空，所有数据都在 `vector_index` 表。

连接信息：`host=127.0.0.1 port=5432 dbname=ogmemory user=ogmem password=ogmem123`

#### 记忆结构

每个抽取单元有 **abstract + content** 两层（`metadata->>'level'` 区分），URI 格式：

- abstract: `ctx://{account}/users/{uid}/memories/{category}/{name}`
- content: `ctx://{account}/users/{uid}/memories/{category}/{name}/content.md`

#### 常用查询

```sql
-- 查某个 user 的记忆按 category 分组统计
SELECT metadata->>'category' as category,
       metadata->>'level' as level,
       count(*) as count,
       sum(length(text)) as total_chars
FROM vector_index
WHERE filters->>'owner_space' = 'user:dream-36-84325a8b-4f54b7c9'
GROUP BY 1, 2 ORDER BY 1, 2;

-- 查某个 user 的 entity 摘要（abstract 层，简短）
SELECT uri, text
FROM vector_index
WHERE filters->>'owner_space' = 'user:dream-36-84325a8b-4f54b7c9'
  AND metadata->>'category' = 'entity'
  AND metadata->>'level' = 'abstract'
ORDER BY uri;

-- 查某个记忆的完整 content（详细版）
SELECT uri, text
FROM vector_index
WHERE filters->>'owner_space' = 'user:dream-36-84325a8b-4f54b7c9'
  AND metadata->>'category' = 'entity'
  AND metadata->>'level' = 'content'
  AND uri LIKE '%antique%'
ORDER BY uri;

-- 查所有 owner_space 的记忆分布
SELECT filters->>'owner_space' as owner_space, count(*)
FROM vector_index
WHERE filters->>'account_id' = 'longmemeval'
GROUP BY 1 ORDER BY count DESC;

-- 查 agent 空间的 case 记忆（跨用户泄漏来源）
SELECT uri, left(text, 100) as preview
FROM vector_index
WHERE filters->>'owner_space' = 'agent:openclaw-longmemeval'
  AND metadata->>'category' = 'case'
  AND metadata->>'level' = 'abstract'
ORDER BY uri;

-- 搜索提到特定 user 的 case 记忆
SELECT uri, text
FROM vector_index
WHERE filters->>'owner_space' = 'agent:openclaw-longmemeval'
  AND metadata->>'category' = 'case'
  AND metadata->>'level' = 'content'
  AND text LIKE '%4f54b7c9%';

-- 查 dream 相关表
SELECT run_id, phase, owner_space, status, trigger, started_at
FROM dream_runs ORDER BY started_at DESC;

SELECT count(*) FROM dream_signals;
SELECT count(*) FROM dream_staged_candidates;
```

#### vector_index 表结构

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | varchar | 记录 ID |
| `uri` | varchar | 记忆 URI（唯一标识） |
| `text` | text | 记忆文本内容 |
| `embedding` | vector | 嵌入向量 |
| `filters` | jsonb | 过滤字段：`account_id`, `owner_space` |
| `metadata` | jsonb | 元数据：`category`, `level`, `context_type`, `parent_uri`, `has_content`, `has_overview`, `routing_key`, `when`, `who`, `where` |
| `level` | integer | 目录层级（0/1/2） |
| `created_at` | timestamp | 创建时间 |
| `updated_at` | timestamp | 更新时间 |

#### 数据量统计

| 表 | 记录数 | 说明 |
|----|--------|------|
| `vector_index` | 30,730 | 所有记忆向量（唯一有数据的表） |
| `context_nodes` | 0 | 空 |
| `session_archives` | 0 | 空 |
| `relation_edges` | 0 | 空 |
| `dream_runs` | 2 | 旧版测试记录 |
| `dream_signals` | 160 | Dream 信号 |
| `dream_staged_candidates` | 20 | Dream 候选 |

#### user 4f54b7c9 的记忆分布示例

| category | level | count | total_chars |
|----------|-------|-------|-------------|
| entity | abstract | 105 | 8,830 |
| entity | content | 105 | 26,790 |
| event | abstract | 208 | 19,219 |
| event | content | 208 | 67,430 |
| pattern | abstract | 9 | 771 |
| pattern | content | 9 | 2,824 |
| preference | abstract | 127 | 9,138 |
| preference | content | 127 | 25,130 |
| profile | abstract | 1 | 52 |
| profile | content | 1 | 116 |
| session_archive | abstract | 42 | 8,434 |
| session_archive | content | 42 | 60,727 |
| session_summary | abstract | 45 | 9,127 |
| session_summary | content | 45 | 56,386 |

## 8. 执行时间线

| 时间 | Step | 状态 |
|------|------|------|
| 13:16 | Step 3: Ingest | ✅ 20,432 条记忆写入 |
| 14:45 | Step 4: before-dream QA（旧版） | ❌ 11题后中断（0%准确率，55%超时） |
| 发现 bug | 诊断 owner_space 跨用户泄漏 | 定位到 seed_retriever + build_context |
| ~15:30 | OpenCode 修复 bug | 2 文件修改（memory_service.py + seed_retriever.py） |
| 16:00 | 重启 compose 服务 | 新代码生效 |
| 16:08 | Step 4: before-dream-v2 QA | ✅ 36题完成（61.1%） |
| ~16:30 | Step 5: DeepDream | ✅ 36题完成（79 URI，avg 77s/题） |
| ~17:00 | Step 6: after-dream QA | ✅ 36题完成（61.1%） |