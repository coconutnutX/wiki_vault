---
type: decision
title: "Wiki Linking Convention"
status: active
tags: [wiki, convention, linking]
created: 2026-05-12
---

# Wiki Linking Convention

## Decision

Wiki 页面之间的关联通过两种机制表达，各有明确边界：

1. **内联双向链接**（`[[Page Name]]`）：在正文中**实际提到某概念或引用某文档的位置**直接链接，而非在文末集中列出。
2. **目录索引**（`_index.md`）：每个目录的 `_index.md` 提供该目录下所有页面的概览和一句话摘要，用于横向发现。

## Rationale

- **Why:** 文末集中列出"相关页面"是冗余的——这些关系已通过 `_index.md` 的概览和 frontmatter 的 `related` 字段表达。内联链接的优势在于精准性：读者在阅读到某个概念时可以直接跳转，而不是需要先滚到文末再找。
- **How to apply:** 新建或迁移 wiki 页面时，不在文末添加"相关 Wiki 页面"段落。相关关系通过 `_index.md` 索引和正文中的自然引用表达。
