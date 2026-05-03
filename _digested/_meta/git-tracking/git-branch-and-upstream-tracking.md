---
title: "Git Branch & Upstream Tracking（分支/上游追踪）"
doc_type: "archive"
status: "archive"
branch: "ethan"
updated: "2026-05-03"
audience: "需要长期维护 `_digested/`，并持续对齐源码演进的人"
purpose: "记录 Codex 仓库的源码基线、分支/remote 结构与 `_digested` 复核锚点，避免口口相传导致的漂移"
owns: "main 与上游源码基线的关系、远端指向（origin/upstream）、ethan 作为 `_digested` 工作分支的角色、以及每次对齐时的可追溯锚点"
update_when:
  - "切换工作分支或重置分支基线时"
  - "新增/变更 remote（origin/upstream）或其默认分支时"
  - "需要记录一次关键的 `_digested` 与源码对齐节点时"
out_of_scope:
  - "具体代码变更内容（那应在 commit message / PR / release notes）"
  - "主题文档本身的导航（仍以 `_digested/README.md` 与 `_digested/_context/` 为准）"
---

# Git Branch & Upstream Tracking（分支/上游追踪）

定位：`_digested/` 会持续更新，而源代码也会持续变化。目标不是"记住 git 命令"，也不是把 `ethan` 领先 `main` 的提交数量当成风险指标，而是确保两件事一直成立：

1. `main` 能持续对齐上游源码基线（否则 `_digested` 会逐步与真实源码脱节）。
2. `_digested` 的关键结论可追溯（能回答"当时对齐哪条 `main` / `upstream` 得出的结论"）。

> 口径：`ethan` 预期会长期领先 `main`，因为它承载大量 `_digested/` 下的源码消化与归档提交。只要这些领先提交主要限于 `_digested/`，它们不是源码分叉风险，也不应作为 upstream 漂移指标。真正需要优先关注的是 `main` 相对上游源码基线的差异。

---

## 当前约定

- 当前工作分支：`ethan`
- `ethan` 继承 `main`，但主要用于提交 `_digested/` 下的源码消化结果
- `main` 代表本 clone 的源码基线；它应周期性对齐真正的上游 `main`
- `_meta` 用来保存这些"对齐关系与历史"的归档记录
- `ethan` 相对 `main` 的 ahead 数只是背景事实；主指标是 `main` 相对上游源码基线的差异

---

## 当前工作区快照（2026-05-03）

在本机 `/Users/bowhead/codex` 的实际 git 状态如下：

### Branches

- 当前分支：`ethan`
- `ethan` HEAD：`35aaa5d9f Bound websocket request sends with idle timeout (#20751)`
- `main` HEAD：`35aaa5d9f Bound websocket request sends with idle timeout (#20751)`
- 当前 `ethan` 与 `main` 指向同一 commit（消化文档工作尚未开始）

### Remotes

- 存在的 remote：`origin`
  - `origin` URL：`git@github.com:ethan-jiang-3/codex.git`
- **未配置 `upstream` remote**
  - 结论：在此 clone 中，`main` 的对齐锚点是 `origin/main`

### Tracking / Divergence

- `main` 跟踪：`origin/main`
- `main...origin/main`：`0 / 0`（`main` 与当前远端跟踪分支一致）
- `ethan` 相对 `main`：当前指向同一 commit（消化工作尚未产生 ahead 提交）

---

## 需要 "upstream" 的话怎么做（可选）

如果你期望存在一个真正的上游 remote（例如 zed-industries/codex），建议在本 clone 里显式添加：

```bash
git remote add upstream <UPSTREAM_REPO_URL>
git fetch upstream
git branch -u upstream/main main
```

添加后，把：
- `upstream` 的 URL
- `upstream/main` 的 commit
- `main` 当前跟踪关系
- `main...upstream/main` 的差异计数
- `upstream/main -> main` 之间涉及的源码路径（排除 `_digested/`）

补充到本文件的"当前工作区快照"里。

---

## 每次更新 `_digested` 的建议记录项（最小集合）

当你觉得某次 `_digested` 更新和源码版本强绑定，建议在本文件追加一段"对齐节点"记录：

- 日期（YYYY-MM-DD）
- 上游源码基线的 commit（优先 `upstream/main`；未配置 upstream 时记录 `origin/main`）
- 本地 `main` 的 commit（短 hash + subject）
- 触发复核的源码路径（排除 `_digested/`）
- `ethan` 的 commit（短 hash + subject，仅用于定位 `_digested` 文档版本）
- remote 的事实（origin/upstream 是否存在 + 指向）

---

## 同步 upstream 的日志策略

由于 upstream 会持续变化，把"每一次同步事件"拆成独立日志文件：

- 本文件只负责：当前 clone 的分支/remote 事实快照、同步与留痕的规则
- 每次 upstream 同步都新增一篇日志：
  - 目录：`_meta/git-tracking/upstream-sync/`
  - 文件名：`YYYY-MM-DD-upstream-sync.md`

入口与模板：
- `_meta/git-tracking/upstream-sync/README.md`
- `_meta/git-tracking/upstream-sync/TEMPLATE.md`

这类日志的核心不是"记录操作命令"，而是：
- 明确 `main` 是否/如何吸纳新的上游源码基线
- 列出本次上游源码变化涉及的路径（排除 `_digested/`）
- 把 `_digested` 的复核点写成 checklist
- 记录 `ethan` 如何承载这次 `_digested` 更新
