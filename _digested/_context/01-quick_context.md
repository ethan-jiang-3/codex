---
title: "Codex — Quick Context"
doc_type: "context"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要在 5 分钟内建立 Codex 基本心智模型的 Agent/工程师"
purpose: "提供一页纸系统总览，并把读者快速导向最有价值的主题文档"
owns: "Codex 的最小心智模型、关键入口、主题分区和任务导向阅读入口"
update_when:
  - "关键入口、主子系统或主题分区发生变化时"
  - "推荐阅读路径需要调整时"
out_of_scope:
  - "单个子系统的细节实现"
  - "历史归档记录"
---

# Codex — Quick Context

> Agent 上下文加速页。目标：先用 5 分钟建立心智模型，再按任务只读 1-3 篇最值钱的 `_digested/` 文档。

## 怎么用这页

- 默认不要通读整个 `_digested/`
- 先读这一页，拿到系统全景、关键入口、主题分区
- 然后按你当前任务，从下面的"按任务跳转"里只选 1-3 篇继续读
- `_meta/` 默认不用看；只有在需要维护记录时再去

## 一句话

Codex 是 Zed 编辑器的 AI Agent 系统，以 `codex-core` crate 为核心，Rust 实现，Bazel + Cargo 双构建系统。它通过 CLI / TUI / app-server / MCP 等多入口面提供 AI 编程能力，核心包括 agent loop（agent 循环）、tool dispatch（工具分发）、skill system（技能系统）、model provider 路由、sandbox（沙箱）执行环境和会话持久化。

## 先建立最小心智模型

1. 用户通过某个 entrypoint（入口面）进入：CLI（`codex-cli/`）、TUI（`codex-rs/tui/`）、app-server（HTTP API）、MCP（`codex-mcp/`）
2. 核心 agent loop 在 `codex-core` crate 中，负责对话循环、工具调用、模型交互
3. 工具（tools）经由 `codex-rs/tools/` 注册和分发，包括 exec（命令执行）、shell、file 等
4. 技能系统（skills）通过 `codex-rs/skills/` 和 `codex-rs/core-skills/` 提供可扩展的 agent 能力
5. 模型提供商（model providers）通过 `codex-rs/model-provider/` 抽象，支持 OpenAI、Anthropic、Ollama、LM Studio 等
6. 沙箱（sandboxing）通过 `codex-rs/sandboxing/`、`linux-sandbox/`、`windows-sandbox-rs/` 提供安全执行隔离
7. 状态和记忆（state & memory）由 `codex-rs/state/`、`codex-rs/memories/`、`codex-rs/thread-store/` 共同支撑

## 五个最该记住的入口

| 入口 | 为什么重要 |
|------|-----------|
| `codex-rs/core/` | 系统心脏：Agent 核心循环、对话生命周期、工具分发 |
| `codex-rs/tools/` | 工具注册与实现；几乎所有的 agent 能力都从这里接出 |
| `codex-rs/tui/` | TUI 交互面；用户主要对话界面 |
| `codex-rs/model-provider/` | 模型抽象层；所有 LLM 调用都经过这里 |
| `codex-rs/skills/` | 技能系统；可编程的 agent 行为扩展 |

## 主题分区怎么理解

| 目录 | 看什么 |
|------|--------|
| `01-beginner/` | 跑起来、配置、排错、常见能力选择 |
| `02-system-architecture/` | 系统全景、crate 地图、信息流、模块边界 |
| `03-agent-core/` | Agent 核心循环、对话生命周期、工具分发（tool dispatch）、agent identity |
| `04-tools-execution-and-sandboxing/` | 工具系统、exec/shell、沙箱、进程加固 |
| `05-skills-and-plugins/` | 技能系统、插件机制、skill 创作与发现 |
| `06-models-and-providers/` | 模型提供商、模型管理、API 路由与适配 |
| `07-state-memory-and-sessions/` | 状态管理、记忆系统、thread-store、会话 |
| `08-tui-cli-and-integration/` | TUI、CLI、app-server、MCP、外部集成面 |

## 按任务跳转

### 我想先理解整个系统

- `02-system-architecture/01-system-landscape.md`
- `02-system-architecture/02-information-flow-and-boundaries.md`
- `_context/02-entrypoints-at-a-glance.md`

### 我要改 agent 核心逻辑

- `03-agent-core/01-agent-loop-and-lifecycle.md`
- `03-agent-core/02-tool-dispatch-and-execution.md`

### 我要看 tools 是怎么接进去和执行的

- `04-tools-execution-and-sandboxing/01-tool-registry-and-dispatch.md`
- `04-tools-execution-and-sandboxing/02-exec-and-shell.md`

### 我要看 skills / plugins 怎么扩展

- `05-skills-and-plugins/01-skill-system.md`
- `05-skills-and-plugins/02-plugin-mechanism.md`

### 我要查 model provider / API 路由

- `06-models-and-providers/01-model-provider-abstraction.md`
- `06-models-and-providers/02-model-routing-and-selection.md`

### 我要查 state / memory / session

- `07-state-memory-and-sessions/01-state-and-thread-store.md`
- `07-state-memory-and-sessions/02-memory-system.md`

### 我要看 TUI / CLI / 集成面

- `08-tui-cli-and-integration/01-tui-architecture.md`
- `08-tui-cli-and-integration/02-cli-and-app-server.md`
- `08-tui-cli-and-integration/03-mcp-and-external-integration.md`

### 我要先把项目跑起来或排错

- `01-beginner/01-quickstart.md`
- `01-beginner/02-config-and-profiles.md`
- `01-beginner/03-logs-and-troubleshooting.md`

## 关键依赖链

```text
codex-rs/model-provider/     ← 所有 LLM 调用的抽象层
    → codex-rs/core/         ← Agent 核心循环
    → codex-rs/tools/        ← 工具注册与分发
    → codex-rs/tui/          ← 用户交互面
    → codex-rs/skills/       ← 技能执行
```

## 常用路径速查

| 路径 | 含义 |
|------|------|
| `codex-rs/core/` | Agent 核心：循环、生命周期、工具分发 |
| `codex-rs/tui/` | TUI 交互面 |
| `codex-rs/cli/` | CLI 入口 |
| `codex-rs/tools/` | 工具实现与注册 |
| `codex-rs/skills/` | 技能系统 |
| `codex-rs/model-provider/` | 模型提供商抽象 |
| `codex-rs/state/` | 状态持久化 |
| `codex-rs/memories/` | 记忆系统 |
| `codex-rs/thread-store/` | 会话/线程存储 |
| `codex-rs/sandboxing/` | 沙箱执行环境 |
| `codex-rs/exec/` | 命令执行 |
| `codex-rs/codex-mcp/` | MCP 协议支持 |
| `codex-rs/app-server/` | HTTP API 服务 |
| `codex-cli/` | CLI 前端（Node.js） |

## 配置位置

- `$CODEX_HOME/config.yaml` — 设置（通过 CLI 管理）
- `$CODEX_HOME/.env` — API 密钥
- `$CODEX_HOME/logs/` — 日志
