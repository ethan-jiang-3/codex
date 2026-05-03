---
title: "Upstream Sync — YYYY-MM-DD"
doc_type: "archive"
status: "archive"
branch: "ethan"
updated: "YYYY-MM-DD"
audience: "维护者"
purpose: "记录一次上游源码基线同步事件的对齐锚点，并产出可执行的 main 对齐与 `_digested` 跟进 TODO"
owns: "本次同步的事实快照、main 对齐结果、源码变化路径、以及 `_digested` 跟进清单"
update_when:
  - "需要补充同步后续跟进项时"
out_of_scope:
  - "逐条复述代码变更内容（引用 commit/PR 即可）"
---

# Upstream Sync — YYYY-MM-DD

口径：主指标是 `main` 相对上游源码基线的变化。`ethan` 预期会长期领先 `main`，因为它承载 `_digested/` 下的源码消化与归档提交。

## 摘要

- 上游源码基线：`upstream/main|origin/main|<other remote/ref>`
- 同步方式：`ff|merge|rebase`（把本地 `main` 对齐到上游源码基线的方式）
- 是否冲突：`yes|no`
- 主要影响面：`agent-core|tools|tui|skills|models|state|sandbox|...`
- 这次日志的目标：把"上游源码变化后 `_digested` 要复核什么"变成明确 TODO

## 对齐锚点（同步前 -> 同步后）

| Ref | Before | After |
|---|---|---|
| `<upstream baseline ref>` | `<hash> <subject>` | `<hash> <subject>` |
| `main` | `<hash> <subject>` | `<hash> <subject>` |
| `ethan`（`_digested` 文档版本锚点） | `<hash> <subject>` | `<hash> <subject>` |

## 操作记录（可选：命令级，短）

```bash
git fetch <remote>
git checkout main
git pull --ff-only <remote> main
git checkout ethan
git merge main
```

## 上游源码变化路径（排除 `_digested/`）

- `path/to/source`
- `path/to/tests`

## 冲突与解决（如有）

- 涉及路径：
- 解决策略（1-2 句）：

## TODO：`main` 对齐计划（同步后必须做）

- [ ] 确认上游源码基线：`upstream/main` / `origin/main` / `<other ref>`
- [ ] 决定 `main` 对齐方式：`ff` / `merge` / `rebase`
- [ ] 完成 `main` 对齐（记录结果 commit）
- [ ] 如果有冲突：列出冲突路径与选择原则
- [ ] 基线验证（至少选一种）：
  - [ ] 跑相关 tests
  - [ ] 只做 smoke check（写明为什么可以）

## TODO：`ethan` / `_digested` 承载计划

- [ ] 记录当前 `ethan` commit
- [ ] 决定是否需要 `merge main -> ethan` / `rebase ethan on main`
- [ ] 如果执行吸纳：记录结果 commit

## TODO：`_digested` 跟进清单

### 需要复核的主题（打勾 + 写清受影响原因）

- [ ] Agent core / loop / identity（`codex-rs/core/` / `03-agent-core/`）
- [ ] Tools / exec / sandbox（`codex-rs/tools/` / `04-tools-execution-and-sandboxing/`）
- [ ] Skills / plugins（`codex-rs/skills/` / `05-skills-and-plugins/`）
- [ ] Models / providers（`codex-rs/model-provider/` / `06-models-and-providers/`）
- [ ] State / memory / threads（`codex-rs/state/` / `07-state-memory-and-sessions/`）
- [ ] TUI / CLI / app-server（`codex-rs/tui/` / `08-tui-cli-and-integration/`）
- [ ] MCP / integration / protocol（`codex-rs/codex-mcp/` / `08-tui-cli-and-integration/`）
- [ ] Security / auth / secrets（`codex-rs/secrets/` / security 相关）

### 需要更新的文档（列出具体路径 + 变更类型）

- [ ] `_digested/.../...md` — `fix facts|add section|update diagram|move path`

### Source-of-truth 抓手

- Baseline:
  - [ ] `<upstream baseline ref>`：`<hash> <subject>`
  - [ ] `main`：`<hash> <subject>`
- Code:
  - [ ] `path/to/code.rs`
- Tests:
  - [ ] `tests/...`

## 备注（可选）

- 如果这次同步导致 `_digested` 目录结构/编号调整：在 `_meta/doc-history/` 里补一条迁移说明。
