---
title: "Git Tracking（分支/上游追踪）"
doc_type: "archive"
status: "archive"
branch: "ethan"
updated: "2026-05-03"
audience: "需要周期性同步 upstream/main，并维护 `_digested` 与源码一致性的维护者"
purpose: "集中存放 git 分支/remote/upstream 对齐锚点与同步日志，避免与文档体系历史混杂"
owns: "分支/remote 事实快照、upstream 同步日志规范与索引入口"
update_when:
  - "remote 或分支对齐策略变化时"
  - "同步日志规范需要调整时"
out_of_scope:
  - "_digested 主题内容的导航（仍以 `_digested/README.md` 与 `_digested/_context/` 为准）"
---

# Git Tracking（分支/上游追踪）

本目录只放与 git 对齐相关的元信息：当前分支/remote 快照、upstream 同步策略、每次同步日志。

## 目录

1. `git-branch-and-upstream-tracking.md` — 当前 clone 的分支/remote 快照 + 同步留痕规则
2. `upstream-sync/` — 上游同步日志（每次同步一篇，按日期归档）

## 与文档 front matter 的关系

- `upstream-sync/*.md` 是细节真相源：记录源码路径、tests、已更新文档和未覆盖风险。
- 主题文档 front matter 里的 `sync_status` 是页级摘要：让维护者或 AI 打开单篇文档时，立刻知道它是否已对齐某段 upstream 基线。
- 缺少 `sync_status` 不自动表示该文档已确认 `unknown`；通常只表示这篇文档还没纳入页级状态回填。
