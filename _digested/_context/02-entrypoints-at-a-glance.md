---
title: "02 — Entrypoints At A Glance（入口一页速查）"
doc_type: "context"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "刚开始读仓库的人，或需要快速定位命令入口和关键文件的人"
purpose: "给出 Codex 的主要入口、下一跳和最值得抓的源码位置"
owns: "命令入口、运行时入口、关键入口文件和下一跳源码抓手"
update_when:
  - "CLI / TUI / app-server / MCP 的启动链或入口文件发生变化时"
  - "新增或移除主要入口时"
out_of_scope:
  - "单个入口的细粒度实现"
  - "历史归档记录"
---

# 02 — Entrypoints At A Glance（入口一页速查）

目标：给"刚开始读仓库的人 / 新 Agent"一页纸定位关键入口，并标出链路下一跳。

> 注意：本文列出的具体源码文件路径和启动链是**占位设计**，尚未经过源码级验证。后续完成对应切片的 upstream sync 复核后，再更新为核实过的事实。

---

## 1) CLI 入口

Codex CLI 是一个 Node.js 前端（`codex-cli/`），通过 Rust backend（`codex-rs/cli/`、`codex-rs/app-server/`）提供 AI 能力。

入口：
- `codex` 命令 → `codex-cli/bin/` → 连接到 Rust backend

你要找的关键点：
- CLI frontend：`codex-cli/`
- CLI backend（Rust）：`codex-rs/cli/`
- App server：`codex-rs/app-server/`、`codex-rs/app-server-client/`
- 协议定义：`codex-rs/app-server-protocol/`

---

## 2) TUI 入口

TUI 是 Rust 原生实现的主要交互面。

入口：
- TUI crate：`codex-rs/tui/`

你要找的关键点：
- TUI 组件与渲染
- 输入处理与事件循环
- 与 core agent 的桥接

---

## 3) MCP 入口（Model Context Protocol）

MCP 支持通过标准协议接入外部工具和编辑器。

入口：
- `codex-rs/codex-mcp/` — MCP 协议实现
- `codex-rs/mcp-server/` — MCP server 端
- `codex-rs/rmcp-client/` — Rust MCP client

---

## 4) App Server / HTTP API 入口

App server 提供 HTTP API 供外部系统集成。

入口：
- `codex-rs/app-server/` — HTTP API 服务
- `codex-rs/app-server-protocol/` — 协议类型定义
- `codex-rs/app-server-transport/` — 传输层

---

## 5) Agent 核心入口（所有路径都会落到这里）

无论从哪个入口进入，最终都会到达 agent 核心：

```
CLI / TUI / MCP / App Server
    → codex-rs/core/         ← Agent 循环
    → codex-rs/tools/        ← 工具执行
    → codex-rs/model-provider/ ← 模型调用
    → codex-rs/skills/       ← 技能执行
```

---

## 6) 配置加载链

所有入口都会经过的配置加载链（具体机制待源码复核确认）：

```
环境变量 > 配置文件 > 默认值
```

关键配置路径：
- `codex-rs/config/` — 配置管理 crate

---

## 7) 构建系统入口

Codex 使用 Bazel + Cargo 双构建系统：
- `MODULE.bazel` — Bazel 模块定义
- `BUILD.bazel` — 根构建文件
- `codex-rs/Cargo.toml` — Cargo workspace
- `defs.bzl` — Bazel 宏定义
- `justfile` — Just 命令入口（开发常用）
