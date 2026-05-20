---
name: promptfoo-extraction-testing
description: 使用 promptfoo 为 oG-Memory extraction 模块搭建测试框架的决策
tags:
  - testing
  - extraction
  - promptfoo
  - decision
created: 2026-05-20
---

# Promptfoo Extraction Testing Framework

## 背景

需要为 oG-Memory 的 react loop extraction 模块搭建测试框架，用于：
- 对比 Eager 和 Lazy 模式的抽取效果
- 管理易错的边缘案例测试集
- 回归测试：优化代码后快速验证

## 选项评估

| 方案 | 优点 | 缺点 |
|------|------|------|
| **纯 pytest** | 直接、无依赖 | 缺少 Web UI、需自己写对比工具 |
| **Promptfoo** | YAML 测试用例管理、Web UI、回归测试、换数据集方便 | 需要 npm、wrapper 脚本 |

## 决策

选择 **Promptfoo + Python script provider**。

### 关键技术点

1. **Provider 类型**：使用 `exec:` 而非 `script:`
   ```yaml
   providers:
     - id: exec:python ./promptfoo_wrapper.py
       config:
         mode: eager
   ```

2. **数据传递**：promptfoo 通过 argv 传递数据，而非 stdin
   - argv[1]: prompt text
   - argv[2]: {"config": {...}, "env": {...}}
   - argv[3]: {"vars": {...}, "test": {...}}

3. **串行运行**：使用 `--max-concurrency 1` 避免 PostgreSQL schema 死锁
   - 原因：`ensure_schema()` 的进程级锁无法防止多进程并发 DDL

4. **真实数据库**：wrapper 使用 PostgreSQL 而非 Mock storage
   - Lazy 模式需要真实存储供 ReAct loop 的 read 工具使用

### 目录结构

```
tests/promptfoo/
├── promptfoo_wrapper.py    # Python wrapper，调用 Extractor
├── promptfooconfig.yaml    # 主配置
├── cases/*.yaml            # 测试案例
├── results/                # 结果输出
└── README.md
```

### 使用方法

```bash
cd tests/promptfoo
promptfoo eval --max-concurrency 1
promptfoo view  # Web UI 对比结果
```

## 相关

- [[lazy-mode-empty-db-fix]]
- [Promptfoo Documentation](https://promptfoo.dev/docs/getting-started)

## 后续

- 考虑修复 PostgreSQL schema 多进程死锁问题
- 扩展测试案例覆盖更多边缘场景