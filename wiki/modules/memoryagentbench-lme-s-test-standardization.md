---
name: memoryagentbench-lme-s-test-standardization
description: MemoryAgentBench LongMemEval S* 测试流程规范：对接 oG-Memory（chunk 模式）、实验命名、目录结构、数据库隔离、before/after dream 两轮评测、踩坑清单
metadata:
  type: reference
---

# MemoryAgentBench LongMemEval S* 测试流程规范

基于 [[longmemeval-test-standardization]] 的经验教训，适配 MemoryAgentBench 框架对 LongMemEval S* 数据集的测试流程。核心差异：S* 采用"inject once, query multiple"设计（一个长 context 对应多个 question），评测使用 LLM-as-judge。

## 1. 对接方案：Chunk 模式

MemoryAgentBench 采用 **chunk 模式** 注入，而非 wiki runner 的 session 模式：
- 每个长 context 按 `agent_chunk_size`（4096）切分成 chunks
- 每个 chunk 调一次 `after_turn`
- user_id 格式：`{user_prefix}-{run_id}-ctx{context_id}`
- session_id 格式：`{user_id}-chunk-{idx:04d}`

### 与 wiki runner 的差异

| 方面 | Wiki runner（session 模式） | MemoryAgentBench（chunk 模式） |
|------|---------------------------|----------------------------|
| 注入单位 | 按 session 边界 | 按 chunk（4096字） |
| session_id | `{user_id}-hist-{idx:04d}-{sid}` | `{user_id}-chunk-{idx:04d}` |
| 时间戳 | 原始 session 时间 | 当前时间 |
| 抽取触发 | 每个 session 触发 | chunk 累积达到 threshold 触发 |

## 2. 模型配置

### Reader vs Judge

| 角色 | 用途 | 配置位置 | 当前值 |
|------|------|----------|--------|
| **Reader** | compose 后回答问题 | agent_config.yaml 的 `model` | `GLM-5` |
| **Judge** | LLM-as-judge 评测 | `llm_based_eval/longmem_qa_evaluate.py` | `gpt-4o` |
| **oG-Memory compose/dream/extract** | 记忆抽取和组装 | oG-Memory 的 `ogmem.yaml` | 配置中定义 |

### 配置文件命名（已修复）

配置文件已重命名为实际模型名称：
- **配置文件路径**: `configs/agent_conf/RAG_Agents/glm-5/Structure_rag_glm-5-ogmemory.yaml`
- **output_dir**: `./outputs/glm-5-ogmemory`

命名与实际模型一致，避免混淆。

## 3. agent_config.yaml 关键字段

```yaml
agent_name: Structure_rag_ogmemory
model: GLM-5                    # Reader 模型
temperature: 0.0
output_dir: ./outputs/{name}

ogmemory_api_url: http://127.0.0.1:8090
ogmemory_account_id: longmemeval
ogmemory_user_prefix: mabench
ogmemory_agent_id: openclaw-longmemeval
ogmemory_token_budget: 128000
ogmemory_run_id: null           # 可选，不设则自动生成 UUID[:8]

ogmemory_wait_timeout: 120.0    # 等待抽取完成的超时
ogmemory_wait_interval: 5.0     # poll 间隔
ogmemory_post_inject_wait: 30.0 # 注入后固定等待（后台抽取时间）

retrieve_num: 100
agent_chunk_size: 4096
```

### 注入流程

```python
# 每个 chunk → after_turn
POST /api/v1/after_turn
{
  "accountId": "longmemeval",
  "userId": "mabench-{run_id}-ctx{N}",
  "agentId": "openclaw-longmemeval",
  "sessionId": "mabench-{run_id}-ctx{N}-chunk-{idx:04d}",
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
  "userId": "mabench-{run_id}-ctx{N}",
  "agentId": "openclaw-longmemeval",
  "sessionId": "mabench-{run_id}-ctx{N}-question",
  "query": "<question>",
  "messages": [{"role": "user", "content": "<question>", "created_at": "..."}],
  "tokenBudget": 128000
}

# compose 返回的 slots →拼接 → 发给 reader LLM
```

### 注入后等待机制

MemoryAgentBench 在 `_memorize_context_chunks` 结束后会：
1. 固定等待 `ogmemory_post_inject_wait`（30s）——后台异步抽取需要时间
2. Poll compose 直到 `hit_count >= 1` 或 `retrieved_chars >= 100`
3. 超时后继续执行（warn 状态）

## 4. 数据库隔离

每轮测试用**新数据库**：

```bash
DB_VER="v6"
echo "123123" | sudo -S -u postgres psql -c "CREATE DATABASE ogmemory_${DB_VER} OWNER ogmem;"
echo "123123" | sudo -S -u postgres psql -d ogmemory_${DB_VER} -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

修改 `oG-Memory/config/ogmem.yaml` 两处 `connection_string` 的 `dbname` → 重启服务。

**⚠️ Before/After dream 用同一数据库**——dream 是对同一记忆的 consolidation。

## 5. 完整流程

```bash
DATE="20260623"
EXPERIMENT="mabench-sstar"
DB_VER="v6"

conda activate py11

# Step 1: 创建数据库
echo "123123" | sudo -S -u postgres psql -c "CREATE DATABASE ogmemory_${DB_VER} OWNER ogmem;"
echo "123123" | sudo -S -u postgres psql -d ogmemory_${DB_VER} -c "CREATE EXTENSION IF NOT EXISTS vector;"

cd /data/Workspace2/oG-Memory
sed -i "s/dbname=ogmemory_v5/dbname=ogmemory_${DB_VER}/g" config/ogmem.yaml

# Step 2: 启动服务
pkill -f "python -u server/app.py"; sleep 3
truncate -s 0 /tmp/ogmem_stdout.log
nohup python -u server/app.py > /tmp/ogmem_stdout.log 2>&1 &
disown
sleep 20
curl -s http://127.0.0.1:8090/api/v1/health

# Step 3: Before-dream 评测
cd /data/Workspace2/repos/MemoryAgentBench
EXP_DIR="output/${DATE}-${EXPERIMENT}"
mkdir -p ${EXP_DIR}/{before-dream,deepdream,after-dream,eval_results}

python main.py \
  --agent_config configs/agent_conf/RAG_Agents/glm-5/Structure_rag_glm-5-ogmemory.yaml \
  --dataset_config configs/data_conf/Accurate_Retrieval/LongMemEval/Longmemeval_s_star.yaml \
  --force

# 复制结果
cp outputs/*/Accurate_Retrieval/*results.json ${EXP_DIR}/before-dream/

# Step 4: DeepDream（可选）
truncate -s 0 /tmp/ogmem_stdout.log
curl -s -X POST http://127.0.0.1:8090/api/v1/deepdream \
  -H 'Content-Type: application/json' \
  -d '{"accountId":"longmemeval"}'
cp /tmp/ogmem_stdout.log ${EXP_DIR}/deepdream/dream_stdout.log

# Step 5: After-dream 评测
python main.py \
  --agent_config configs/agent_conf/RAG_Agents/glm-5/Structure_rag_glm-5-ogmemory.yaml \
  --dataset_config configs/data_conf/Accurate_Retrieval/LongMemEval/Longmemeval_s_star.yaml \
  --force

cp outputs/*/Accurate_Retrieval/*results.json ${EXP_DIR}/after-dream/

# Step 6: LLM-as-judge 评测
python llm_based_eval/longmem_qa_evaluate.py \
  --evaluated_method glm-5-ogmemory \
  --dataset longmemeval_s* \
  --judge_model gpt-4o \
  --judge_api_key $OPENAI_API_KEY \
  --judge_api_base https://chatapi.littlewheat.com/v1 \
  --output_dir ${EXP_DIR}/eval_results/

# Step 7: 保存日志
cp /tmp/ogmem_stdout.log ${EXP_DIR}/service_stdout.log
```

## 6. 踩坑清单

| 问题 | 原因 | 解决 |
|------|------|------|
| after_turn 500 RLS 错误 | 旧数据库有 FORCE RLS | 用新数据库或 `ALTER TABLE NO FORCE RLS` |
| compose 返回空记忆 | 注入后立即查询，抽取未完成 | 固定等待 + poll compose |
| 配置文件名与实际模型不符 | 历史遗留 | **已修复：重命名为 glm-5-ogmemory.yaml** |
| reader/judge 模型混淆 | 一个 `model` 字段用于 reader，judge 在 eval 脚本 | 文档明确区分，judge 可配置 |
| judge 无 apikey 报错 | 硬编码依赖 OPENAI_API_KEY | 添加 `--judge_api_key` 参数支持 |
| output 目录混杂 | 多轮测试在同一目录 | 每轮独立实验目录 |

## 7. 当前环境状态（2026-06-23）

| 项目 | 值/状态 |
|------|---------|
| oG-Memory 数据库 | `ogmemory_v6`（新建，schema 完整） |
| Reader model | `GLM-5` |
| Judge model | `gpt-4o`（可配置） |
| agent_config | `configs/agent_conf/RAG_Agents/glm-5/Structure_rag_glm-5-ogmemory.yaml` |
| dataset_config | `configs/data_conf/Accurate_Retrieval/LongMemEval/Longmemeval_s_star.yaml` |
