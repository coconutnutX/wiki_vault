---
name: longmemeval-result-comparison-xlsx-guide
description: LongMemEval 测试结果对比 XLSX 生成流程与踩坑记录
metadata:
  type: reference
---

# LongMemEval 测试结果对比 XLSX 生成指南

从 before-dream / after-dream QA 的 `details.jsonl` 和 DeepDream 结果生成完整对比 XLSX 的操作流程，包括过程中的修正细节。

关联：[[ogmemory-deepdream-longmemeval-e2e-runbook]]、[[longmemeval-dream36-test-report]]

## 1. 数据源文件

生成对比表需要 3 个数据源：

| 数据源 | 路径模板 | 内容 |
|--------|---------|------|
| before-dream QA | `output/0611-before-dream-v2-{run_id}/details.jsonl` | compose 检索结果、reader 回答、judge 判分 |
| after-dream QA | `output/0611-after-dream-{run_id}/details.jsonl` | 同上，dream 后 |
| DeepDream | `output/0611-dream-36-ingest/deepdream_results.json` | dream 状态、iterations、created_uris |

**注意路径前缀**：output 目录可能有日期前缀（如 `0611-`），具体路径要看 `ls output/` 确认。

## 2. 生成脚本

将以下 Python 脚本保存为 `scripts/generate_comparison_xlsx.py`，或直接在 `cd /data/Workspace2/repos/longmemeval` 后运行：

```python
import json, re
from openpyxl import Workbook
from openpyxl.styles import Font, PatternFill, Alignment, Border, Side

# ============================================================
# 配置区：每次测试后修改这里
# ============================================================
BD_DIR = 'output/0611-before-dream-v2-84325a8b'    # before-dream QA 输出目录
AD_DIR = 'output/0611-after-dream-84325a8b'         # after-dream QA 输出目录
DD_DIR = 'output/0611-dream-36-ingest'              # dream 输出目录
DATA_FILE = 'data/longmemeval_s_dream_36.json'      # 原始数据文件
OUT_FILE = 'output/dream36-full-comparison.xlsx'    # 输出 XLSX 路径

# ============================================================
# 数据加载
# ============================================================
with open(f'{BD_DIR}/details.jsonl') as f:
    bd = {json.loads(l)['question_id']: json.loads(l) for l in f}

with open(f'{AD_DIR}/details.jsonl') as f:
    ad = {json.loads(l)['question_id']: json.loads(l) for l in f}

with open(f'{DD_DIR}/deepdream_results.json') as f:
    dd = json.load(f)
dd_map = {r.get('question_id', ''): r for r in dd}

# ============================================================
# evidence_diff：只显示差异条目
# ============================================================
def diff_evidence(bd_ev, ad_ev):
    """Compare before/after retrievedEvidence by full text content.

    关键设计：用全文内容做 set 对比，而非按 key 匹配。
    因为 compose 每次检索 top-15，dream 前后顺序和内容可能完全不同，
    按 key（如第一行摘要）匹配会把所有条目都标记为 REMOVED/ADDED，
    而实际上很多条目只是换了个位置或措辞略有变化。

    用 set(full_text) 对比：完全相同的条目不算差异，只有真正消失或新增的才算。
    """
    def parse_items(text):
        """拆分出单个记忆条目，跳过 header 行。"""
        if not text.strip():
            return []
        # 按 "- [category]" pattern 分割
        parts = re.split(r'\n(?=- \[)', text)
        items = []
        for p in parts:
            p = p.strip()
            if not p or p.startswith('##'):
                continue  # 跳过 "## Retrieved Memories" 等 header
            items.append(p)
        return items

    bd_items = parse_items(bd_ev)
    ad_items = parse_items(ad_ev)

    bd_set = set(bd_items)
    ad_set = set(ad_items)

    removed = sorted(bd_set - ad_set)
    added = sorted(ad_set - bd_set)
    same_count = len(bd_set & ad_set)

    if not removed and not added:
        return f"All {same_count} items identical"

    lines = [f"Same: {same_count}, Removed: {len(removed)}, Added: {len(added)}"]

    if removed:
        lines.append("\n--- REMOVED after dream ---")
        for r in removed:
            lines.append(r[:300])  # 截断避免单元格过大

    if added:
        lines.append("\n--- ADDED after dream ---")
        for a in added:
            lines.append(a[:300])

    return "\n".join(lines)
```

> **踩坑 1：`parse_items` 必须跳过 `##` 开头的 header 行。**
> 最初的版本没有跳过 header，把 `## Retrieved Memories` 当成了条目，
> 导致 key 匹配时所有条目的 key 都撞上了 header，diff 函数输出全是 "No difference"。

> **踩坑 2：用全文做 set 对比，而不是按 key 匹配。**
> 第一版 diff 按 `[{category}] {first_line_80chars}` 做 key，
> 但 compose top-15 在 dream 前后顺序完全不同，几乎所有 key 都变了，
> 导致把全部条目标记为 REMOVED + ADDED，实际很多只是换了位置。
> 改为 `set(full_text)` 后，28/36 题 correctly 显示 "All items identical"。

## 3. XLSX 列结构

| 列组 | 列名 | 含义 |
|------|------|------|
| **题目** | `qid`, `question_type`, `question`, `expected_answer`, `user_id` | 题目基本信息 |
| **Dream结果** | `dream_status`, `dream_iterations`, `dream_uris_created`, `dream_tools_used`, `dream_elapsed_s`, `dream_error` | DeepDream 运行详情 |
| **Before-dream stats** | `bd_result`, `bd_hypothesis`, `bd_hit_count`, `bd_hit_categories`, `bd_compose_elapsed_s`, `bd_retrieved_chars`, `bd_cross_user_leak` | compose 检索统计 |
| **Before-dream content** | `bd_identity_context`, `bd_session_context`, `bd_episodic_context`, `bd_retrieved_evidence` | compose 各层检索原文 |
| **After-dream stats** | `ad_result`, `ad_hypothesis`, `ad_hit_count`, ... | 同 before，dream后 |
| **After-dream content** | `ad_identity_context`, `ad_session_context`, `ad_episodic_context`, `ad_retrieved_evidence` | 同 before，dream后 |
| **Diff** | `evidence_diff` | before/after retrievedEvidence 的差异对比 |
| **变化** | `result_change` | GAINED / LOST / 空 |

## 4. 样式与格式细节

### 不要用 CSV，要用 XLSX

> **踩坑 3：CSV 打不开。**
> `retrieved_evidence` 等字段包含大量多行文本（换行符、引号、`&#x27;` 等 HTML 转义字符），
> Excel/WPS 无法正确解析多行 CSV。XLSX 原生支持多行文本单元格，直接双击能打开。

### 不要冻结行

> **踩坑 4：用户反馈冻结行不方便。**
> 第一版用了 `ws.freeze_panes = 'F2'`（冻结首行+前5列），用户要求去掉。
> 直接删除 `freeze_panes` 设置即可。

### 颜色标注

```python
# CORRECT = 绿底, WRONG = 红底
gained_fill = PatternFill(start_color="C6EFCE", fill_type="solid")
lost_fill = PatternFill(start_color="FFC7CE", fill_type="solid")

# result_change 列: LOST = 红底+红粗字, GAINED = 绿底+绿粗字
lost_font = Font(bold=True, color="9C0006")
gained_font = Font(bold=True, color="006100")

# evidence_diff 列: 有差异的 = 黄底高亮
yellow_fill = PatternFill(start_color="FFFF00", fill_type="solid")
```

### 跨用户泄漏检测

```python
# 检测 retrievedEvidence + sessionContext + episodicContext 中
# 是否出现了其他 user_id 的 suffix
uid = str(b.get('user_id', ''))          # 如 dream-36-84325a8b-4f54b7c9
my_suffix = uid.rsplit('-', 1)[-1]        # 如 4f54b7c9

all_text = retrieved_evidence + session_context + episodic_context
found = re.findall(r'dream-36-84325a8b-(\w+)', all_text)
leak_users = set(u for u in found if u != my_suffix)
```

> **注意**：跨用户泄漏有两层——vector 层（已修复）和 agent case 层（残留）。
> 详见 [[longmemeval-dream36-test-report]] §4。

## 5. Reader/Judge API 配置

> **踩坑 5：API key / base_url 变化导致 QA eval 失败。**
> 环境变量 `OPENAI_API_KEY` 和 `OPENAI_BASE_URL` 可能随时变化，
> 而 `run_qa_eval.py` 默认读环境变量。

**当前使用的 API**：

| 参数 | 值 |
|------|-----|
| API Key | `beb569ea-db9e-47a3-8d40-ff3fcbabd9cf` |
| Base URL | `https://ark.cn-beijing.volces.com/api/coding/v3` |
| Model | `doubao-seed-2-0-code-preview-260215` |

**传入方式**：通过环境变量覆盖（不要用 `--reader-base-url` 参数，脚本可能不支持或行为不一致）：

```bash
OPENAI_BASE_URL=https://ark.cn-beijing.volces.com/api/coding/v3 \
OPENAI_API_KEY=beb569ea-db9e-47a3-8d40-ff3fcbabd9cf \
python scripts/run_qa_eval.py \
  --reader-model doubao-seed-2-0-code-preview-260215 \
  --judge-model doubao-seed-2-0-code-preview-260215 \
  ...
```

> **踩坑 6：`gpt-4o-mini` 在火山引擎 API 上不支持。**
> 火山引擎返回 `UnsupportedModel` 错误。必须用 `doubao-seed-2-0-code-preview-260215`。

> **踩坑 7：`chatapi.littlewheat.com/v1` 的 API key 和火山引擎不同。**
> 如果环境变量里的 `OPENAI_API_KEY` 是火山引擎的 key，
> 但手动指定了 `--reader-base-url=https://chatapi.littlewheat.com/v1`，
> 会出现 401 Invalid Token。两者必须配套使用。

## 6. 数据查看辅助

### 从 SQL 看某个 user 的抽取内容

数据库中 `context_nodes`、`session_archives`、`relation_edges` 可能均为空，所有数据在 `vector_index` 表。

连接：`host=127.0.0.1 port=5432 dbname=ogmemory user=ogmem password=ogmem123`

```sql
-- 查某个 user 的记忆按 category 分组统计
SELECT metadata->>'category' as category,
       metadata->>'level' as level,
       count(*) as count,
       sum(length(text)) as total_chars
FROM vector_index
WHERE filters->>'owner_space' = 'user:dream-36-84325a8b-{QID}'
GROUP BY 1, 2 ORDER BY 1, 2;

-- 查某个 user 的 entity 摘要
SELECT uri, text
FROM vector_index
WHERE filters->>'owner_space' = 'user:dream-36-84325a8b-{QID}'
  AND metadata->>'category' = 'entity'
  AND metadata->>'level' = 'abstract'
ORDER BY uri;

-- 查某个记忆的完整 content
SELECT uri, text
FROM vector_index
WHERE filters->>'owner_space' = 'user:dream-36-84325a8b-{QID}'
  AND metadata->>'category' = 'entity'
  AND metadata->>'level' = 'content'
  AND uri LIKE '%antique%'
ORDER BY uri;

-- 查 agent 空间 case 记忆（跨用户泄漏来源）
SELECT uri, text
FROM vector_index
WHERE filters->>'owner_space' = 'agent:openclaw-longmemeval'
  AND metadata->>'category' = 'case'
  AND metadata->>'level' = 'content'
  AND text LIKE '%{QID}%';
```

记忆结构：每个抽取单元有 abstract + content 两层（`metadata->>'level'` 区分），URI 格式：
- abstract: `ctx://{account}/users/{uid}/memories/{category}/{name}`
- content: `ctx://{account}/users/{uid}/memories/{category}/{name}/content.md`

### 从 details.jsonl 看某题完整链路

```python
import json
with open('output/0611-before-dream-v2-{run_id}/details.jsonl') as f:
    for line in f:
        d = json.loads(line)
        if d['question_id'] == '{QID}':
            print(d['memory_context']['identityContext'])     # Profile 摘要
            print(d['memory_context']['sessionContext'])      # Session 状态
            print(d['memory_context']['retrievedEvidence'])   # 检索到的记忆原文（核心）
            print(d['hypothesis'])                            # reader 回答
            print(d['result'])                                # judge 判定
            print(d['expected'])                              # 正确答案
            print(d['compose_stats'])                         # compose 检索统计
```

## 7. 完整生成脚本（可直接运行）

```python
import json, re
from openpyxl import Workbook
from openpyxl.styles import Font, PatternFill, Alignment, Border, Side

# ============ 配置区 ============
BD_DIR = 'output/0611-before-dream-v2-84325a8b'
AD_DIR = 'output/0611-after-dream-84325a8b'
DD_DIR = 'output/0611-dream-36-ingest'
OUT_FILE = 'output/dream36-full-comparison.xlsx'

# ============ 加载 ============
with open(f'{BD_DIR}/details.jsonl') as f:
    bd = {json.loads(l)['question_id']: json.loads(l) for l in f}
with open(f'{AD_DIR}/details.jsonl') as f:
    ad = {json.loads(l)['question_id']: json.loads(l) for l in f}
with open(f'{DD_DIR}/deepdream_results.json') as f:
    dd = json.load(f)
dd_map = {r.get('question_id', ''): r for r in dd}

# ============ diff 函数 ============
def diff_evidence(bd_ev, ad_ev):
    def parse_items(text):
        if not text.strip(): return []
        parts = re.split(r'\n(?=- \[)', text)
        items = []
        for p in parts:
            p = p.strip()
            if not p or p.startswith('##'): continue  # 跳过 header
            items.append(p)
        return items

    bd_set = set(parse_items(bd_ev))
    ad_set = set(parse_items(ad_ev))
    removed = sorted(bd_set - ad_set)
    added = sorted(ad_set - bd_set)
    same = len(bd_set & ad_set)

    if not removed and not added:
        return f"All {same} items identical"

    lines = [f"Same: {same}, Removed: {len(removed)}, Added: {len(added)}"]
    if removed:
        lines.append("\n--- REMOVED after dream ---")
        for r in removed: lines.append(r[:300])
    if added:
        lines.append("\n--- ADDED after dream ---")
        for a in added: lines.append(a[:300])
    return "\n".join(lines)

# ============ 样式 ============
hdr_font = Font(bold=True, size=11, color="FFFFFF")
hdr_fill = PatternFill(start_color="4472C4", fill_type="solid")
wrap = Alignment(wrap_text=True, vertical="top")
bdr = Border(left=Side('thin'), right=Side('thin'),
             top=Side('thin'), bottom=Side('thin'))
ok_fill = PatternFill(start_color="C6EFCE", fill_type="solid")
ng_fill = PatternFill(start_color="FFC7CE", fill_type="solid")
diff_fill = PatternFill(start_color="FFFF00", fill_type="solid")

wb = Workbook()
ws = wb.active
ws.title = "Dream36 Comparison"

headers = [
    'qid','question_type','question','expected_answer','user_id',
    'dream_status','dream_iterations','dream_uris_created','dream_tools_used','dream_elapsed_s','dream_error',
    'bd_result','bd_hypothesis','bd_hit_count','bd_hit_categories','bd_compose_elapsed_s','bd_retrieved_chars','bd_cross_user_leak',
    'bd_identity_context','bd_session_context','bd_episodic_context','bd_retrieved_evidence',
    'ad_result','ad_hypothesis','ad_hit_count','ad_hit_categories','ad_compose_elapsed_s','ad_retrieved_chars','ad_cross_user_leak',
    'ad_identity_context','ad_session_context','ad_episodic_context','ad_retrieved_evidence',
    'evidence_diff','result_change',
]

for ci, h in enumerate(headers, 1):
    c = ws.cell(row=1, column=ci, value=h)
    c.font = hdr_font; c.fill = hdr_fill; c.alignment = wrap; c.border = bdr

row = 2
for qid in sorted(bd):
    b, a, d = bd[qid], ad[qid], dd_map.get(qid, {})
    uid = str(b.get('user_id',''))
    my = uid.rsplit('-',1)[-1] if uid else ''
    bc, ac = b.get('memory_context',{}), a.get('memory_context',{})

    # 跨用户泄漏
    def find_leak(ctx, my):
        t = ctx.get('retrievedEvidence','') + ctx.get('sessionContext','') + ctx.get('episodicContext','')
        return ', '.join(sorted(set(u for u in re.findall(r'dream-36-84325a8b-(\w+)', t) if u != my)))

    br, ar = b.get('result',''), a.get('result','')
    ch = 'LOST' if br=='CORRECT' and ar=='WRONG' else 'GAINED' if br=='WRONG' and ar=='CORRECT' else ''
    ev = diff_evidence(bc.get('retrievedEvidence',''), ac.get('retrievedEvidence',''))

    vals = [
        qid, b.get('question_type',''), b.get('question',''), str(b.get('expected','')), uid,
        d.get('status',''), d.get('iterations',''), len(d.get('created_uris',[])),
        ', '.join(d.get('tools_used',[])), d.get('elapsed_s',''), d.get('error',''),
        br, b.get('hypothesis',''), b.get('hit_count',0),
        ', '.join(b.get('compose_stats',{}).get('hit_categories',[])),
        round(b.get('compose_elapsed_ms',0)/1000,1), b.get('retrieved_chars',0), find_leak(bc,my),
        bc.get('identityContext',''), bc.get('sessionContext',''), bc.get('episodicContext',''), bc.get('retrievedEvidence',''),
        ar, a.get('hypothesis',''), a.get('hit_count',0),
        ', '.join(a.get('compose_stats',{}).get('hit_categories',[])),
        round(a.get('compose_elapsed_ms',0)/1000,1), a.get('retrieved_chars',0), find_leak(ac,my),
        ac.get('identityContext',''), ac.get('sessionContext',''), ac.get('episodicContext',''), ac.get('retrievedEvidence',''),
        ev, ch,
    ]

    for ci, v in enumerate(vals, 1):
        c = ws.cell(row=row, column=ci, value=v)
        c.alignment = wrap; c.border = bdr
        if ci==12: c.fill = ok_fill if v=='CORRECT' else ng_fill if v=='WRONG' else PatternFill()
        if ci==23: c.fill = ok_fill if v=='CORRECT' else ng_fill if v=='WRONG' else PatternFill()
        if ci==33 and v and not v.startswith("All"): c.fill = diff_fill
        if ci==34:
            if v=='LOST': c.fill = ng_fill; c.font = Font(bold=True, color="9C0006")
            elif v=='GAINED': c.fill = ok_fill; c.font = Font(bold=True, color="006100")
    row += 1

# 列宽
widths = [16,18,45,30,35,10,10,10,40,10,10,10,35,8,30,10,8,15,30,30,30,50,10,35,8,30,10,8,15,30,30,30,50,60,10]
for i, w in enumerate(widths):
    ws.column_dimensions[chr(65+i) if i<26 else chr(64+i//26)+chr(65+i%26)].width = w
for r in range(2, row): ws.row_dimensions[r].height = 80

wb.save(OUT_FILE)
print(f'Written: {OUT_FILE} ({row-2} rows)')
```

每次测试后只需修改配置区的 5 个路径，然后运行即可。

## 8. 踩坑总结

| # | 问题 | 原因 | 修复 |
|---|------|------|------|
| 1 | evidence_diff 全显示 "No difference" | `parse_items` 没跳过 `##` header 行，header 被当成条目导致 key 冲突 | `if p.startswith('##'): continue` |
| 2 | evidence_diff 把所有条目标记为 REMOVED+ADDED | 按 key（第一行摘要）匹配，compose top-15 前后顺序不同导致全部 key 变了 | 改为 `set(full_text)` 全文对比 |
| 3 | CSV 文件下载后打不开 | `retrieved_evidence` 包含大量多行文本+特殊字符，Excel 无法解析 | 改用 XLSX（`openpyxl`） |
| 4 | 冻结行不方便浏览 | `freeze_panes='F2'` 锁定了首行+前5列 | 删除 freeze_panes 设置 |
| 5 | QA eval 报 401 Invalid Token | 环境变量 `OPENAI_API_KEY` 是火山引擎 key，但手动指定了 `chatapi.littlewheat.com` | 两者必须配套，用环境变量统一覆盖 |
| 6 | QA eval 报 UnsupportedModel | `gpt-4o-mini` 在火山引擎 API 不支持 | 改用 `doubao-seed-2-0-code-preview-260215` |
| 7 | output 目录有 `0611-` 前缀找不到文件 | 后来批量重命名加了日期前缀 | 用 `ls output/` 确认实际路径再填入配置区 |

## 相关文档

- [[ogmemory-deepdream-longmemeval-e2e-runbook]] — 端到端测试操作手册（注入、QA、dream 流程）
- [[longmemeval-dream36-test-report]] — dream_36 测试完整结果与分析