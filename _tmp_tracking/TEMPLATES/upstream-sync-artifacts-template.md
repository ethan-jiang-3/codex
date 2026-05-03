---
title: "Upstream Sync Artifacts — Templates"
date: "YYYY-MM-DD"
status: "reference"
scope: "Templates for the four upstream-sync tracking artifacts"
---

# Upstream Sync Artifacts — Templates

本文件包含四类上游同步追踪工件的模板：
1. Triage（分类拆题）
2. Owner Map（源码→文档映射）
3. Coverage Map（已看/未看台账）
4. Progressive Sync Plan（渐进同步计划）

使用顺序：release note / upstream delta → Triage → Owner Map → Coverage Map → Sync Plan → 执行阶段切片。

---

## 1) Triage Table

```markdown
---
title: "RELEASE_vX.Y.Z Triage for _digested Sync"
date: "YYYY-MM-DD"
status: "triage_only"
scope: "Release-note indexed source delta from <before_hash> to <after_hash>"
---

# RELEASE_vX.Y.Z Triage for `_digested` Sync

目的：把 release note / changelog 先拆成可追踪主题。本文只做 triage，不更新 `_digested` 正文。

使用口径：
- 每个主题后续都要回到源码和 tests 复核。
- 优先级按"运行时行为变化风险"排序，不按 release note 篇幅排序。

## Triage Table

| Priority | Theme | Source clue | Candidate source paths | Candidate tests | Candidate `_digested` owner docs | Notes |
|---|---|---|---|---|---|---|
| P0 | ... | ... | `codex-rs/...` | `tests/...` | `_digested/0X-.../...md` | ... |
| P1 | ... | ... | ... | ... | ... | ... |
| P2 | ... | ... | ... | ... | ... | ... |
```

## 2) Owner Map

```markdown
---
title: "vX.Y Source Path Owner Map for _digested"
date: "YYYY-MM-DD"
status: "owner_map_only"
scope: "Source delta from <before> to <after>, excluding _digested/"
---

# vX.Y Source Path Owner Map for `_digested`

目的：给后续分阶段同步建立"源码路径 → owner 文档"定位表。本文只做 owner map，不更新正文。

使用口径：
- 先按路径找到 owner，再读源码/tests 验证。
- 一个路径可能有主 owner 和副 owner。

## Owner Map

| Source path / pattern | Primary `_digested` owner | Secondary owner / notes | Triage reason |
|---|---|---|---|
| `codex-rs/core/**` | `03-agent-core/01-agent-loop.md` | `03-agent-core/02-tool-dispatch.md` | ... |
| `codex-rs/tui/**` | `08-tui-cli-and-integration/01-tui-architecture.md` | ... | ... |
```

## 3) Coverage Map

```markdown
---
title: "vX.Y Coverage Map for Upstream Sync"
date: "YYYY-MM-DD"
status: "tracking"
scope: "What we have actually reviewed vs. not yet reviewed"
---

# vX.Y Coverage Map

目的：回答三件事：
1. 哪些源码路径已经看过
2. 哪些只是登记了但没复核
3. TS/前端面是否纳入计划

口径：
- "已看过"：已读源码且用于 owner 文档或 sync log 写回。
- "未看过" ≠ 不重要。

## 按存储分区统计

| Bucket | Total changed | Reviewed | Unreviewed |
|---|---:|---:|---:|
| `agent-core` | N | N | N |
| `tools-execution` | N | N | N |
| `tui-rs` | N | N | N |
| `cli-frontend` | N | 0 | N |

## 已看过的路径

- `codex-rs/core/src/agent.rs`
- ...

## 已登记但未复核

- `codex-rs/tui/**`
- ...

## 后续建议顺序

1. ...
2. ...
```

## 4) Progressive Sync Plan

```markdown
---
title: "vX.Y Progressive Sync Plan"
date: "YYYY-MM-DD"
status: "planning"
scope: "Staged sync plan for vX.Y upstream delta"
---

# vX.Y Progressive Sync Plan

## 阶段划分

### 阶段 0：Git 对齐（必须最先做）
- [ ] 确认上游基线
- [ ] main 对齐方式决策
- [ ] ethan 合入
- [ ] 记录锚点到 sync log

### 阶段 1：Triage + Owner Map（不改正文）
- [ ] Release note triage 表
- [ ] Path → owner map
- [ ] 产出 next slice 建议

### 阶段 2：Coverage Map（不改正文）
- [ ] 统计 changed files 按存储分区
- [ ] 标记 reviewed / unreviewed
- [ ] 识别前端盲区

### 阶段 3：高优先级运行时切片（开始改正文）
批次 A：agent core / tool dispatch
批次 B：state / memory / sessions
批次 C：model provider / routing

### 阶段 4：中优先级面（继续改正文）
批次 D：TUI / CLI runtime
批次 E：skills / plugins
批次 F：MCP / integration

### 阶段 5：低优先级面 + 收尾
批次 G：security / config / docs
批次 H：前端 TS/TSX 面

## 每阶段固定方法

1. 从 release note / changelog 找方向
2. `git diff <before>..<after> -- <paths>` 锁定变化
3. 读源码 + tests
4. 只更新该主题的 owner 文档
5. 追加到 sync log：源码路径、tests、已更新文档、未覆盖风险

## 不做事项

- 不把 release note 当成源码真相写进正文
- 不把 coverage map / owner map 混进 owner 正文
- 不把 "哪些文件看过" 的台账混进 owner 正文
- 不在一次切片里跨 3+ 个主题目录
```
