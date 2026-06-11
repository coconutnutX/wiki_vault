---
name: ogmemory-deepdream-longmemeval-test-plan
description: 用 LongMemEval 数据验证 deepdream 流程的测试计划
metadata:
  type: project
---

# oG-Memory DeepDream LongMemEval 测试计划

验证 deepdream（process_correction + process_consolidation）在真实多 session 数据上的完整流程。

## 选定数据集

**`longmemeval_s_dream_openclaw_smoke_9.json`**（最小可用集）

| 指标           | 数值                                                          |
| ------------ | ----------------------------------------------------------- |
| 题目数          | 9                                                           |
| 每题 session 数 | 38-55（平均 48.7）                                              |
| 题型           | multi-session, temporal-reasoning, knowledge-update（每型 3 条） |

> ⚠️ `longmemeval_s_dream_core.json` 是 Git LFS 指针（171MB），本地未下载，暂不可用。

其他可用数据集（稳定后逐步升级）：

| 数据集 | 题目 | 唯一 session | 用途 |
|--------|------|-------------|------|
| openclaw_smoke_9 | 9 | 440 | 最小冒烟 ← 当前 |
| dream_18 | 18 | 855 | 低成本主测 |
| dream_smoke_30 | 30 | 1442 | 中规模冒烟 |
| dream_36 | 36 | 1638 | 正式主测 |

## 与 LoCoMo 测试方式对比

LoCoMo 和 LongMemEval 的测试流程有本质差异，理解这些差异有助于正确实施 LongMemEval 测试。

### 流程对比表

| 步骤 | LoCoMo 方式 | LongMemEval 方式 | 与 LoCoMo 区别 |
|------|-------------|------------------|----------------|
| **数据规模** | 4 个 session（Caroline + Melanie） | 9 题 × 48.7 sessions/题 = ~440 sessions | **远大于 LoCoMo** |
| **user_id** | 单一 `ogmem-small` | **每题独立** `lme-{run_id}-{qid}` | **核心差异：按题隔离** |
| **account_id** | `acct-demo` | `longmemeval` | 不同 account |
| **注入方式** | 直接调用 `/ingest_batch` + `/after_turn` | 用 `longmemeval_openclaw_runner.py` 逐题注入 | **已有 runner 脚本，无需手写** |
| **等待抽取** | 120 秒轮询 | runner 内置 `post_ingest_wait=30s` + poll | 更智能 |
| **DeepDream** | 单次调用（整个用户） | **每题单独调用**（9 次） | **多次调用而非单次** |
| **产出验证** | 查看 dream category 记忆数和内容 | 每题独立的 dream 记忆 | 需按 user_id 分组查询 |
| **评测方式** | 手动 compose + 验证 | longmemeval 框架自动评测 | 有自动化评测流程 |

### 关键差异：user_id 隔离设计

**LoCoMo**: 单一 user_id，所有 session 属于一个用户，dream 一次调用处理全部记忆。

**LongMemEval**: 每条题目 = 一个独立虚拟用户，必须按题隔离 user_id。

原因：
1. **评测语义正确性**: compose 回答时只应从该题的 haystack_sessions 记忆中召回，不应混入其他题的 session
2. **防止跨题目干扰**: 共享 user_id 会导致 compose 召回无关记忆，降低评测准确率
3. **dream 输入独立**: 每题的 dream 只处理该题的记忆，产出不影响其他题

### 简化版测试流程示例

```bash
# Phase 1: 数据注入（复用 runner）
cd /data/Workspace2/repos/longmemeval
python -m longmemeval_test.cli run configs/dream-s-smoke-ogmemory-before.toml --only health_check,run

# Phase 2: DeepDream（新脚本）
truncate -s 0 /tmp/ogmem_stdout.log
python scripts/run_deepdream.py \
  --data-file data/longmemeval_s_dream_openclaw_smoke_9.json \
  --api-url http://127.0.0.1:8090 \
  --account-id longmemeval \
  --user-prefix dream-before \
  --run-id <from runner output>

# Phase 3: 验证产出
PGPASSWORD=ogmem123 psql -h 127.0.0.1 -U ogmem -d ogmemory -c "
SET app.account_id = 'longmemeval';
SELECT uri, category, abstract FROM context_nodes WHERE category = 'dream';
"
```

## user_id 隔离问题（关键设计决策）

LongMemEval 的语义设计：**每条题目 = 一个独立虚拟用户**。`haystack_sessions` 是该虚拟用户的全部对话历史。compose 回答时只应从该虚拟用户的记忆中召回——题目间不应互相干扰。

现有 `longmemeval_openclaw_runner.py` 的做法是每题分配独立 `user_id`（`{user_prefix}-{run_id}-{qid}`），这是正确的评测隔离。

### 方案对比

| | 方案 A：按题目隔离注入 | 方案 B：共享 user_id 注入 |
|---|---|---|
| user_id | 每题独立：`lme-{run_id}-{qid}` | 全局共享：`dream-lme-smoke9` |
| dream 输入规模 | 每题 30-70 条记忆（较小） | 9 题合并 ~200+ 条记忆（较大） |
| 评测隔离 | ✅ 正确：题目间无干扰 | ❌ compose 会召回跨题目干扰 |
| dream 对 compose 的影响 | 每题内 dream 产出只影响本题 compose | 全局 dream 产出影响所有题 compose |
| dream 产出质量 | 输入少可能产出少或无产出 | 输入充足，dream 更容易发现 consolidation/correction |

### 选择：方案 A（按题目隔离注入）

理由：
1. 评测语义正确性是最重要的。LongMemEval 的标准要求题目间独立
2. compose 也用同一 user_id，共享 user_id 会导致召回跨题目的无关记忆，降低评测准确率
3. dream 的核心目的是验证流程跑通，不是产出数量最大化
4. 单题 30-70 条记忆足够让 dream 跑起来（wiki e2e guide 用 locomo 4 session 注入，产出了 5 条 dream）
5. dream 对每题分别调用，9 次调用覆盖更多场景，验证更全面

### 实施细节

- 注入：复用现有 runner 的按题维度注入逻辑（每题独立 user_id）
- dream：对每题的 user_id 分别调用 `/api/v1/deepdream`
- compose：用对应题目 user_id 调 compose，评测隔离
- 9 次 dream 调用串行执行（每次约 2-5 分钟，总计约 20-45 分钟）

## 测试流程

### Phase 1: 数据注入（Ingest） — 复用 runner

1. **清空数据库**
   ```sql
   TRUNCATE TABLE session_archives, context_nodes, vector_index, outbox_events, dream_recalls, relation_edges CASCADE;
   ```

2. **用 longmemeval 框架注入**
   - 使用 `dream-s-smoke-ogmemory-before.toml` 配置
   - 只跑 `health_check,run` 步骤（跳过 official_eval 和 stats）
   - runner 会为每题分配独立 user_id 并逐 session 调 `after_turn`
   ```bash
   cd /data/Workspace2/repos/longmemeval
   python -m longmemeval_test.cli run configs/dream-s-smoke-ogmemory-before.toml --only health_check,run --skip stats
   ```

3. **等待后台抽取完成**
   - runner 已内置 `post_ingest_wait=30s` 和 `wait_policy=poll`
   - 注入完成后额外等待 60-120 秒确保抽取完成
   - 验证：context_nodes 中记忆数足够

### Phase 2: DeepDream 调用 — 新脚本

1. **新脚本** `scripts/run_deepdream.py`
   - 读取与注入相同的 dream 数据集
   - 为每条题目构造与 runner 相同格式的 user_id（复用 runner 的 `user_prefix-run_id-qid` 逻辑）
   - 逐题调用 `/api/v1/deepdream`
   - 参数：`--data-file`, `--api-url`, `--account-id`, `--user-prefix`, `--run-id`

2. **调用示例**
   ```bash
   cd /data/Workspace2/repos/longmemeval
   truncate -s 0 /tmp/ogmem_stdout.log
   python scripts/run_deepdream.py \
     --data-file data/longmemeval_s_dream_openclaw_smoke_9.json \
     --api-url http://127.0.0.1:8090 \
     --account-id longmemeval \
     --user-prefix dream-before \
     --run-id <from runner output>
   ```

3. **等待所有 dream 完成**（每题约 2-5 分钟，9 题 ≈ 20-45 分钟）

### Phase 3: 验证 Dream 产出

1. **日志追踪** — 检查 `/tmp/ogmem_stdout.log`：
   - 每题 `deepdream: entry` 和 `deepdream: completed` 出现 9 次
   - `DreamOutput` 产出条目（topic, sub_type, provenance_ids）
   - `process_correction` 或 `process_consolidation` 被调用
   - `write_memory result: target_uri=... action=create` 写入成功

2. **数据库验证**
   ```sql
   SET app.account_id = 'longmemeval';

   -- Dream 列表（每题的 user_id 下各自产出）
   SELECT uri, category, abstract FROM context_nodes WHERE category = 'dream';

   -- Dream 详细（含 metadata 中 tool_stats/sub_type/correct_fact/confidence）
   SELECT uri, abstract, content, metadata FROM context_nodes WHERE category = 'dream';

   -- 所有记忆类别统计
   SELECT category, COUNT(*) FROM context_nodes GROUP BY category ORDER BY category;
   ```

3. **重点验证字段**
   - `metadata` JSON 中是否有 `tool_stats` → `sub_type`, `confidence`, `correct_fact`（如为 correction）
   - `provenance_ids` 是否从 LLM 指定的 `affected_uris`/`source_uris` 解析而来
   - `routing_key` 是否包含 correction/consolidation 类型
   - dream_type 是 `"correction"` 或 `"consolidation"`（不再是 `"promotion"`）

### Phase 4: QA 评估（流程跑通后）

用 longmemeval 框架做 before/after 对比：

```bash
# 1. Before dream: 注入 + compose + 评测（已有 runner 配置）
python -m longmemeval_test.cli run configs/dream-s-smoke-ogmemory-before.toml

# 2. Run deepdream per question (新脚本)
python scripts/run_deepdream.py ...

# 3. After dream: 用新 user_prefix 重跑 compose + 评测
python -m longmemeval_test.cli run configs/dream-s-smoke-ogmemory-after.toml
```

比较 before/after 的 accuracy、hit_count、retrieved_chars。

> **重要**：before/after 配置使用不同 `user_prefix`（`dream-before` / `dream-after`），但 dream 是对 before 的 user_id 调用的，所以 after 的 compose 必须复用 before 的 user_id 才能看到 dream 产出。这里需要确认 after 配置的 user_id 是否正确。

## 需要新建的脚本

### LongMemEval 仓库脚本能否复用？

**不需要修改 LongMemEval 官方脚本**，因为：

1. **LongMemEval 仓库 (`/data/Workspace2/repos/LongMemEval`)** 提供的是：
   - 数据集（`longmemeval_s_cleaned.json`）
   - 检索脚本（`run_retrieval.py`）
   - 生成脚本（`run_generation.py`）
   - 评测脚本（`evaluate_qa.py`）
   
   这些是**通用的评测框架**，与具体 memory 系统无关。

2. **oG-Memory 适配层** (`longmemeval_test/` 模块) 是独立代码，专门用于：
   - 将 LongMemEval 数据注入 oG-Memory
   - 用 oG-Memory 的 compose API 回答问题
   - 对接 LongMemEval 的评测脚本

3. **DeepDream 调用** 是 oG-Memory 的特定功能，LongMemEval 原仓库没有对应脚本，需要自己写。

### 可直接复用的部分

| 脚本/配置 | 用途 | 复用情况 |
|----------|------|----------|
| `longmemeval_openclaw_runner.py` | 注入 + compose + 评测 | ✅ 已实现，直接用 |
| `configs/dream-s-smoke-ogmemory-before.toml` | before dream 配置 | ✅ 已有 |
| `configs/dream-s-smoke-ogmemory-after.toml` | after dream 配置 | ✅ 已有 |
| 数据集 `longmemeval_s_dream_openclaw_smoke_9.json` | 最小冒烟数据 | ✅ 已有 |

### 需要新建的部分

| 脚本 | 用途 | 状态 |
|------|------|------|
| `scripts/run_deepdream.py` | 逐题调用 deepdream API | ❌ 待写（接口已设计好） |

### `scripts/run_deepdream.py`

职责：对每题的 user_id 调 deepdream API

参数：
- `--data-file`：dream 数据集 JSON
- `--api-url`：oGMemory URL（默认 `http://127.0.0.1:8090`）
- `--account-id`：默认 `longmemeval`
- `--user-prefix`：默认 `lme`（与 runner 一致）
- `--run-id`：必须与注入时使用的 run_id 一致
- `--http-timeout`：单次 dream 调用超时（默认 300s）
- `--limit`：只处理前 N 条题目
- `--question-type`：只处理指定题型

核心逻辑：
```python
for entry in data:
    qid = entry["question_id"]
    user_id = f"{user_prefix}-{run_id}-{qid}"
    payload = {
        "userId": user_id,
        "accountId": account_id,
        "sessionId": f"{user_id}-dream",
    }
    result = POST /api/v1/deepdream
    print(f"qid={qid} user={user_id}: status={result['status']} created_uris={result['created_uris']}")
```

## 预期产出

- 每题 dream 产出 1-3 条 correction/consolidation 记忆（9 题总计 ~10-20 条）
- 每条 dream 的 metadata 包含完整的 tool_stats（sub_type, confidence, 正确的 provenance_ids）
- 验证 process_correction 和 process_consolidation 都被 ReAct loop 正确调用
- 验证 dream 产出能被后续 compose 正确召回

## 风险与注意事项

1. **user_id 一致性**：注入和 dream 必须用同一个 user_id 格式，compose 评测也必须一致
2. **account_id RLS**：所有操作使用 `longmemeval` account，psql 查询前必须 `SET app.account_id`
3. **LLM 超时/rate limit**：9 次串行 dream 调用，代理网关不稳定可能导致某次超时
4. **抽取等待**：注入完成后需等待抽取，时间不足会导致 dream 输入记忆不够
5. **run_id 同步**：注入 runner 和 dream 脚本的 run_id 必须一致，否则 user_id 不匹配

## 总结

| 问题 | 回答 |
|------|------|
| 能否类似地测 LongMemEval？ | ✅ 可以，但流程更复杂（多题独立 user_id） |
| 与 LoCoMo 的区别？ | 数据规模大、按题隔离 user_id、多次 dream 调用、有自动化评测流程 |
| LongMemEval 仓库脚本能否直接复用？ | ✅ 评测脚本可复用，注入/评测有 runner，deepdream 需新建脚本 |
| 需要做什么修改？ | **无需修改原仓库**，只需新建 `run_deepdream.py`（本文档已设计接口） |
| user_id 隔离的重要性？ | **核心差异**：共享 user_id 会导致评测语义错误，compose 会召回跨题目干扰 |

**建议**：直接按照本文档的测试流程执行，runner 和配置已准备就绪，只需实现 `run_deepdream.py` 脚本。

## 相关文档

- [[ogmemory-deep-dream-e2e-test-guide]] — locomo 数据的端到端测试指南
- [[ogmemory-deep-dream-framework]] — deepdream 框架设计