---
title: "08 — TUI, CLI & Integration 索引"
doc_type: "index"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解 Codex 用户交互面、CLI、MCP 和外部集成的人"
purpose: "提供 TUI、CLI、app-server、MCP 和外部集成面的阅读路径和文档边界"
owns: "TUI 架构、CLI 入口、app-server、MCP 协议、外部集成相关文档"
update_when:
  - "TUI 架构或 CLI 入口发生变化时"
  - "新增或移除集成协议时"
out_of_scope:
  - "TUI 组件级 UI 细节"
  - "历史归档记录"
---

# 08 — TUI, CLI & Integration 索引

> 本层覆盖 Codex 的所有"对外接口面"：用户怎么交互（TUI、CLI），外部系统怎么集成（MCP、HTTP API、WebRTC）。

## 计划文档（尚未编写，待源码复核后逐步填充）

1. `01-tui-architecture.md` — TUI 架构：渲染、事件循环、组件树、与 agent core 的桥接
2. `02-cli-and-app-server.md` — CLI 入口与 app-server（HTTP API）
3. `03-mcp-and-external-integration.md` — MCP 协议、外部工具集成、实时通信

## 关键 crate

- `codex-rs/tui/` — TUI 交互面（Rust 原生）
- `codex-rs/cli/` — CLI backend（Rust）
- `codex-cli/` — CLI frontend（Node.js）
- `codex-rs/app-server/` — HTTP API 服务
- `codex-rs/app-server-client/` — API client
- `codex-rs/app-server-protocol/` — 协议类型
- `codex-rs/app-server-transport/` — 传输层
- `codex-rs/codex-mcp/` — MCP 协议实现
- `codex-rs/mcp-server/` — MCP server
- `codex-rs/rmcp-client/` — Rust MCP client
- `codex-rs/realtime-webrtc/` — WebRTC 实时通信
- `codex-rs/responses-api-proxy/` — Responses API 代理
- `codex-rs/connectors/` — 外部连接器

## 关键概念（待源码复核确认）

- TUI 事件循环与渲染管线
- CLI 命令分发（slash commands）
- App-server HTTP API 路由与生命周期
- MCP 协议交互：initialize、tools/list、tools/call
- 实时通信（WebRTC）的使用场景
