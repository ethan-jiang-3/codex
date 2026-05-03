---
title: "_meta 索引"
doc_type: "archive"
status: "archive"
branch: "ethan"
updated: "2026-05-03"
audience: "需要追踪 `_digested` 历史演进、旧目录结构或归档记录的人"
purpose: "明确 `_meta` 是历史归档层，而不是当前导航层"
owns: "`_digested` 的历史结构、重组记录、旧路径说明和维护归档"
update_when:
  - "需要补充文档体系历史记录时"
  - "目录重组、迁移或归档策略变化时"
out_of_scope:
  - "当前推荐阅读顺序"
  - "现行主题文档的实现说明"
---

# _meta 索引

这里放 `_digested/` 自身的历史与维护元信息。

`_meta/` 是**归档层**，不参与当前主导航，也不应被当成现行文档体系的 single source of truth。

## 目录

按主题拆分子目录：

1. `doc-history/` — 文档体系历史（目录重组、旧路径映射、探索归档）
2. `git-tracking/` — 分支 / remote / upstream 对齐与同步日志（可追溯锚点）
