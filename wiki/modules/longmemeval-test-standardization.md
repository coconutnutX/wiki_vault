---
name: longmemeval-test-standardization
description: LongMemEval 测试流程规范：实验命名、目录结构、数据库隔离、日志管理、服务启停、run_config 完整配置、踩坑清单
metadata:
  type: reference
---

# LongMemEval 测试流程规范

从 0611 和 0622 两轮测试的经验教训中提炼的标准化流程。核心原则：**每次测试是独立的、可追溯的、完整记录的实验单元**。

关联：[[ogmemory-deepdream-longmemeval-e2e-runbook]]（操作手册）、[[longmemeval-dream36-test-report]]（0611 测试报告）、[[longmemeval-dream36-retest-progress-0622]]（0622 重测进度）

## 1. 历史踩坑清单

| 问题                            | 发生轮次        | 根因                                                 | 本规范应对                           |
| ----------------------------- | ----------- | -------------------------------------------------- | ------------------------------- |
| output 目录混杂                   | 0611 + 0622 | 多轮测试结果放在同一目录，命名风格不一致（有的带日期有的不带）                    | 统一命名规范，每轮独立目录                   |
| signal_record 脏数据残留           | 0622        | TRUNCATE 只覆盖主表，漏了 signal_record 等                  | 新建数据库，彻底避免脏数据                   |
| schema migration 中途阻塞         | 0622        | 代码升级后 ensure_schema 的 DO block 未正确触发 P1A migration | 测试前先验证 schema 健康                |
| node_id 全 NULL 导致 compose 500 | 0622        | 新 schema 要求 node_id NOT NULL，但注入时没有生成              | 启动服务补齐 schema + backfill        |
| doubao 配额耗尽中途换模型              | 0622        | Volcengine 配额 7月3日才重置                              | run_config 记录所有模型配置，模型变更视为新实验轮次 |
| reader/judge key 在命令行环境变量     | 0611 + 0622 | 不留记录，下次无法追溯                                        | 写入 run_config.json              |
| 服务 cwd 不正确导致 dream tool 不加载   | 0611        | 从其他目录启动 server/app.py                              | 服务启停规范强制检查 cwd                  |
| psql 查询忘记 SET app.account_id  | 0611        | RLS FORCE 模式下返回 0 行                                | 数据库操作规范提醒 SET                   |

## 2. 实验轮次命名规范

每次完整测试（注入 → before QA → dream → after QA）称为一个**实验轮次（run）**。

### 命名格式

```
{日期8位}-{实验名}-{run_id}
```

- 日期：`YYYYMMDD` 格式
- 实验名：描述实验核心变量，如 `dream36-v3`、`dream36-doubao-gpt4omini-glm5`
- run_id：runner 自动生成的 8 位 hex

### 示例

```
20260611-dream36-gpt4o-doubao-84325a8b
20260622-dream36-doubao-gpt4omini-glm5-00080bf4
```

### 模型变更 = 新实验轮次

**如果中途更换任何一个角色的 LLM 模型（提取、compose/dream、reader、judge、embedding），视为新实验轮次，不从旧轮次继续。** 原因：不同模型组合产出的记忆质量和判分标准不同，结果不可比。

## 3. 目录结构规范

```
/data/Workspace2/repos/longmemeval/output/
└── {日期}-{实验名}-{run_id}/          ← 一轮测试的所有产物在同一目录
    ├── run_config.json                 ← 完整实验配置（所有模型/key/tag）
    ├── ingest/                         ← Step 2: 注入产物
    │   ├── details.jsonl
    │   └── pipeline.log
    ├── before-dream/                   ← Step 3: before-dream QA
    │   ├── details.jsonl
    │   ├── results.json
    │   └── ogmemory-compose.hypotheses.jsonl
    ├── deepdream/                      ← Step 4: DeepDream 结果
    │   ├── deepdream_results.json
    │   └── dream_stdout.log            ← dream 过程的服务端日志副本
    ├── after-dream/                    ← Step 5: after-dream QA
    │   ├── details.jsonl
    │   ├── results.json
    │   └── ogmemory-compose.hypotheses.jsonl
    ├── service_stdout.log              ← 整轮测试的服务端日志完整副本
    └── experiment_summary.json         ← 自动生成的实验总结（准确率对比等）
```

### 原则

1. **所有文件在一棵树下**，不散落
2. **每步结束后立即复制结果到实验目录**，不依赖 runner 的默认 output 路径
3. **测试结束保存服务端日志副本**，避免下一轮 truncate 覆盖丢失

### 旧结果归档

历史遗留的混杂 output 目录应归档到按日期命名的目录：

```
output/0611-dream36-gpt4o-doubao-84325a8b/    ← 0611 全部产物归档于此
output/0622-dream36-doubao-glm5-00080bf4/      ← 0622 全部产物归档于此
```

归档后原位置的内容可以删除或移走，避免与新一轮测试混淆。

## 4. run_config.json 完整配置

每轮测试必须生成一份完整的 `run_config.json`，包含**所有**模型/API 配置，不靠口头约定或散落的环境变量。

### 必须包含的字段

```json
{
  "run_id": "00080bf4",
  "experiment_name": "dream36-doubao-gpt4omini-glm5",
  "date": "2026-06-22",
  "data_file": "data/longmemeval_s_dream_36.json",
  "account_id": "longmemeval",
  "user_prefix": "dream-36",
  "user_id_format": "dream-36-00080bf4-{qid}",
  "agent_id": "openclaw-longmemeval",
  "models": {
    "extract": {
      "model": "doubao-seed-2-0-code-preview-260215",
      "api_base": "https://ark.cn-beijing.volces.com/api/coding/v3",
      "api_key_alias": "volcengine-doubao"
    },
    "compose_dream": {
      "model": "gpt-4o-mini",
      "api_base": "https://chatapi.littlewheat.com/v1",
      "api_key_alias": "chatapi-gpt4omini"
    },
    "reader": {
      "model": "GLM-5",
      "api_base": "http://113.46.219.251:8080/v1",
      "api_key_alias": "glm5-local"
    },
    "judge": {
      "model": "GLM-5",
      "api_base": "http://113.46.219.251:8080/v1",
      "api_key_alias": "glm5-local"
    },
    "embedding": {
      "model": "text-embedding-ada-002",
      "api_base": "https://chatapi.littlewheat.com/v1",
      "api_key_alias": "chatapi-embedding"
    }
  },
  "db_state_at_start": {
    "tables_truncated": ["all_business_tables"],
    "schema_version": "P1A",
    "schema_verified": true,
    "node_id_backfilled": true
  },
  "oG-Memory_branch": "dev_0608",
  "server_pid": 481386,
  "server_cwd_verified": true
}
```

**注意**：`api_key_alias` 只记录别名不记录实际 key。实际 key 存在安全位置（config/ogmem.yaml、环境变量），需要时可查。不要把 API key 明文写入 run_config。

## 5. 数据库隔离规范

### 5.1 推荐方案：创建新数据库（已验证可行 ✅）

**每轮测试创建一个新的 PostgreSQL 数据库**，而不是 TRUNCATE 旧数据库。这是最干净的隔离方式：

- 旧数据库完全保留，随时可以回看历史数据
- 新数据库 schema 由 ensure_schema 自动创建（最新版本，不会有 migration 阻塞）
- 没有脏数据残留风险
- 删除旧数据库只需一条 `DROP DATABASE` 命令

#### 创建新数据库步骤

```bash
# 1. 创建新数据库（用 postgres 超级用户，sudo 密码: 123123）
echo "123123" | sudo -S -u postgres psql -c "CREATE DATABASE ogmemory_v{版本号} OWNER ogmem;"

# 例如：
echo "123123" | sudo -S -u postgres psql -c "CREATE DATABASE ogmemory_v3 OWNER ogmem;"

# 2. 安装 pgvector 扩展（新数据库需要）
echo "123123" | sudo -S -u postgres psql -d ogmemory_v3 -c "CREATE EXTENSION IF NOT EXISTS vector;"

# 3. 修改 ogmem.yaml 的 connection_string（两个位置）
# storage.connection_string: dbname=ogmemory_v3
# vector_db.connection_string: dbname=ogmemory_v3

# 4. 重启服务（ensure_schema 会自动创建所有表）
pkill -f "python -u server/app.py"
sleep 3
truncate -s 0 /tmp/ogmem_stdout.log
cd /data/Workspace2/oG-Memory
nohup python -u server/app.py > /tmp/ogmem_stdout.log 2>&1 &
disown
sleep 10 && curl -s http://127.0.0.1:8090/api/v1/health

# 5. 触发一次 API 调用让 ensure_schema 执行
curl -s -X POST http://127.0.0.1:8090/api/v1/compose \
  -H 'Content-Type: application/json' \
  -d '{"account_id":"longmemeval","user_id":"schema-init","agent_id":"test","query":"init"}'
```

#### 数据库命名约定

```
ogmemory_{N}_{month}_{day}
```

- N从1开始递增
- 历史数据库：`ogmemory`（v1 = 0611 测试）
- 切换只需改 `ogmem.yaml` 的 `dbname` 并重启服务

#### ⚠️ pgvector 扩展必须手动安装

新数据库创建后 `pgvector` 扩展不存在，需要 `postgres` 超级用户手动安装：

```bash
echo "123123" | sudo -S -u postgres psql -d ogmemory_v{N} -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

否则 vector_index 表的 `embedding` 列（`vector(1536)` 类型）会报错 `type "vector" does not exist`。

#### ⚠️ 废弃表说明

以下 5 张表存在于旧数据库（0622 之前的版本创建），但**当前代码完全不使用**：

- `search_recall` — 已被 `signal_record` 表替代（`SignalRecorder` 类写入 `signal_record`）
- `dream_runs`
- `dream_signals`
- `dream_staged_candidates`
- `dream_signal_query_seen`

这些表在 Python 代码中零引用，不需要创建，不需要 TRUNCATE，不需要补齐。新数据库只需确保 `signal_record` 表存在（由 `db_schema.sql` 创建）。

### 5.6 RLS 操作注意事项

- **所有 psql 查询前必须 `SET app.account_id = 'longmemeval'`**，否则 RLS FORCE 模式下返回 0 行
- **手动修 schema 时需要临时 NO FORCE**：`ALTER TABLE context_nodes NO FORCE ROW LEVEL SECURITY` → 操作 → `ALTER TABLE context_nodes FORCE ROW LEVEL SECURITY`（恢复）
- **不要忘记恢复 FORCE**，否则 RLS 保护失效

### 5.7 数据库连接信息

| 参数         | 默认值            | 说明             |
| ---------- | -------------- | -------------- |
| host       | `127.0.0.1`    | 固定             |
| port       | `5432`         | 固定             |
| dbname     | `ogmemory_{N}` | **每轮测试用不同数据库** |
| user       | `ogmem`        | 固定             |
| password   | `ogmem123`     | 固定             |
| account_id | `longmemeval`  | 写入 run_config  |

连接字符串模板：`host=127.0.0.1 port=5432 dbname=ogmemory_v{N} user=ogmem password=ogmem123`

创建新数据库需要 postgres 超级用户权限（sudo 密码：123123）。

## 6. 日志管理规范

| 日志               | 运行时路径                         | 保存路径                                | 说明                               |
| ---------------- | ----------------------------- | ----------------------------------- | -------------------------------- |
| 服务 stdout/stderr | `/tmp/ogmem_stdout.log`       | `{实验目录}/service_stdout.log`         | 每轮开始前 truncate；**整轮结束后复制到实验目录**  |
| dream 过程服务端日志    | `/tmp/ogmem_stdout.log`（同一文件） | `{实验目录}/deepdream/dream_stdout.log` | dream 步骤开始前 truncate；dream 结束后复制 |
| 注入 pipeline.log  | runner 默认 output 位置           | `{实验目录}/ingest/pipeline.log`        | runner 自动生成                      |
| QA eval 脚本日志     | 脚本 stdout                     | 脚本自身不写日志文件                          | 如需保存，手动 tee                      |

### 时间线

```
轮次开始 → truncate /tmp/ogmem_stdout.log → Step 2 注入 → Step 3 before QA
→ truncate /tmp/ogmem_stdout.log（dream 前清空便于追踪）
→ Step 4 dream → 复制 /tmp/ogmem_stdout.log 到 deepdream/dream_stdout.log
→ Step 5 after QA → 复制 /tmp/ogmem_stdout.log 到 service_stdout.log（完整日志）
```

## 7. 服务启停规范

### 启动（每轮开始或代码变更后重启）

```bash
# 停止旧服务
pkill -f "python -u server/app.py"
sleep 3

# 清空日志
truncate -s 0 /tmp/ogmem_stdout.log

# 启动（cwd 必须是 oG-Memory 目录）
cd /data/Workspace2/oG-Memory
nohup python -u server/app.py > /tmp/ogmem_stdout.log 2>&1 &
disown
SERVER_PID=$!

# 等待启动
sleep 10
curl -s http://127.0.0.1:8090/api/v1/health

# 验证 cwd
ls -la /proc/${SERVER_PID}/cwd
# 必须指向 /data/Workspace2/oG-Memory

# 记录 PID 到 run_config
echo "server_pid: ${SERVER_PID}" 
```

### 验证清单

| 检查项 | 期望 | 命令 |
|--------|------|------|
| health | `{"status":"ok","sql":"connected"}` | `curl -s http://127.0.0.1:8090/api/v1/health` |
| cwd | `/data/Workspace2/oG-Memory` | `ls -la /proc/${PID}/cwd` |
| LLM 模型 | 与 run_config 一致 | health 输出的 `llm` 字段 |

### ⚠️ cwd 陷阱

服务进程的 cwd 必须是 `/data/Workspace2/oG-Memory`。否则 dream 的 prompt-driven tool 不加载（`prompt_driven_specs=[]`），LLM 只会用 operational tools 而不产出 consolidation/correction。

症状：`Template not found: dream/prompts/dream.yaml`、`prompt_driven_specs=[]`。

## 8. 流程顺序（不可变更）

```
Step 1: 清空数据库 + 验证 schema → Step 2: 注入 → Step 3: Before-dream QA → Step 4: DeepDream → Step 5: After-dream QA → Step 6: 对比结果
```

**先 before QA → dream → after QA** 的顺序是设计意图，不能颠倒：

- Before-dream QA 的 compose 会积累 signal_record 统计数据
- Dream 的 `query_signal_stats` 需要 signal_record 数据才能精准判断哪些记忆值得 consolidation
- 如果先跑 dream 再跑 QA，signal_record 为空，dream 只能靠 acquire_recent 而没有召回频率信号

## 9. 脚本适配需求

当前脚本（runner、run_qa_eval.py、run_deepdream.py）的默认 output 路径不支持本规范的目录结构。有两种方案：

### 方案 A：脚本参数化输出路径（推荐）

修改脚本支持 `--output-dir` 参数，让每步直接写入实验目录的对应子目录。需要改：

- `run_qa_eval.py`：`--output-dir` → 直接写 `{dir}/before-dream/` 或 `{dir}/after-dream/`
- `run_deepdream.py`：`--output-dir` → 直接写 `{dir}/deepdream/`
- runner：`--output-dir` → 直接写 `{dir}/ingest/`

### 方案 B：后置复制（当前可行）

脚本仍用默认 output 路径，每步结束后手动复制到实验目录。简单但容易遗漏。

**建议先用方案 B 过渡，后续改脚本支持方案 A。**

## 10. 完整流程速查（新数据库方案）

```bash
# ==================================================
# 实验轮次: {日期}-{实验名}-{run_id}
# ==================================================

DATE="YYYYMMDD"
EXPERIMENT="dream36-v3"
DB_VER="v3"  # 数据库版本号，本轮使用 ogmemory_v3

# ==============================
# Step 0: 环境 + 分支
# ==============================
conda activate py11
cd /data/Workspace2/oG-Memory && git checkout dev_0608

# ==============================
# Step 1: 创建新数据库
# ==============================
echo "123123" | sudo -S -u postgres psql -c "CREATE DATABASE ogmemory_${DB_VER} OWNER ogmem;"
echo "123123" | sudo -S -u postgres psql -d ogmemory_${DB_VER} -c "CREATE EXTENSION IF NOT EXISTS vector;"

# ==============================
# Step 2: 修改 ogmem.yaml + 重启服务
# ==============================
# 手动修改 config/ogmem.yaml 的两处 connection_string:
#   dbname=ogmemory → dbname=ogmemory_v3
# 或用 sed:
sed -i "s/dbname=ogmemory /dbname=ogmemory_${DB_VER} /g" config/ogmem.yaml

pkill -f "python -u server/app.py"; sleep 3
truncate -s 0 /tmp/ogmem_stdout.log
cd /data/Workspace2/oG-Memory
nohup python -u server/app.py > /tmp/ogmem_stdout.log 2>&1 &
disown
sleep 10 && curl -s http://127.0.0.1:8090/api/v1/health
# 记录 PID，验证 cwd

# 触发一次 API 调用让 ensure_schema 创建核心表
curl -s -X POST http://127.0.0.1:8090/api/v1/compose \
  -H 'Content-Type: application/json' \
  -d '{"account_id":"longmemeval","user_id":"schema-init","agent_id":"test","query":"init"}'

# ==============================
# Step 2a: Schema 健康检查
# ==============================
PGPASSWORD=ogmem123 psql -h 127.0.0.1 -U ogmem -d ogmemory_${DB_VER} -c \
  "SELECT table_name FROM information_schema.tables WHERE table_schema='public' ORDER BY table_name;"
# 期望 25 个表全部存在

PGPASSWORD=ogmem123 psql -h 127.0.0.1 -U ogmem -d ogmemory_${DB_VER} -c \
  "SELECT column_name FROM information_schema.columns WHERE table_name='vector_index' AND column_name IN ('space_id','owner_space','node_uri','node_id');"
# 期望 4 列全部存在

# ==============================
# Step 3: 注入数据
# ==============================
cd /data/Workspace2/repos/longmemeval
python -m longmemeval_test.cli run configs/dream-36-ingest.toml --only health_check,run

# 读取 run_id 并创建实验目录
RUN_ID=$(python3 -c "import json; print(json.load(open('output/dream-36-ingest/run_config.json'))['run_id'])")
EXP_DIR="output/${DATE}-${EXPERIMENT}-${RUN_ID}"
mkdir -p ${EXP_DIR}/{ingest,before-dream,deepdream,after-dream}

# 复制注入产物到实验目录
cp output/dream-36-ingest/run_config.json ${EXP_DIR}/
cp output/dream-36-ingest/details.jsonl ${EXP_DIR}/ingest/
cp output/dream-36-ingest/pipeline.log ${EXP_DIR}/ingest/

# 写入完整 run_config（追加模型信息 + dbname）

# ==============================
# Step 4: Before-dream QA 评测
# ==============================
OPENAI_API_KEY="{reader/judge key}" \
OPENAI_BASE_URL="{reader/judge api base}" \
python scripts/run_qa_eval.py \
  --runner-output output/dream-36-ingest \
  --data-file data/longmemeval_s_dream_36.json \
  --api-url http://127.0.0.1:8090 \
  --account-id longmemeval \
  --reader-model {reader_model} \
  --judge-model {judge_model} \
  --tag before-dream

# 复制结果到实验目录
cp -r output/before-dream-${RUN_ID}/* ${EXP_DIR}/before-dream/

# ==============================
# Step 5: DeepDream
# ==============================
truncate -s 0 /tmp/ogmem_stdout.log

python scripts/run_deepdream.py \
  --runner-output output/dream-36-ingest \
  --data-file data/longmemeval_s_dream_36.json \
  --api-url http://127.0.0.1:8090 \
  --account-id longmemeval \
  --pause 30 --timeout 600

# 复制结果到实验目录
cp output/dream-36-ingest/deepdream_results.json ${EXP_DIR}/deepdream/
cp /tmp/ogmem_stdout.log ${EXP_DIR}/deepdream/dream_stdout.log

# ==============================
# Step 6: After-dream QA 评测
# ==============================
OPENAI_API_KEY="{reader/judge key}" \
OPENAI_BASE_URL="{reader/judge api base}" \
python scripts/run_qa_eval.py \
  --runner-output output/dream-36-ingest \
  --data-file data/longmemeval_s_dream_36.json \
  --api-url http://127.0.0.1:8090 \
  --account-id longmemeval \
  --reader-model {reader_model} \
  --judge-model {judge_model} \
  --tag after-dream

cp -r output/after-dream-${RUN_ID}/* ${EXP_DIR}/after-dream/

# ==============================
# Step 7: 保存完整服务日志 + 生成总结
# ==============================
cp /tmp/ogmem_stdout.log ${EXP_DIR}/service_stdout.log

python3 -c "
import json
run_id = json.load(open('${EXP_DIR}/run_config.json'))['run_id']
summary = {}
for tag in ['before-dream', 'after-dream']:
    path = '${EXP_DIR}/' + tag + '/results.json'
    d = json.load(open(path))
    summary[tag] = {
        'accuracy': d['accuracy'],
        'correct': d['correct'],
        'graded': d['graded'],
        'by_question_type': d['by_question_type']
    }
json.dump(summary, open('${EXP_DIR}/experiment_summary.json', 'w'), indent=2)
for tag, s in summary.items():
    print(f'{tag}: accuracy={s[\"accuracy\"]} ({s[\"correct\"]}/{s[\"graded\"]})')
    for qtype, stats in s['by_question_type'].items():
        print(f'  {qtype}: {stats.get(\"accuracy\",\"n/a\")} ({stats[\"correct\"]}/{stats[\"graded\"]})')
"
```

## 11. 当前环境状态（2026-06-22）

### 已验证：新数据库方案可行 ✅

- `ogmemory_v3` 数据库已创建，20 个有效表全部存在（包括 P1A migration 新列 + signal_record）
- pgvector 0.8.2 扩展已安装
- `signal_record` 表由 `db_schema.sql` 自动创建，无需手动补齐
- 服务切换到新数据库后 health check 通过，compose API 正常
- `ogmemory_v3` 可直接用于下一轮测试

### 当前活跃数据库

| 数据库 | 状态 | 说明 |
|--------|------|------|
| `ogmemory` | 含 0622 中断数据（13k context_nodes + 23k vector_index + 629 signal_record脏数据） | 旧数据，可作为历史参考 |
| `ogmemory_v3` | 空（schema 完整 + pgvector） | ✅ 可直接用于下一轮测试 |

### 当前服务

- oG-Memory 服务运行中（已恢复指向 `ogmemory` 旧数据库）
- 下次测试开始时改为 `ogmemory_v3`

### 其他遗留

| 项 | 状态 | 处理建议 |
|---|---|---|
| output 目录 | 0611 + 0622 两轮混杂 | 归档到按日期命名的目录 |
| ChromaDB 文件 | 4 个 SQLite（可忽略） | 不影响，PG 已替代 |
| Docker OpenGauss | 已停止 7 周 | 不影响，已用 apt PostgreSQL |
| LLM 配额 | doubao 配额耗尽（7月3日重置） | compose/dream 用 gpt-4o-mini |

**下一轮测试**：直接使用 `ogmemory_v3` → 改 ogmem.yaml → 重启服务 → 按 Section 10 流程执行。
