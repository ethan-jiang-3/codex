---
title: "_digested — Codex 源码消化文档"
doc_type: "index"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "刚接手仓库的人，以及需要快速建立 Codex 心智模型的 Agent/工程师"
purpose: "作为 `_digested` 的总导航，说明目录分工、阅读顺序和文档维护契约"
owns: "`_digested` 的导航方式、目录边界、维护原则和阅读入口"
update_when:
  - "目录结构发生变化时"
  - "文档维护规则或导航方式发生变化时"
  - "新增或移除 canonical 文档时"
out_of_scope:
  - "具体实现机制的细节解释"
  - "历史归档内容的逐条维护记录"
---

# _digested — Codex 源码消化文档

`_digested/` 是对 Codex（Zed AI Agent）源码的消化文档层。这里不改源码，只整理"源码已经是什么样"的事实、边界、入口和排错抓手。

## 怎么使用

- `README.md` 和 `_context/` 负责导航。
- `01-beginner/` 到 `08-tui-cli-and-integration/` 的主题文档负责自说明，正文应尽量独立阅读，不依赖其他 `_digested` 文档才能看懂。
- `_meta/` 只保留历史归档，不是当前导航的一部分。
- 每篇文档头部都有 YAML front matter，用来说明这篇文档的读者、目标、负责范围和后续补充入口。
- 当某篇文档被纳入一次正式的 upstream sync 复核时，还会在 front matter 里追加结构化 `sync_status`，用于标记它对齐的是哪段源码基线。

## 目录分工

- `_context/`：快速上下文与入口速查。允许保留强导航。
- `01-beginner/`：上手、配置、排错、运行模式、常见用户问题。
- `02-system-architecture/`：系统全景、crate 地图、信息流、模块边界。
- `03-agent-core/`：Agent 核心循环、对话生命周期、工具分发、agent identity。
- `04-tools-execution-and-sandboxing/`：工具系统、exec/shell、沙箱、进程加固。
- `05-skills-and-plugins/`：技能系统、插件机制、skill 创作与发现。
- `06-models-and-providers/`：模型提供商、模型管理、路由、API 适配。
- `07-state-memory-and-sessions/`：状态管理、记忆系统、thread-store、会话。
- `08-tui-cli-and-integration/`：TUI、CLI、app-server、MCP、外部集成面。
- `_meta/`：历史结构、归档记录、git 对齐追踪、upstream sync 日志。

## 阅读顺序

如果只想快速进入状态，按这个顺序：

1. 先看 `_context/01-quick_context.md`
2. 再看 `_context/02-entrypoints-at-a-glance.md`
3. 然后进入你关心的主题目录
4. 只有在需要追历史决策时才看 `_meta/`

## 维护契约

- 主题文档应以源码与 tests 为准，不把别的 `_digested` 文档当成依赖。
- `README` 和 `_context` 可以做文档级跳转，其它主题文档默认不做依赖式交叉引用。
- **术语必须中英文对照**：Codex 里不够大众/容易翻译跑偏的专有术语，首次出现必须写成「中文（English）」并保留可回溯的英文原词，避免只写中文导致与源码/讲稿对不上。
- 当一个话题需要补充时，优先补到 front matter 的 `owns` 所声明的那篇文档。
- 当一条信息只和历史演进有关，而不属于当前推荐读法时，应放入 `_meta/`。
- **写作风格**：所有文档遵循 `WRITING-STYLE.md` 的结构模板、编号规范、表格原型和深度梯度规则。
- **上游同步工作流**：`_skills/codex-progressive-sync/SKILL.md` 定义了如何执行一次完整的 upstream sync 切片。

### `sync_status` front matter 约定

当一篇文档被正式纳入 `_meta/git-tracking/upstream-sync/` 体系复核后，front matter 可追加：

```yaml
sync_status:
  source: "upstream-sync"
  baseline_ref: "origin/main"
  baseline_before: "<short hash>"
  baseline_after: "<short hash>"
  state: "updated"
```

字段口径：

- `source`：当前状态来自 upstream sync 复核，而不是普通文字润色。
- `baseline_ref`：这次对齐所参考的源码基线 ref；本 clone 当前是 `origin/main`。
- `baseline_before` / `baseline_after`：这轮复核所覆盖的源码基线 short hash。
- `state`：当前只允许 `updated`、`reviewed_no_change`、`unknown`。

维护口径：

- **缺少 `sync_status` 不自动等于 `unknown`**。它更常表示"这篇文档还没被纳入这套页级状态回填"。
- `reviewed_no_change` 只能在 tracking 已明确记载"已复核且无需改正文"时使用，不能靠"git 没改"倒推。
- 细节证据（源码路径、tests、未覆盖风险）仍以 `_meta/git-tracking/upstream-sync/*.md` 为准；`sync_status` 只提供单页可见的摘要状态。

### `_digested` 与源码的版本对齐（很重要）

Codex 源码会持续变化，`_digested/` 也会持续更新。为了让文档论断可追溯，需要把关键的 git 结构与对齐锚点记录下来。

核心口径：

- `main` 是源码基线分支，应该周期性对齐上游源码基线（优先 `upstream/main`；未配置 upstream 时，以当前约定的远端基线为准，例如 `origin/main`）。
- `ethan` 是 `_digested/` 的工作分支，预期会长期领先 `main`，因为它承载源码消化与归档提交。
- 真正需要追踪的是：上游源码基线到 `main` 改了哪些源码路径（排除 `_digested/`），这些变化是否需要复核 `_digested` 的哪些主题文档。
- **分支/remote/上游对齐的事实记录**放在 `_meta/`（归档层），作为历史锚点。

权威入口：

- `_meta/git-tracking/git-branch-and-upstream-tracking.md`
- `_meta/git-tracking/upstream-sync/README.md`
