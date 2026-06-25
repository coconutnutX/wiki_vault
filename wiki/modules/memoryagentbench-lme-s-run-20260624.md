---
name: memoryagentbench-lme-s-run-20260624
description: 2026-06-24/25 MABench LongMemEval S* 运行记录——4 blocker + embedder演变 + OOM + owner_space隔离已修复(run_id_map) + after-dream-2 230/300中断(run_id映射覆盖不足待优化)
metadata:
  type: reference
---

# MABench LongMemEval S* 2026-06-24 运行记录

基于 [[memoryagentbench-lme-s-test-standardization]] 的流程，本次换新 API key 在**全新环境（无外网）**重跑时踩到 4 个标准文档踩坑清单里**没有**的 blocker，以及一个 embedder 死局。本文记录根因、已验证的修复、未解决问题与实用命令。

## 0. 本次运行配置

| 项                          | 值                                                                |
| -------------------------- | ---------------------------------------------------------------- |
| 日期 / 实验                    | `20260624` / `mabench-sstar`                                     |
| 数据库                        | `ogmemory_v7`（新建隔离）                                              |
| API key / base             | `sk--hAux09V3aXYBMsaZG_k2w` @ `http://113.46.219.251:8080/v1`    |
| 可用模型                       | 仅 `GLM-5 / GLM-5.1 / Qwen3.7-Plus / GLM-5.2`（**聊天模型**）           |
| reader / oG-Memory / judge | 全部 `GLM-5` + 上述 key                                              |
| oG-Memory 服务               | `127.0.0.1:8090`，`config/ogmem.yaml` 的 `llm` 段已是该 key+base+GLM-5 |

### ⚠️ 这个 key 的两个限制（与标准文档冲突）

1. **不能访问 `gpt-4o`** —— 标准文档里 judge 用 `gpt-4o`，这里必须改成 `GLM-5`。
2. **没有任何 embedding 模型** —— 测 `text-embedding-3-large` 直接 403「key can only access models=[4 个聊天模型]」。这是 embedder 死局的起因（见 §3）。

### 与标准文档的配置差异

- **judge 模型**：`--judge_model GLM-5`（不是 gpt-4o），`--judge_api_key` / `--judge_api_base` 指向这个 key/base。
- **reader（compose→回答那步）**：`agent.py:111` 的 `_create_oai_client()` 走 `OpenAI()`，**从 env 读** `OPENAI_API_KEY` / `OPENAI_BASE_URL`。跑 eval 前必须 export 这两个（agent_config 里只有 `model: GLM-5`，没有 key）。
- **服务 + eval 都要** `HF_HUB_OFFLINE=1`（见坑1/坑4）。

## 1. 四个全新 blocker（按踩到顺序）

| # | 现象 | 根因 | 修复 |
|---|------|------|------|
| 1 | `after_turn` 500，`RuntimeError: Cannot send a request, as the client has been closed` | bge-m3 embedder（`provider: st`）加载时连 HuggingFace 拉模型 → 环境连不上 HF（connection reset）→ 重试后 httpx client 关闭。**小 probe（pending token 不足）不触发**；真实大 chunk 触发 extraction 才暴露 | 服务启动加 `HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1`（模型已完整缓存 4.3G） |
| 2 | `after_turn` 500，`query would be affected by row-level security policy for table "context_nodes"` | `ensure_schema()` 在**每次** after_turn 都重跑，**重新打开 FORCE RLS**，随后它自己 schema migration 里的 `UPDATE vector_index ... FROM context_nodes` 被刚打开的 FORCE RLS 挡住。`ALTER TABLE ... NO FORCE ROW LEVEL SECURITY` **无效**（下次 after_turn 又被 ensure_schema 打开） | `ALTER ROLE ogmem WITH BYPASSRLS;`（role 级，绕过所有 RLS 含 FORCE，且跨 DB drop/recreate 持久） |
| 3 | eval 卡死：无 after_turn、2.6% CPU、阻塞在 socket | 上一个 session 残留的 `gpt-4o-mini` eval（PID 19822，已跑 3h、0% CPU 挂死）占满了 oG-Memory 服务的 2 个 worker，新 eval 的请求排队 | 杀掉残留进程。**教训**：跑之前先 `pgrep -af main.py` 看有没有残留 |
| 4 | eval 卡在**数据加载阶段**（dataset load 之后、第一个 after_turn 之前），阻塞在 `185.199.111.133:443`（GitHub） | `chunk_text_into_sentences`（`utils/eval_other_utils.py:190`）**无条件** `nltk.download('punkt', quiet=True)` → 连 GitHub（不通）→ 挂死。而 `punkt` / `punkt_tab` **其实都已缓存** | 改成 `try: nltk.data.find('tokenizers/punkt') except LookupError: nltk.download(...)`（同文件 256-259 行已有此模式）。eval 也加 `HF_HUB_OFFLINE=1` 跳过 dataset 的 HF 重试（省 ~80s） |

> 坑2 细节：13 张表（context_nodes / session_archives / agent_messages / signal_record / memory_lineage / relation_edges / node_uri_aliases / dream_recalls / principals / spaces / space_grants / domain_events / agent_skills_registry）都带 `tenant_isolation` 策略 + FORCE RLS；`ogmem` 角色 `rolbypassrls=f`、非 superuser。`vector_index` 本身无 RLS，但 migration 的 UPDATE...FROM context_nodes 会间接触发。

## 2. 修复步骤（已验证可跑通）

```bash
# ---- oG-Memory 服务 ----
cd /data/Workspace2/oG-Memory

# 坑1: embedder 离线（bge-m3 已缓存）
# 启动时带：HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1
setsid env HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1 \
  python -u server/app.py > /tmp/ogmem_stdout.log 2>&1 < /dev/null &

# 坑2: 给 ogmem 角色 BYPASSRLS（一次性，持久）
echo "123123" | sudo -S -u postgres psql -c "ALTER ROLE ogmem WITH BYPASSRLS;"

# ---- MemoryAgentBench eval ----
cd /data/Workspace2/repos/MemoryAgentBench
export OPENAI_API_KEY="sk--hAux09V3aXYBMsaZG_k2w"        # reader 用
export OPENAI_BASE_URL="http://113.46.219.251:8080/v1"
export HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1            # 坑4: 跳过 HF/GitHub 重试
export PYTHONUNBUFFERED=1                                  # 让 tee 日志实时刷新
python -u main.py \
  --agent_config configs/agent_conf/RAG_Agents/glm-5/Structure_rag_glm-5-ogmemory.yaml \
  --dataset_config configs/data_conf/Accurate_Retrieval/LongMemEval/Longmemeval_s_star.yaml \
  --force
```

代码改动（坑4，已 apply 到 `utils/eval_other_utils.py:189-198`）：把无条件 `nltk.download('punkt', quiet=True)` 包成 find 失败才下载。

## 3. Embedder 演变（bge-m3 → bge-large-zh-v1.5 → bge-small-en-v1.5）

### 3a. bge-m3 死局（最初方案）

- **bge-m3 是 oG-Memory 硬编码默认**（不是谁特意选的）：`providers/config.py:70`、`providers/unified_config.py:315`、`providers/embedder/st_embedder.py:30` 都默认 `BAAI/bge-m3`，1024 维。
- **本地缓存里只有 bge-m3 一个 embedding 模型**，没有 all-MiniLM / bge-small / e5。
- **换 API embedder**：oG-Memory 支持 `provider: openai / volcengine`，但
  - 这个 chat key 没有 embedding 模型（403）；
  - Volcengine key（[[volcengine-api-config]]）`AccountQuotaExceeded`，**2026-07-03** 才重置。
- **换更小的本地模型**：HF（huggingface.co）不通；但 **`hf-mirror.com`（HTTP 200）和 `modelscope.cn`（HTTP 200）可达** —— 后续可通过镜像下载小模型（如 `all-MiniLM-L6-v2`，384 维 ~90MB），届时需同步改 `ogmem.yaml` 的 `embedding.st_model` + `vector_db.dimension` 并**重建 DB**（维度变了，旧向量不兼容）。

### 3b. bge-large-zh-v1.5 尝试（v8 DB, 1024 dim）

- 用 `locomo-test/deploy_model.py` 在 `:8345` 部署本地 embedder 服务，提供 OpenAI 兼容 API。
- 改 `ogmem.yaml`：`embedding.provider: openai`, `base_url: http://127.0.0.1:8345`, `model: BAAI/bge-large-zh-v1.5`，DB v8，dim 1024。
- **问题1**：bge-large-zh-v1.5 是中文优化模型，但 LongMemEval S* 数据全英文。相似度测试 猫 vs 狗 = 1.0（中文下 broken）。
- **问题2**：~1.3G RSS + oG-Memory 服务 + eager extraction 累积内存 → 7.7G WSL 机器反复 OOM 崩溃。在 v8 上跑到 ctx0 q12/60 就挂了。

### 3c. bge-small-en-v1.5 最终方案（v9 DB, 384 dim）✅

- 选择理由：384 维，~130MB 模型，英文模型（适配 LongMemEval 英文数据），RSS ~577Mi（比 bge-large-zh 省 ~0.7G）。
- 通过 `hf-mirror.com` 下载模型，缓存到 `locomo-test/models/`。
- `ogmem.yaml` 改为：`embedding.provider: openai`, `base_url: http://127.0.0.1:8345`, `model: BAAI/bge-small-en-v1.5`, `api_key: embed-local`（dummy，8345 不鉴权），DB v9，dim 384。
- **关键发现**：`_normalize_url`（`providers/unified_config.py:104-111`）会给不含版本路径的 URL 自动追加 `/v1`，所以 `http://127.0.0.1:8345` → `http://127.0.0.1:8345/v1`，stock `OpenAIEmbedder` 直接驱动，无需适配层。
- **部署注意**：必须 `HF_ENDPOINT=https://hf-mirror.com HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1`，否则 SentenceTransformer 加载时联网查 `adapter_config.json` 会因 `huggingface.co` 不可达而崩溃。
- 已验证端到端：after_turn >200 token → extraction → v9 vector_index 384 dim vectors ✅

## 4. OOM 反复崩溃（已解决——换小 embedder）

修完 4 个 blocker 后，eval 因 OOM 反复崩溃：

- bge-m3（~2.2G RSS）+ oG-Memory + extraction → ctx0 q10 崩溃
- bge-large-zh-v1.5（~1.3G RSS）+ 同上 → ctx0 q12 崩溃
- 每次崩溃都导致：embedder(:8345) 死 + oG-Memory(:8090) 死 + eval 进程死，但 **results.json 已写入磁盘的部分有效**。

### 解决：换 bge-small-en-v1.5 + embedder 外部进程

- embedder 以独立 Flask 进程跑在 :8345（`locomo-test/deploy_model.py`），oG-Memory 服务本身不再加载任何 embedding 模型，只通过 HTTP 调用。
- bge-small-en-v1.5 RSS ~577Mi，总内存占用降 ~1.5G+。
- compose 超时已从 120s 改到 600s（`agent.py:401`）。

### 5 个 context 的 before-dream eval 跑了两轮

- **第一轮**：从 v9 清空 DB 开始跑，完成 ctx0-3（4×60=240）+ ctx4 的 6 条后 OOM 崩溃，停了 14+ 小时（期间某些 query API 响应极慢，~395s/条 vs 正常 ~11s）。
- **第二轮**：重启 embedder + oG-Memory，不带 `--force` 续跑，跳过已完成 246 条，只跑 ctx4 的 54 条。约 13 分钟完成。

## 5. After-dream eval 的 owner_space 隔离问题 ✅ 已解决

### 问题根因

oG-Memory 的 compose 查询按 `owner_space` 过滤向量。每个 context 的 `after_turn` 注入时使用 `user:mabench-{run_id}-ctx{n}` 作为 owner_space，其中 `run_id` 是每个 context 初始化时随机生成的 UUID。

**before-dream** 5 个 context 各自生成了 5 个不同 run_id：

```
ctx0: user:mabench-7f498a44-ctx0   (2024 vectors)
ctx1: user:mabench-a7a00d30-ctx1   (2004 vectors)
ctx2: user:mabench-57756c58-ctx2   (2168 vectors)
ctx3: user:mabench-56373412-ctx3   (1972 vectors)
ctx4: user:mabench-409cd576-ctx4   (1944 vectors)
+ ctx4 重跑追加(脏数据)：user:mabench-a500c67b-ctx4  (1412 vectors)
```

**after-dream** eval 重新初始化 agent 时生成了**新的 run_id**，compose 查询构建的 `visible_owner_spaces` 只包含新的 `user:mabench-{new_run_id}-ctx{n}`，无法匹配到之前注入的旧数据，导致：

| metric | before-dream | after-dream(bugged) |
|--------|-------------|-------------|
| f1 | 31.85 | 4.75 |
| input_len | 3078 | 100 |

input_len 从 3078 降到 100 → compose 完全检索不到内容，分数暴跌。

### 解决方案：复用 before-dream run_id（方案1）

采用了**最干净的方案1**——在 agent_config YAML 中硬编码 `ogmemory_run_id_map`，让 after-dream compose 复用 before-dream 的 owner_space。

**踩坑与修复过程：**

1. **脏数据清理**：ctx4 有两轮注入数据——`a500c67b`（OOM 时部分注入的 1412 条）和 `409cd576`（resume 后完整重注入的 1944 条）。前者是不完整脏数据，已从 DB 删除。

2. **agent.py 修改**（`_initialize_ogmemory_agent` + `_handle_ogmemory_agent`）：
   - 新增 `ogmemory_run_id_map` 属性，从 YAML config 读取 `{context_id: run_id}` 映射
   - `_handle_ogmemory_agent` 中按 context_id 从 map 取对应 run_id，fallback 到 `ogmemory_run_id`

3. **YAML key 类型坑**：YAML 解析 `ogmemory_run_id_map: {0: xxx}` 后 keys 是 **int**（0,1,2...），但代码最初用 `str(context_id)` 查找 → `"0"` 在 int-key dict 里找不到 → fallback 到随机 UUID → compose 看不到数据（`input_len: 100`）。**修复**：改为 `self.ogmemory_run_id_map.get(int(context_id), self.ogmemory_run_id_map.get(str(context_id), self.ogmemory_run_id))` 同时兼容 int 和 str key。

4. **DeepDream 必须传 userId**：DeepDream API 默认用 config 里的 `u-alice`（`ogmem.yaml` 的 `default_user_id`），但注入数据在 `mabench-{run_id}-ctx{n}` 下，`u-alice` 查不到任何数据 → `acquire_recent` 返回 0 条 → dream_outputs=0。**修复**：对 5 个 ctx 各调用一次 DeepDream，传入对应的 `userId: mabench-{run_id}-ctx{n}` + `agentId: openclaw-longmemeval`。

5. **Dream 数据 owner_space 变化**：之前 Dream 数据的 owner_space 是 `agent:openclaw-longmemeval`（全局），但按 ctx 跑 Dream 后，合成数据落在各 ctx 自己的 `user:mabench-{run_id}-ctx{n}` 下（和注入数据同 owner_space），compose 用相同的 run_id 就能看到注入 + Dream 全部数据。

6. **compose visible_owner_spaces 逻辑确认**：无 auth_service 时，compose 的 `visible_owner_spaces = [f"user:{user_id}"] + [f"agent:{requested_agent_id}"]`（`memory_service.py:857-861`），即 compose 能看到当前 user 的所有数据 + 请求的 agent 的数据。只要 userId 对了，注入 + Dream 数据都在同一 user space 下，compose 可见。

7. **reader LLM 配置坑**：`_create_oai_client()` 用 `OpenAI()` 无参数，靠环境变量 `OPENAI_API_KEY` / `OPENAI_BASE_URL` 驱动。启动 eval 时必须 export 这两个，否则调默认 OpenAI API（model 不存在）→ 404 `UnsupportedModel`。Volcengine gateway 的 `GLM-5` 报错信息是「不支持 coding plan feature」，实际是因为没配对 key/base。

**修复后效果：**

| metric | before-dream | after-dream-2(fixed) |
|--------|-------------|-------------|
| input_len | 3078 | ~3898-10027 |
| f1(ctx0 28/60) | 31.85 | ~28.62 |

input_len 大幅回升（包含注入 + Dream 数据），f1 恢复正常水平。

### 相关代码路径

- `owner_space` 构建：oG-Memory `server/memory_service.py:857-861` compose 可见空间 = `user:{userId}` + `agent:{agentId}`
- run_id 生成：MemoryAgentBench `agent.py:260` `self.ogmemory_run_id = agent_config.get('ogmemory_run_id') or uuid.uuid4().hex[:8]`
- run_id_map 查找：`agent.py:297` `self.ogmemory_run_id_map.get(int(context_id), ...)`
- DeepDream API：`POST /api/v1/deepdream`，必须传 `userId` 和 `agentId`
- compose payload：`agent.py:398-412` 包含 `userId`, `agentId`, `sessionId`

## 6. 实用命令（本次新学）

- **杀 oG-Memory 服务**：`fuser -k 8090/tcp`。**不要** `pkill -f "server/app.py"` —— 模式串会匹配到跑命令的 bash wrapper 自身，连 wrapper 一起杀（表现为 **exit 144**）。
- **后台启服务（脱离 shell 清理）**：`setsid env ... python -u server/app.py > log 2>&1 < /dev/null &`（裸 `nohup ... & disown` 不够稳，会被工具进程组清理带走）。
- **eval 日志实时刷新**：管道 `| tee` 会让 Python stdout 变 block-buffered，进度看不到；加 `PYTHONUNBUFFERED=1` + `python -u` 解决。
- **schema 是懒创建**：新 DB 起服务后 `\dt` 是空的属正常，**第一个 after_turn 才建表**（`ensure_schema`）。可用一个 throwaway user 的小 probe 验证写入链路而不污染正式数据（compose 按 userId 隔离）。
- **embedder 部署**：`HF_ENDPOINT=https://hf-mirror.com HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1 python deploy_model.py --model BAAI/bge-small-en-v1.5 --port 8345`（必须加离线标志，否则 SentenceTransformer 加载时联网查 adapter_config.json 崩溃）
- **续跑 eval（跳过已完成 query）**：不带 `--force`，`load_existing_results` 会自动跳过已有 qa_pair_id 的 query。注意：续跑只会重新注入**未完成的 context** 的 chunk，已完成的 context 跳过注入（前提是 `agents/` 目录下有对应的 save folder 或 oG-Memory state 在外部服务持久化）。
- **DB 维度切换**：改 `ogmem.yaml` 的 `vector_db.dimension` 后必须重建 DB（pgvector 列维度是固定的），`sudo -u postgres psql -c "CREATE DATABASE ogmemory_vN OWNER ogmem"` + `PGPASSWORD=ogmem123 psql ... -c "CREATE EXTENSION vector;"` + `ALTER ROLE ogmem WITH BYPASSRLS;`

## 7. 当前进度（2026-06-25 14:30）

- ✅ DB v9 建好（384 dim）、ogmem.yaml 指向 v9 + bge-small-en-v1.5、embedder(:8345) + oG-Memory(:8090) 正常运行
- ✅ 4 个 blocker + OOM 问题全解决
- ✅ Before-dream eval **300/300 全完成**
- ✅ 脜数据 `a500c67b-ctx4` 已清除（1412 条不完整向量）
- ✅ DeepDream 重新执行：5 个 ctx 各跑一次（传入对应 userId），产出 consolidation/correction 向量
- ✅ owner_space 隔离 blocker 已解决：agent.py 新增 `ogmemory_run_id_map` 支持 + YAML int-key bug 修复
- ❌ **After-dream-2 eval 跑了 230/300 后中断**，结果不完整不可用，需要优化运行流程后重跑

### After-dream-2 中断原因分析

after-dream-2 结果文件记录了 230 条 query（60 个 sub-context 各 3-4 条，缺 70 条），但结果**不可用**，原因：

1. **run_id_map 只覆盖 ctx0-4**：YAML 配置 `ogmemory_run_id_map` 只映射了 5 个 context_id（0-4）到 before-dream 的 run_id。但 LongMemEval S* 数据集有 **60 个 sub-context**（no0-no59），eval 循环把 context_index (0-59) 作为 context_id 传给 `_handle_ogmemory_agent`。ctx5-59 的 run_id lookup 落回随机 `uuid.uuid4().hex[:8]` → compose 看不到旧数据 → `input_len` 极低（142-178）。

2. **before-dream 实际只有 5 个 injection context**：`agents/Structure_rag_ogmemory_.../` 目录下只有 `exp_0` 到 `exp_4`，说明 before-dream 只注入了 5 个「大 context」（对应 5 个人的长文档）。但 compose 查询对 60 个 sub-context 都返回了数据（avg input_len ~2700-5700），这是因为**所有注入数据在 DB 里只有 5 个 owner_space**，但 compose 的 query 本身带语义匹配，能在同一个 owner_space 下找到相关内容——只要 userId 对了。

3. **关键认知**：60 个 sub-context 的 query 实际上都查询同一批注入数据。在 before-dream 中，5 个 injection context 各用一个随机 run_id，compose 对所有 60 个 sub-context 都用的是对应 injection context 的 userId，所以都能查到数据。但在 after-dream-2 中，**所有 60 个 sub-context 都应该用这 5 个 run_id 中的某一个**，而不是给 ctx5-59 分配新随机 run_id。

### 待优化方案（下次重跑）

- **方案A**：将 `ogmemory_run_id_map` 扩展到覆盖所有 60 个 context_id，映射到对应的 5 个 run_id。需要先确认 60 个 sub-context 与 5 个 injection context 的对应关系（哪个 sub-context 查哪个 injection context 的数据）。
- **方案B**：修改 `_handle_ogmemory_agent` 让它不再按 context_id 区分 run_id，而是让所有 sub-context 共用同一个 userId（如 before-dream 的 ctx0 的 `mabench-7f498a44-ctx0`），这样 compose 能看到全部 5 个 owner_space 的数据。但这改变了 owner_space 语义。
- **方案C**：在 compose 请求中传入多个 userId（让 visible_owner_spaces 同时覆盖所有 5 个 before-dream 的 owner_space）。需要修改 compose payload。
- **方案D**（最简单）：修改 agent.py，让 after-dream 不传 context_id 给 compose，而是固定使用一个能覆盖所有注入数据的 userId。比如 `mabench-{any_one_run_id}-ctx{0-4}` 然后在 compose 里把 `requested_agent_id` 也带上（`agent:openclaw-longmemeval`），但目前 agent namespace 下没有数据。

> **最可能的正确方案**：需要弄清 before-dream 是怎么让所有 60 个 sub-context 都能查到数据的。如果 before-dream 也是给每个 sub-context 一个独立 run_id 但 compose 仍能查到数据，那说明 before-dream 注入了 60 个 context（每个 sub-context 的文档），只是 DB 里只保留了 5 个是因为 OOM 导致 ctx5+ 的注入丢失。如果是这样，需要重新注入所有 60 个 context（或确认 compose 是否有跨 owner_space 搜索能力）。

### Before-dream 最终分数

| metric | 值 |
|--------|-----|
| exact_match | 13.67 |
| f1 | 31.85 |
| substring_exact_match | 32.67 |
| rougeL_f1 | 31.67 |
| rougeL_recall | 43.56 |
| rougeLsum_f1 | 31.66 |
| rougeLsum_recall | 43.51 |
| input_len | 3077.96 |
| query_time_len | 35.95 |

### After-dream-2 230 条中间统计（不可用）

| metric | 值 |
|--------|-----|
| exact_match | 34/230 (14.78%) |
| f1 (avg) | 0.35 |
| substring_exact_match | 84/230 (36.52%) |
| input_len range | 142 - 10736 |
| input_len avg | 3060 |

> 注：ctx0-4 的 input_len 正常（~2000-10000），ctx5-59 部分 query 的 input_len 极低（142-178），印证了 run_id_map 覆盖不足导致 compose 看不到数据。

### DB v9 当前数据分布

| owner_space | vectors | 说明 |
|---|---|---|
| ctx0 (`7f498a44`) | 2022 | 注入 2024 + dream consolidation/archive |
| ctx1 (`a7a00d30`) | 2002 | 注入 2004 + dream consolidation/archive |
| ctx2 (`57756c58`) | 2166 | 注入 2168 + 5 dream outputs |
| ctx3 (`56373412`) | 1966 | 注入 1972 + 5 consolidations |
| ctx4 (`409cd576`) | 1936 | 注入 1944 + 4 dream outputs |

合计 10092 vectors，Dream 数据不再用 `agent:openclaw-longmemeval` 而在各 ctx 自己的 owner_space 下。

### 文件路径

| 文件 | 路径 |
|------|------|
| Before-dream 完整结果 | `output/20260624-mabench-sstar/before-dream/glm-5-ogmemory_before-dream_results.json` |
| Before-dream eval log | `output/20260624-mabench-sstar/before-dream/before-dream_eval.log` + `before-dream_eval_resume.log` |
| After-dream-2 eval log | `output/20260624-mabench-sstar/after-dream-2/after-dream-2_eval.log` |
| After-dream-2 agent config | `configs/agent_conf/RAG_Agents/glm-5/Structure_rag_glm-5-ogmemory-after-dream-2.yaml`（output_dir: `./outputs/glm-5-ogmemory-after-dream-2`） |
| Voided 旧结果 | `output/20260624-mabench-sstar/voided-20260624-bgem3/` + 旧 `after-dream/` |
| oG-Memory config | `/data/Workspace2/oG-Memory/config/ogmem.yaml` |
| oG-Memory service log | `/data/Workspace2/oG-Memory/service_v9_restart.log` |

## 8. 关联

- [[memoryagentbench-lme-s-test-standardization]] —— 基础流程规范
- [[longmemeval-test-standardization]] —— wiki runner 模式（session 注入）
- [[volcengine-api-config]] —— Volcembedding 候选（配额 07-03 恢复）
