---
type: session
title: "claude-obsidian Plugin Setup Notes"
status: active
tags: [plugin, obsidian, troubleshooting, setup]
created: 2026-05-11
updated: 2026-05-11
related:
  - "[[OpenClaw Memory System]]"
---

# claude-obsidian Plugin Setup Notes

安装和使用 claude-obsidian 插件过程中遇到的问题及解决方案。

## 插件信息

- **版本**: 1.6.0
- **来源**: `claude-obsidian-marketplace`（GitHub: AgriciDaniel/claude-obsidian）
- **安装路径**: `/home/aaa/.claude/plugins/cache/claude-obsidian-marketplace/claude-obsidian/1.6.0/`
- **配置**: `settings.json` → `enabledPlugins` → `claude-obsidian@claude-obsidian-marketplace: true`

## 插件数据存储路径

插件系统在 `~/.claude/plugins/` 下维护多个子目录：

```
~/.claude/plugins/
├── installed_plugins.json    # 安装记录（版本、路径、commit sha、时间戳）
├── blocklist.json            # 插件黑名单
├── install-counts-cache.json # 安装计数缓存
├── cache/                    # 插件代码缓存（按 marketplace/plugin/version 组织）
│   ├── omc/                  # oh-my-claudecode
│   ├── og-memory/            # oG-Memory
│   └── claude-obsidian-marketplace/
│       └── claude-obsidian/
│           └── 1.6.0/        # 实际的 skill 代码、模板、配置
├── data/                     # 插件运行时数据（按 plugin-id 组织）
│   └── claude-obsidian-claude-obsidian-marketplace/  # 安装后为空
├── marketplaces/             # marketplace 仓库克隆
│   └── claude-obsidian-marketplace/
└── local/                    # 本地插件
```

关键发现：
- `data/` 目录在安装后是空的，vault 结构不会自动创建在 data 中
- skill 代码从 `cache/` 目录加载，不在 `data/` 中
- `installed_plugins.json` 记录了每个插件的 `installPath`、`version`、`gitCommitSha`

## Marketplace 配置

在 `settings.json` 中注册 marketplace 源：

```json
{
  "extraKnownMarketplaces": {
    "claude-obsidian-marketplace": {
      "source": {
        "source": "github",
        "repo": "AgriciDaniel/claude-obsidian"
      }
    }
  },
  "enabledPlugins": {
    "claude-obsidian@claude-obsidian-marketplace": true
  }
}
```

## 跨仓库访问 Vault

claude-obsidian 的设计意图是：一个 vault 服务多个 Claude Code 项目。实现方式：

### 方法：全局 CLAUDE.md 指引

在 `~/.claude/CLAUDE.md` 中添加 vault 路径（对所有项目生效）：

```markdown
## Wiki Knowledge Base
Path: /data/Workspace2/wiki-vault

When you need context not already in this project:
1. Read wiki/hot.md first (recent context cache)
2. If not enough, read wiki/index.md
3. If you need domain details, read the relevant domain sub-index
4. Only then drill into specific wiki pages

Do NOT read the wiki for general coding questions or tasks unrelated to the current work.
```

### 限制

- **读取**：跨项目读取 vault 内容正常工作（通过 CLAUDE.md 路径指引 + 绝对路径）
- **写入**：claude-obsidian skill 使用相对路径（`wiki/`、`.raw/`），在非 vault CWD 下无法自动工作
- **解决方案**：
  1. 在 `wiki-vault/` 目录下启动 Claude Code（skill 全自动）
  2. 在其他项目中告诉 Claude "save this"，我手动用绝对路径写入 vault
  3. 在其他项目中告诉 Claude "ingest [file]"，我先将文件放到 `.raw/` 再执行 ingest

## 遇到的问题

### 1. `/wiki`、`/save` 输入后无反应

**现象**: 在 Claude Code 输入框直接打 `/wiki` 或 `/save`，没有任何输出。

**原因**: skill 已经注册并加载成功，但 vault 未初始化（`.raw/`、`wiki/` 等目录不存在），skill 检测不到 vault 结构后静默跳过。

**解决**: 通过 `Skill` 工具手动调用 `claude-obsidian:wiki`，按 skill 指引完成 vault scaffold 后恢复正常。

> [!key-insight] 直接输入 `/save` 可能不被 harness 识别为 skill trigger。可以告诉 Claude "save this" 或手动调用 `claude-obsidian:save`。

### 2. Vault 建在了 Workspace2 根目录

**现象**: scaffold 默认在当前工作目录创建 vault 结构（`.raw/`、`wiki/`、`.obsidian/`、`.git/`），污染了包含多个项目的 Workspace2 根目录。

**原因**: claude-obsidian skill 基于当前工作目录（CWD）操作，没有提供自定义 vault 路径的选项。

**解决**: 手动将所有 vault 文件移动到 `wiki-vault/` 子目录，删除根目录的 `.git`，在子目录重新 `git init`。

### 3. 根目录创建符号链接的方案被否决

**尝试**: 在 Workspace2 根目录创建 `wiki → wiki-vault/wiki` 等符号链接，让 skill 在根目录也能找到 vault。

**问题**: 用户不希望在根目录出现额外的文件/链接。

**解决**: 删除符号链接，改为在 `~/.claude/CLAUDE.md`（全局指令）中添加 Wiki Knowledge Base 路径指引：
```
## Wiki Knowledge Base
Path: /data/Workspace2/wiki-vault
```
这样所有项目的 Claude Code 会话都能看到 vault 路径，但 skill 仍需在 `wiki-vault/` 目录下启动才能自动工作。

### 4. `/save` 仍然不工作

**现象**: vault 已建好、CLAUDE.md 已更新路径，但直接输入 `/save` 仍然无反应。

**原因**: skill 使用相对路径（`wiki/`、`.raw/`）操作，当前 CWD 是 `/data/Workspace2` 而非 `/data/Workspace2/wiki-vault/`，skill 找不到 vault 结构。

**解决**: 通过 `Skill` 工具手动调用 `claude-obsidian:save`，我在执行时使用绝对路径写入 `/data/Workspace2/wiki-vault/wiki/` 下的文件。

### 5. `_index.md` 未同步更新

**现象**: save 操作更新了 `index.md`、`log.md`、`hot.md`，但 `wiki/concepts/_index.md` 没有更新。

**原因**: save skill 的 workflow 指引中只要求更新 index/log/hot，没有明确要求更新对应子目录的 `_index.md`。

**解决**: 手动补充。后续 save/ingest 时应确保所有相关的 `_index.md` 同步更新。

## 正确的使用方式

1. **启动位置**: 在 `/data/Workspace2/wiki-vault/` 目录下启动 Claude Code，skill 能自动找到 vault
2. **跨项目访问**: 在任何项目中通过 `~/.claude/CLAUDE.md` 的路径指引，我可以读取 vault 内容
3. **手动触发**: 告诉我 "save this" / "ingest [file]" 比直接打 `/save` 更可靠
4. **git 管理**: vault 有独立 git 仓库，remote: `git@github.com:coconutnutX/wiki_vault.git`

## 插件 Skills 列表

| Skill | 触发方式 |
|-------|---------|
| wiki | `/wiki` 或 `Skill: claude-obsidian:wiki` |
| save | `/save` 或 "save this" |
| ingest | "ingest [filename]" |
| wiki-query | "query: [question]" |
| wiki-lint | "lint the wiki" |
| autoresearch | `/autoresearch [topic]` |
| canvas | `/canvas` |
| defuddle | "defuddle [url]" |
