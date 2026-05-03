---
title: "03 — Agent Core 索引"
doc_type: "index"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解 Agent 核心循环、对话生命周期和工具分发的人"
purpose: "提供 Agent Core 层的阅读路径和文档边界"
owns: "Agent 核心循环、对话生命周期、工具分发、agent identity 相关文档"
update_when:
  - "Agent 核心循环或生命周期发生重大变化时"
  - "工具分发机制变化时"
out_of_scope:
  - "单个工具的业务逻辑"
  - "TUI/CLI 的交互实现"
  - "历史归档记录"
---

# 03 — Agent Core 索引

> 本层覆盖 Codex 的心脏：Agent 如何从用户输入走到模型调用、工具执行、响应生成。核心 crate 是 `codex-rs/core/`。

## 文档

1. `01-agent-loop-and-lifecycle.md` — Agent 生命周期：7 阶段创建→运行→销毁，三种 teardown 语义（shutdown / release / close），channel 架构
2. `02-tool-dispatch-and-execution.md` — 工具分发：三条 schema 来源、ToolRouter 组装、7 阶段 dispatch 链路、25 个可用性 gate、AGENT_LOOP_TOOLS 概念澄清
3. `03-agent-identity-and-system-prompt.md` — Agent 身份（JWT + Ed25519）+ system prompt 11 层装配 + reference context diff 机制 + prompt caching 失效触发条件
4. `04-context-and-compression.md` — ContextManager 结构、Memento compaction 策略、pre-turn/mid-turn 触发、失败回退、与 prompt caching 的交互

## 关键 crate

- `codex-rs/core/` — 核心循环、对话生命周期
- `codex-rs/core-api/` — 核心 API 类型定义
- `codex-rs/core-plugins/` — 核心插件 trait
- `codex-rs/core-skills/` — 核心技能
- `codex-rs/agent-identity/` — Agent 身份定义
- `codex-rs/agent-graph-store/` — Agent 图存储

## 关键源码抓手（待复核确认）

- Agent 循环入口
- 对话消息组装
- 模型调用流程
- 工具调用截获与分发
- 响应处理与持久化
