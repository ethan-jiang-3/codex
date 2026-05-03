---
title: "02 — System Architecture 索引"
doc_type: "index"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解 Codex 系统全景、crate 组织和模块边界的人"
purpose: "提供系统架构层的阅读路径和文档边界"
owns: "系统架构层文档、crate 地图、信息流和模块边界"
update_when:
  - "crate 结构或主要模块边界发生变化时"
  - "新增或移除主要子系统时"
out_of_scope:
  - "单个 crate 的内部实现细节"
  - "历史归档记录"
---

# 02 — System Architecture 索引

Codex 是 Zed 编辑器的 AI Agent 系统，包含约 100 个 Rust crate（`codex-rs/`），外加一个 Node.js CLI 前端（`codex-cli/`）。本层负责建立系统全景心智模型。

## 计划文档（尚未编写，待源码复核后逐步填充）

1. `01-system-landscape.md` — 系统全景：crate 地图、主子系统、依赖关系
2. `02-information-flow-and-boundaries.md` — 信息流与模块边界：从用户输入到 agent 响应

## 关键 crate 分组（初步，待源码复核）

### Agent 核心
`core`、`core-api`、`core-plugins`、`core-skills`、`agent-identity`、`agent-graph-store`

### 交互面
`tui`、`cli`、`app-server`、`app-server-client`、`app-server-protocol`、`app-server-transport`

### 工具与执行
`tools`、`exec`、`exec-server`、`sandboxing`、`linux-sandbox`、`windows-sandbox-rs`、`process-hardening`、`shell-command`、`shell-escalation`

### 技能与插件
`skills`、`core-skills`、`plugin`、`core-plugins`

### 模型与提供商
`model-provider`、`model-provider-info`、`models-manager`、`lmstudio`、`ollama`、`chatgpt`

### 状态与记忆
`state`、`memories`、`thread-store`

### 集成与协议
`codex-mcp`、`mcp-server`、`rmcp-client`、`responses-api-proxy`、`realtime-webrtc`、`connectors`、`codex-api`、`codex-client`

### 安全
`secrets`、`execpolicy`、`execpolicy-legacy`、`keyring-store`、`device-key`

### 基础设施
`config`、`utils`、`async-utils`、`file-system`、`file-search`、`git-utils`、`network-proxy`、`login`、`feedback`、`analytics`、`otel`、`rollout`、`hooks`

## 构建系统

- Bazel（`MODULE.bazel`、`BUILD.bazel`、`defs.bzl`）
- Cargo（`codex-rs/Cargo.toml` workspace）
- Just（`justfile` — 开发命令入口）
- Nix（`flake.nix`、`default.nix`）
