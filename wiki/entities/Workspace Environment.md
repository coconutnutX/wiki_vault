---
type: entity
title: "Workspace Environment"
status: active
created: 2026-05-11
updated: 2026-05-11
tags: [environment, wsl, workspace, setup]
---

# Workspace Environment

## OS & Runtime
- **平台**: WSL2 (Windows Subsystem for Linux 2)，Linux 6.6.87.2-microsoft-standard-WSL2
- **Shell**: bash
- **主工作目录**: `/data/Workspace2`

## 项目布局

```
/data/Workspace2/
├── oG-Memory/       # 主要开发项目
├── repos/           # 参考项目
│   └── openclaw/    # OpenClaw — 参考其 Memory System 实现
└── ...              # 其他项目
```

## Python 环境
- **包管理**: Conda
- **常用环境**: `conda activate py11`
- 大部分 Python 依赖装在 `py11` 虚拟环境中

## Wiki Vault (claude-obsidian)
- **WSL 路径**: `/mnt/c/Data/wiki-vault`
- **Windows 路径**: `C:\Data\wiki-vault`（供 Windows 端 Obsidian 打开）
- 跨文件系统访问：wiki 数据存在 Windows 侧，通过 `/mnt/c/` 在 WSL 中读写

## 关联
- 主项目：[[oG-Memory]]（待创建）
- 参考项目：[[OpenClaw Memory System]]
