---
title: "Upstream Sync Log（上游同步日志）"
doc_type: "archive"
status: "archive"
branch: "ethan"
updated: "2026-05-03"
audience: "需要周期性同步上游源码基线，并维护 `_digested` 与源码一致性的维护者"
purpose: "把每次上游源码基线同步变成可追溯事件：main 对齐了什么、源码改了哪里、对 `_digested` 有何影响"
owns: "sync 日志的命名规范、记录模板、最小必填字段与检索方式"
update_when:
  - "同步流程或记录模板需要调整时"
  - "新增更细的归档策略时"
out_of_scope:
  - "逐条复述 upstream 的代码变更内容（应该引用 commit/PR）"
  - "当前 `_digested` 的主导航（仍以 `_digested/README.md` 与 `_digested/_context/` 为准）"
---

# Upstream Sync Log（上游同步日志）

目标：我们会"时不时同步上游源码基线"。为了避免 `_digested` 和源码基线脱节，每次同步都写一篇日志，固定落在本目录。

口径：这里的主指标是 `main` 相对上游源码基线（优先 `upstream/main`，未配置 upstream 时为当前约定的远端基线，例如 `origin/main`）的变化。`ethan` 预期会长期领先 `main`，因为它承载 `_digested/` 下的源码消化与归档提交；这个 ahead 数只作为定位文档版本的背景信息，不作为 upstream 漂移指标。

---

## 命名规范（每次同步一篇）

- 文件名：`YYYY-MM-DD-upstream-sync.md`
- 例如：`2026-05-03-upstream-sync.md`

如果一天多次同步，在文件名后追加序号：
- `2026-05-03-upstream-sync-2.md`

---

## 最小记录项（强制）

每次同步日志至少要回答清楚下面这些问题：

1. 同步前后上游源码基线与本地 `main` 各指向哪个 commit（短 hash + 1 行 subject）。
2. `main` 的同步方式：fast-forward / merge / rebase（以及为什么）。
3. 本次上游源码变化涉及哪些路径（排除 `_digested/`）；如果发生冲突，列出冲突路径。
4. `_digested` 需要跟进的主题以及对应文档路径。
5. `ethan` 的 commit 锚点（仅用于定位 `_digested` 文档版本，不用 ahead 数判断同步风险）。

如果某个 owner 文档已经完成正式复核，推荐同时把页级摘要状态写回该文档 front matter 的 `sync_status`；但这不替代本目录日志，因为源码/tests 证据和未覆盖风险仍应留在 sync log。

---

## 推荐附加项（可选但很有价值）

- 影响面评估：这次上游源码变更对"agent 核心、工具面、TUI、sandbox、skills"是否有直接影响。
- 回归抓手：建议跑哪些 tests（只列路径/命令，不要粘贴长输出）。
- 迁移记录：如果 `_digested` 做了目录重组/编号调整，把"旧路径 -> 新路径"记录到 `_meta/doc-history/`。

## 大版本同步要分阶段

如果 upstream delta 很大，不要假装一次同步就能把 `_digested` 全部消化完：

1. 先记录 git 锚点、release note 线索和高层复核方向。
2. 后续按主题逐步深挖：release note 定位方向，源码/tests 确认事实，最后只更新对应 owner 文档。
3. 每完成一个主题复核，再回到该次 sync log 追加阶段结果。

---

## 模板

使用 `TEMPLATE.md` 作为起点创建新日志文件：
- `cp _meta/git-tracking/upstream-sync/TEMPLATE.md _meta/git-tracking/upstream-sync/YYYY-MM-DD-upstream-sync.md`
