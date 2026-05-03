---
title: "07 — State, Memory & Sessions 索引"
doc_type: "index"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解 Codex 状态管理、记忆系统和会话持久化的人"
purpose: "提供状态、记忆和会话层的阅读路径和文档边界"
owns: "状态管理、记忆系统、thread-store、会话持久化相关文档"
update_when:
  - "状态存储或会话机制发生重大变化时"
  - "记忆系统架构变化时"
out_of_scope:
  - "单条记忆的内部格式"
  - "历史归档记录"
---

# 07 — State, Memory & Sessions 索引

> 本层覆盖 Codex 的数据持久化面：对话状态怎么存、记忆系统怎么工作、thread-store 怎么组织会话。

## 计划文档（尚未编写，待源码复核后逐步填充）

1. `01-state-and-thread-store.md` — 状态管理与 thread-store：会话持久化、检索、恢复
2. `02-memory-system.md` — 记忆系统：MEMORY.md、USER.md、外部 memory provider

## 关键 crate

- `codex-rs/state/` — 状态持久化
- `codex-rs/memories/` — 记忆系统
- `codex-rs/thread-store/` — 线程/会话存储
- `codex-rs/agent-graph-store/` — Agent 图存储

## 关键概念（待源码复核确认）

- Thread 生命周期：创建、活跃、压缩、结束
- 记忆冻结快照 vs 运行时刷新
- 会话检索（session search）
- 外部 memory provider 接口
