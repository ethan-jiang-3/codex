---
title: "Progressive Discovery Archive"
doc_type: "archive"
status: "archive"
branch: "ethan"
updated: "2026-05-03"
audience: "维护者"
purpose: "记录 `_digested` 建立过程中的探索路径、设计决策和结构变更"
owns: "初始设计决策、目录结构调整记录和探索过程中的关键认知变更"
update_when:
  - "`_digested` 目录结构发生重组或编号调整时"
  - "发现原有设计假设需要修正时"
out_of_scope:
  - "当前主题文档的实现细节"
---

# Progressive Discovery Archive

## 2026-05-03 — 初始结构建立

从 Hermes-Agent 的 `_digested/` 方法论迁移到 Codex 项目。

**设计决策**：

1. **8 主题分区**（对标 Hermes 的 8 主题模型）：
   - `01-beginner/` — 上手与排错
   - `02-system-architecture/` — 系统全景
   - `03-agent-core/` — Agent 核心
   - `04-tools-execution-and-sandboxing/` — 工具与沙箱
   - `05-skills-and-plugins/` — 技能与插件
   - `06-models-and-providers/` — 模型与提供商
   - `07-state-memory-and-sessions/` — 状态与记忆
   - `08-tui-cli-and-integration/` — TUI/CLI/集成面

2. **Git 追踪体系**完整迁移自 Hermes-Agent：
   - `main` = 源码基线，`ethan` = 消化文档工作分支
   - `_meta/git-tracking/git-branch-and-upstream-tracking.md` 记录分支/remote 快照
   - `_meta/git-tracking/upstream-sync/` 存放每次上游同步日志
   - `TEMPLATE.md` 和 `README.md` 保持与 Hermes 一致的口径

3. **`_tmp_tracking/`** 配套建立：
   - `_digested_isuues/` — 消化文档相关问题追踪
   - `_install_issues/` — 安装与环境问题

4. 所有主题文档的正文**尚未开始**编写——当前只有目录结构和 README 设计文档。后续按 progressive sync 方式逐步填充。

**待确认**：
- 上游 remote 是否需要添加（当前只有 `origin`）
- 主题分区是否需要进一步细化（例如 security 是否应该独立成 09-）
- 各个 crate 的详细职责划分需要源码复核后才能精确写入 owner 文档
