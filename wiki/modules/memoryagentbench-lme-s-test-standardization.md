---
name: memoryagentbench-lme-s-test-standardization
description: MemoryAgentBench LongMemEval S* 测试流程规范：对接 oG-Memory（固定 user_id owner_space、chunk 级进度恢复、--contexts 选择运行、原子写入、日期前缀输出目录）
metadata:
  type: reference
---

# MemoryAgentBench LongMemEval S* 测试流程规范

基于 [[longmemeval-test-standardization]] 的经验教训和 [[memoryagentbench-lme-s-run-20260624]] 的踩坑实战，适配 MemoryAgentBench 框架对 LongMemEval S* 数据集的测试流程。

**2026-06-25 重构版本**：owner_space 从随机 run_id 改为固定 user_id；注入恢复从累计 query 推算改为 chunk 级进度文件；新增 `--contexts` 参数支持选择运行；新增日期前缀输出目录规范。

## 1. 对接方案：Chunk 模式 + 固定 owner_space

### 数据集结构

LongMemEval S* 有 **5 个大 context**（每 ~1.6M 字符），每个 context 有 **60 个 QA pair**，共 300 个 query。不是 60 个 sub-context。

`max_test_samples: 5` → 5 个 context，每个 context ~88-95 chunks。

### Chunk 模式注入

MemoryAgentBench 采用 **chunk 模式** 注入：
- 每个长 context 按 `agent_chunk_size`（4096）切分成 chunks
- 每个 chunk 调一次 `after_turn`
- 每个 context 对应一个固定的 `user_id`（来自 `ogmemory_user_ids` 列表）
- session-id 格式：`{user_id}-chunk-{idx:04d}`

### 固定 owner_space（核心变更）

**旧方案（已弃用）**：`user_id = mabench-{random_run_id}-ctx{N}`，每次初始化生成随机 run_id，导致 after-dream compose 无法找到 before-dream 注入的数据。

**新方案**：每个 context 对应 YAML 中明确指定的 `ogmemory_user_ids`，注入、dream、compose 全用同一个 user_id：

```yaml
ogmemory_user_ids:
  - user_0    # context 0 的 owner_space = "user:user_0"
  - user_1    # context 1 的 owner_space = "user:user_1"
  - user_2
  - user_3
  - user_4
```

- 注入：`userId = user_0` → 数据落在 `user:user_0` owner_space
- Dream：`userId = user_0` → dream 数据也落在 `user:user_0`
- compose：`userId = user_0` → 能看到注入 + dream 全部数据
- **不再需要 `ogmemory_run_id` / `ogmemory_run_id_map` / `ogmemory_user_prefix`**（全部删除）

### 与 wiki runner 的差异

| 方面         | Wiki runner（session 模式）          | MemoryAgentBench（chunk 模式）  |
| ---------- | -------------------------------- | --------------------------- |
| 注入单位       | 按 session 边界                     | 按 chunk（4096字）              |
| user_id    | 动态 run_id 或固定                    | **固定 user_id（YAML 指定）**     |
| session_id | `{user_id}-hist-{idx:04d}-{sid}` | `{user_id}-chunk-{idx:04d}` |
| 时间戳        | 原始 session 时间                    | 当前时间                        |
| 抽取触发       | 每个 session 触发                    | chunk 累积达到 threshold 触发     |

## 2. 模型配置

### Reader vs Judge

| 角色 | 用途 | 配置位置 | 当前值 |
|------|------|----------|--------|
| **Reader** | compose 后回答问题 | agent_config.yaml 的 `model` + `OPENAI_API_KEY/BASE_URL` env | `GLM-5` |
| **Judge** | LLM-as-judge 评测 | `llm_based_eval/longmem_qa_evaluate.py` 的 `--judge_model` | `GLM-5`（当前 key 无 gpt-4o） |
| **oG-Memory compose/dream/extract** | 记忆抽取和组装 | oG-Memory 的 `ogmem.yaml` | 配置中定义 |

### 配置文件

| 配置 | 路径 | 用途 |
|------|------|------|
| before-dream | `configs/agent_conf/RAG_Agents/glm-5/Structure_rag_glm-5-ogmemory.yaml` | 注入 + query |
| after-dream | `configs/agent_conf/RAG_Agents/glm-5/Structure_rag_glm-5-ogmemory-after-dream.yaml` | query only（不注入） |
| dataset | `configs/data_conf/Accurate_Retrieval/LongMemEval/Longmemeval_s_star.yaml` | S* 数据集配置 |
| smoke test | `configs/data_conf/Accurate_Retrieval/LongMemEval/Longmemeval_s_star_smoke.yaml` | 1 sample 快速验证 |

注意：after-dream 配置和 before-dream 完全一致（相同的 `ogmemory_user_ids`），只改 `output_dir` 和 `rag_retrieved_dir`。不再需要单独的 after-dream-2 配置。

## 3. agent_config.yaml 关键字段

```yaml
agent_name: Structure_rag_ogmemory
model: GLM-5                    # Reader 模型
temperature: 0.0
input_length_limit: 10000000
buffer_length: 1000
output_dir: ./outputs/glm-5-ogmemory          # ← 每次跑前改日期前缀
rag_retrieved_dir: ./outputs/rag_retrieved     # ← 每次跑前改日期前缀

ogmemory_api_url: http://127.0.0.1:8090
ogmemory_account_id: longmemeval
ogmemory_agent_id: openclaw-longmemeval
ogmemory_token_budget: 128000

# 固定 user_ids：每个 context 对应确定的 owner_space
# 注入、dream、compose 全用同一个 user_id per context
ogmemory_user_ids:              # 必须指定，无 fallback
  - user_0
  - user_1
  - user_2
  - user_3
  - user_4

ogmemory_wait_timeout: 120.0    # 等待抽取完成的超时
ogmemory_wait_interval: 5.0     # poll 间隔
ogmemory_post_inject_wait: 30.0 # 注入后固定等待（后台抽取时间）
ogmemory_inject_throttle: 1.5   # chunk 注入间隔（防止 LLM gateway 过载）

retrieve_num: 100
agent_chunk_size: 4096
```

**已删除的字段**：`ogmemory_run_id`、`ogmemory_run_id_map`、`ogmemory_user_prefix`

### `output_dir` 和 `rag_retrieved_dir` 说明

- `output_dir`：存放评测结果 JSON（`*_results.json`）和 agent 实验目录
- `rag_retrieved_dir`：存放 compose 返回的检索上下文 JSON（每个 query 一个文件）
- 两者必须用**相同的日期前缀**，方便查找同一轮实验的所有中间结果
- `rag_retrieved_dir` 是新增字段；如果 YAML 中没有指定，代码默认 `./outputs/rag_retrieved`

## 4. 输出目录命名规范（新增）

每次跑新实验前，必须把 `output_dir` 和 `rag_retrieved_dir` 加上日期前缀，方便找到历史结果。

### 命名规则

| 场景 | `output_dir` | `rag_retrieved_dir` |
|------|-------------|---------------------|
| 当天第 1 次 | `./outputs/20260625-glm-5-ogmemory` | `./outputs/20260625-rag_retrieved` |
| 当天第 2 次 | `./outputs/20260625-2-glm-5-ogmemory` | `./outputs/20260625-2-rag_retrieved` |
| 当天第 3 次 | `./outputs/20260625-3-glm-5-ogmemory` | `./outputs/20260625-3-rag_retrieved` |
| after-dream 第 1 次 | `./outputs/20260625-glm-5-ogmemory-after-dream` | `./outputs/20260625-rag_retrieved` |
| after-dream 第 2 次 | `./outputs/20260625-2-glm-5-ogmemory-after-dream` | `./outputs/20260625-2-rag_retrieved` |

**关键原则**：
- 同一轮 before-dream 和 after-dream 共享**同一个** `rag_retrieved_dir`（检索结果是同一批）
- `output_dir` 区分 before/after-dream（`glm-5-ogmemory` vs `glm-5-ogmemory-after-dream`）
- 同一天多次跑时，序号 `2`, `3`... 加在日期后面（`20260625-2-`），不影响 `rag_retrieved` 内部按 `agent_name` 分层的结构

### 修改方式

跑之前用 `sed` 一次性改 YAML：

```bash
DATE_PREFIX="20260625"
AGENT_YAML="configs/agent_conf/RAG_Agents/glm-5/Structure_rag_glm-5-ogmemory.yaml"
AFTER_YAML="configs/agent_conf/RAG_Agents/glm-5/Structure_rag_glm-5-ogmemory-after-dream.yaml"

sed -i "s|output_dir: ./outputs/.*|output_dir: ./outputs/${DATE_PREFIX}-glm-5-ogmemory|" ${AGENT_YAML}
sed -i "s|rag_retrieved_dir: ./outputs/.*|rag_retrieved_dir: ./outputs/${DATE_PREFIX}-rag_retrieved|" ${AGENT_YAML}

sed -i "s|output_dir: ./outputs/.*|output_dir: ./outputs/${DATE_PREFIX}-glm-5-ogmemory-after-dream|" ${AFTER_YAML}
sed -i "s|rag_retrieved_dir: ./outputs/.*|rag_retrieved_dir: ./outputs/${DATE_PREFIX}-rag_retrieved|" ${AFTER_YAML}
```

### 目录结构示例

```
outputs/
├── 20260625-glm-5-ogmemory/
│   └── Accurate_Retrieval/
│       └── longmemeval_s*_unknown_in400000_..._results.json
├── 20260625-glm-5-ogmemory-after-dream/
│   └── Accurate_Retrieval/
│       └── longmemeval_s*_unknown_in400000_..._results.json
├── 20260625-rag_retrieved/
│   └── Structure_rag_ogmemory/
│       └── k_100/longmemeval_s*/chunksize_4096/
│           ├── query_0_context_0.json
│           ├── query_1_context_0.json
│           └── ...
├── 20260625-2-glm-5-ogmemory/     ← 当天第 2 次跑
├── 20260625-2-rag_retrieved/
└── ...
```

### 为什么要改代码

`rag_retrieved` 路径原来硬编码在 `agent.py` 中（`./outputs/rag_retrieved/{agent_name}/...`），每次跑都写入同一个目录，历史结果会被覆盖。现在改为从 YAML `rag_retrieved_dir` 读取，可以每次指定带日期前缀的目录，保留历史。

`output_dir` 原来也是 YAML 字段，只需每次跑前改 YAML 即可。

### 注入流程

```python
# 每个 chunk → after_turn
POST /api/v1/after_turn
{
  "accountId": "longmemeval",
  "userId": "user_0",                              # 固定 user_id
  "agentId": "openclaw-longmemeval",
  "sessionId": "user_0-chunk-0001",                 # {user_id}-chunk-{idx}
  "messages": [
    {"role": "user", "content": "<formatted chunk>", "created_at": "..."},
    {"role": "assistant", "content": "I have memorized this content.", "created_at": "..."}
  ],
  "prePromptMessageCount": 0
}
```

### 查询流程

```python
# compose 检索 + reader 回答
POST /api/v1/compose
{
  "accountId": "longmemeval",
  "userId": "user_0",                              # 同注入用相同 user_id
  "agentId": "openclaw-longmemeval",
  "sessionId": "user_0-question",
  "query": "<question>",
  "messages": [{"role": "user", "content": "<question>", "created_at": "..."}],
  "tokenBudget": 128000
}
```

compose 返回 → 提取 `identityContext`/`episodicContext`/`dreamContext`/`sessionContext`/`taskContext`/`retrievedEvidence` slots → 拼接 → 发给 reader LLM

### 注入后等待机制

MemoryAgentBench 在 `_memorize_context_chunks` 结束后会：
1. 固定等待 `ogmemory_post_inject_wait`（30s）——后台异步抽取需要时间
2. Poll compose 直到 `hit_count >= 1` 或 `retrieved_chars >= 100`
3. 超时后继续执行（warn 状态）

## 5. 中断恢复机制（新增）

### 注入进度文件

每个 context 注入时写 `agents/exp_N/inject_progress.json`，记录 chunk 级进度：

```json
{
  "context_index": 0,
  "chunk_idx": 87,
  "total_chunks": 88,
  "user_id": "user_0"
}
```

- `chunk_idx == total_chunks - 1` → 注入已完成，下次 skip 注入
- `chunk_idx < total_chunks - 1` → 注入中断，下次从 `chunk_idx + 1` 续注
- 文件不存在 → 完整注入

### Query 恢复

results.json 中每个 query 记录 `context_index` 和 `qa_pair_id`，恢复时按 `(context_index, qa_pair_id)` 精确 skip：

- `completed_map` = `{ctx: set(qa_pair_ids)}` — 从 saved results 构建
- 某 context 的所有 qa_pair_id 都在 completed_map → 整个 context skip
- 否则 → 只 skip 已完成的 query，继续跑剩余 query

### 原子写入

`save_results_to_file` 改为先写 `.tmp` 再 `os.replace`，防止写入一半中断导致文件损坏。

### 旧恢复逻辑（已弃用）

旧方案 `_calculate_last_completed_context_id` 用累计 query 数推算 context 进度，在 context 非完全完成时会出错（如 ctx3 只跑 30/60 个 query，误判为 ctx3 已完成）。

## 6. `--contexts` 参数（新增）

支持指定运行哪些 context，方便用单个 context 快速验证：

```bash
# 只跑 context 0（快速验证）
python main.py --agent_config ... --dataset_config ... --contexts 0

# 跑 context 1,3,4（补充之前没跑的）
python main.py --agent_config ... --dataset_config ... --contexts 1,3,4

# 跑全部（不指定 --contexts）
python main.py --agent_config ... --dataset_config ...
```

**注意**：`--contexts` 指定的是数据集中的原始 index（0-4），不是 enumerate 的顺序 index。

## 7. JSON 增强记录（新增）

results.json 每个 query 追加：

| 字段 | 含义 |
|------|------|
| `context_index` | 该 query 属于哪个 context |
| `compose_input_len` | compose 返回的记忆字符长度 |
| `compose_raw` | compose 完整返回（含 stats、各 slot） |

`compose_raw` 较大但只对 oG-Memory agent 追加，用于事后分析 compose 返回质量。

## 8. 数据库隔离

每轮测试用**新数据库**：

```bash
DB_VER="v10"
echo "123123" | sudo -S -u postgres psql -c "CREATE DATABASE ogmemory_${DB_VER} OWNER ogmem;"
echo "123123" | sudo -S -u postgres psql -d ogmemory_${DB_VER} -c "CREATE EXTENSION IF NOT EXISTS vector;"
echo "123123" | sudo -S -u postgres psql -c "ALTER ROLE ogmem WITH BYPASSRLS;"
```

修改 `oG-Memory/config/ogmem.yaml` 两处 `connection_string` 的 `dbname` → 重启服务。

**⚠️ Before/After dream 用同一数据库**——dream 是对同一记忆的 consolidation。

**⚠️ 绝不修改旧数据库**——如果要跑新测试，必须创建新数据库（如 v10），不能 wipe v9。

## 9. 完整流程

```bash
DATE_PREFIX="20260625"
DB_VER="v10"

conda activate py11

# ---- 前置：embedder 服务 ----
cd /data/Workspace2/locomo-test
HF_ENDPOINT=https://hf-mirror.com HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1 \
  python -u deploy_model.py --model BAAI/bge-small-en-v1.5 --port 8345 > /tmp/embedder.log 2>&1 < /dev/null &
sleep 10

# ---- Step 1: 创建数据库 ----
echo "123123" | sudo -S -u postgres psql -c "CREATE DATABASE ogmemory_${DB_VER} OWNER ogmem;"
echo "123123" | sudo -S -u postgres psql -d ogmemory_${DB_VER} -c "CREATE EXTENSION IF NOT EXISTS vector;"
echo "123123" | sudo -S -u postgres psql -c "ALTER ROLE ogmem WITH BYPASSRLS;"

cd /data/Workspace2/oG-Memory
sed -i "s/dbname=ogmemory_v[0-9]/dbname=ogmemory_${DB_VER}/g" config/ogmem.yaml

# ---- Step 2: 启动 oG-Memory 服务 ----
fuser -k 8090/tcp 2>/dev/null; sleep 3
setsid env HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1 \
  python -u server/app.py > /tmp/ogmem_stdout.log 2>&1 < /dev/null &
sleep 20
curl -s http://127.0.0.1:8090/api/v1/health

# ---- Step 3: 修改 YAML 输出目录（加日期前缀） ----
cd /data/Workspace2/repos/MemoryAgentBench

AGENT_YAML="configs/agent_conf/RAG_Agents/glm-5/Structure_rag_glm-5-ogmemory.yaml"
AFTER_YAML="configs/agent_conf/RAG_Agents/glm-5/Structure_rag_glm-5-ogmemory-after-dream.yaml"

sed -i "s|output_dir: ./outputs/.*|output_dir: ./outputs/${DATE_PREFIX}-glm-5-ogmemory|" ${AGENT_YAML}
sed -i "s|rag_retrieved_dir: ./outputs/.*|rag_retrieved_dir: ./outputs/${DATE_PREFIX}-rag_retrieved|" ${AGENT_YAML}

sed -i "s|output_dir: ./outputs/.*|output_dir: ./outputs/${DATE_PREFIX}-glm-5-ogmemory-after-dream|" ${AFTER_YAML}
sed -i "s|rag_retrieved_dir: ./outputs/.*|rag_retrieved_dir: ./outputs/${DATE_PREFIX}-rag_retrieved|" ${AFTER_YAML}

# ---- Step 4: Before-dream 评测 ----
export OPENAI_API_KEY="sk--hAux09V3aXYBMsaZG_k2w"
export OPENAI_BASE_URL="http://113.46.219.251:8080/v1"
export HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1 PYTHONUNBUFFERED=1

# 快速验证：先跑 context 0
python -u main.py \
  --agent_config ${AGENT_YAML} \
  --dataset_config configs/data_conf/Accurate_Retrieval/LongMemEval/Longmemeval_s_star.yaml \
  --contexts 0

# 验证通过后跑剩余 context
python -u main.py \
  --agent_config ${AGENT_YAML} \
  --dataset_config configs/data_conf/Accurate_Retrieval/LongMemEval/Longmemeval_s_star.yaml \
  --contexts 1,2,3,4

# 或直接跑全部（不带 --contexts）
python -u main.py \
  --agent_config ${AGENT_YAML} \
  --dataset_config configs/data_conf/Accurate_Retrieval/LongMemEval/Longmemeval_s_star.yaml \
  --force

# ---- Step 5: DeepDream ----
truncate -s 0 /tmp/ogmem_stdout.log

# 对每个 context 跑 Dream（必须传 userId）
for i in 0 1 2 3 4; do
  curl -s -X POST http://127.0.0.1:8090/api/v1/deepdream \
    -H 'Content-Type: application/json' \
    -d "{\"accountId\":\"longmemeval\",\"userId\":\"user_${i}\",\"agentId\":\"openclaw-longmemeval\"}"
  sleep 5
done

cp /tmp/ogmem_stdout.log ./outputs/${DATE_PREFIX}-rag_retrieved/dream_stdout.log

# ---- Step 6: After-dream 评测 ----
python -u main.py \
  --agent_config ${AFTER_YAML} \
  --dataset_config configs/data_conf/Accurate_Retrieval/LongMemEval/Longmemeval_s_star.yaml

# ---- Step 7: LLM-as-judge 评测 ----
python llm_based_eval/longmem_qa_evaluate.py \
  --evaluated_method glm-5-ogmemory \
  --dataset longmemeval_s* \
  --judge_model GLM-5 \
  --judge_api_key $OPENAI_API_KEY \
  --judge_api_base $OPENAI_BASE_URL \
  --output_dir ./outputs/${DATE_PREFIX}-glm-5-ogmemory/eval_results/

python llm_based_eval/longmem_qa_evaluate.py \
  --evaluated_method glm-5-ogmemory-after-dream \
  --dataset longmemeval_s* \
  --judge_model GLM-5 \
  --judge_api_key $OPENAI_API_KEY \
  --judge_api_base $OPENAI_BASE_URL \
  --output_dir ./outputs/${DATE_PREFIX}-glm-5-ogmemory-after-dream/eval_results/

# ---- Step 8: 保存日志 ----
cp /tmp/ogmem_stdout.log ./outputs/${DATE_PREFIX}-glm-5-ogmemory/service_stdout.log
```

### 中断后恢复

中断后重新运行**不带 `--force`**，恢复逻辑自动生效：
- 注入进度：从 `inject_progress.json` 判断是否需要续注
- Query 进度：从 `results.json` 构建 `completed_map`，精确 skip 已完成的 `qa_pair_id`
- 只跑特定 context：用 `--contexts` 指定

```bash
# 中断后续跑 ctx3,4（假设 ctx0-2 已完成）
python -u main.py \
  --agent_config ... --dataset_config ... \
  --contexts 3,4
# 不带 --force，会自动 skip 已完成的 query
```

### 同一天多次跑

如果同一天需要重跑（比如换 DB 版本、换参数），递增序号：

```bash
DATE_PREFIX="20260625-2"

# 重新 sed 替换 YAML（序号变了）
sed -i "s|output_dir: ./outputs/.*|output_dir: ./outputs/${DATE_PREFIX}-glm-5-ogmemory|" ${AGENT_YAML}
sed -i "s|rag_retrieved_dir: ./outputs/.*|rag_retrieved_dir: ./outputs/${DATE_PREFIX}-rag_retrieved|" ${AGENT_YAML}

# after-dream 同理
sed -i "s|output_dir: ./outputs/.*|output_dir: ./outputs/${DATE_PREFIX}-glm-5-ogmemory-after-dream|" ${AFTER_YAML}
sed -i "s|rag_retrieved_dir: ./outputs/.*|rag_retrieved_dir: ./outputs/${DATE_PREFIX}-rag_retrieved|" ${AFTER_YAML}
```

## 10. 踩坑清单

| 问题 | 原因 | 解决 |
|------|------|------|
| after_turn 500 RLS 错误 | `ensure_schema()` 每次重开 FORCE RLS | `ALTER ROLE ogmem WITH BYPASSRLS` |
| compose 返回空记忆 | 注入后立即查询，抽取未完成 | 固定等待 + poll compose |
| embedder 连 HuggingFace 挂死 | `nltk.download()` / SentenceTransformer 联网 | `HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1` |
| reader/judge 模型混淆 | 一个 `model` 用于 reader，judge 在 eval 脚本 | 文档明确区分，judge 可配置 |
| reader 未配 env | `_create_oai_client()` 用 `OpenAI()` 无参数 | 必须 export `OPENAI_API_KEY` / `OPENAI_BASE_URL` |
| OOM 崩溃 | 大 embedder + oG-Memory + extraction 累积 | 换小 embedder（bge-small-en-v1.5, 384 dim） |
| LLM gateway 过载（503→hang） | 88 chunk 同时触发 extraction | `ogmemory_inject_throttle: 1.5` stagger chunk 注入 |
| owner_space 隔离问题 | run_id 随机生成 → compose 看不到旧数据 | **已重构：固定 `ogmemory_user_ids`，不再用 run_id** |
| run_id_map YAML key 类型 | YAML int key vs str lookup | **已删除：不再使用 run_id_map** |
| resume 错误跳过 context | 累计 query 数推算 context 进度 | **已重构：`completed_map` 按 qa_pair_id 确恢复** |
| results.json 写入损坏 | 中断时写入一半 | **已修复：原子写入（.tmp → rename）** |
| 残留进程卡死 | 旧 eval 占满 worker | 跑之前先 `pgrep -af main.py` 检查 |
| 杀服务 exit 144 | `pkill -f` 模式串匹配到 wrapper bash | 用 `fuser -k 8090/tcp` |
| DB 维度切换 | 改 dim 后旧向量不兼容 | 必须重建 DB + CREATE EXTENSION vector + BYPASSRLS |
| rag_retrieved 历史被覆盖 | 硬编码路径，每次跑写入同一目录 | **已修复：`rag_retrieved_dir` 可配置 + 日期前缀** |
| wipe 旧 DB 数据 | 误删 v9 数据 | **必须新建 DB（v10），绝不修改旧 DB** |

## 11. 当前环境状态（2026-06-25）

| 项目 | 值/状态 |
|------|---------|
| oG-Memory 数据库 | `ogmemory_v10`（384 dim, bge-small-en-v1.5） |
| embedder | `BAAI/bge-small-en-v1.5` @ `:8345`（OpenAI API 模式） |
| Reader model | `GLM-5` |
| Judge model | `GLM-5`（当前 key 无 gpt-4o） |
| owner_space | `user:user_0` ~ `user:user_4`（固定） |
| agent_config | `configs/agent_conf/RAG_Agents/glm-5/Structure_rag_glm-5-ogmemory.yaml` |
| dataset_config | `configs/data_conf/Accurate_Retrieval/LongMemEval/Longmemeval_s_star.yaml` |
| `rag_retrieved_dir` | YAML 中可配置，默认 `./outputs/rag_retrieved` |

## 12. 关联

- [[longmemeval-test-standardization]] — wiki runner 模式的 session 注入规范
- [[memoryagentbench-lme-s-run-20260624]] — 2026-06-24/25 实际运行踩坑记录
- [[volcengine-api-config]] — Volcengine embedding 候选（配额 07-03 恢复）
