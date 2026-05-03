---
title: "Phase 1 Upstream Sync — 02-system-architecture"
date: "2026-05-03"
status: "completed"
scope: "Phase 1 of progressive production plan: system landscape + information flow"
---

# Phase 1 Sync Log — 02-system-architecture

## 对齐锚点

- **baseline_ref**: `origin/main`
- **baseline_commit**: `35aaa5d9f` — Bound websocket request sends with idle timeout (#20751)
- **ethan_commit**: `6ffd48dc6` — Phase 0: git baseline lock + 106-crate dependency scan

## 已复核源码路径

- `codex-rs/Cargo.toml` — workspace 定义（106 member + 2 path-only）
- `codex-rs/core/src/session/turn.rs` — `run_turn()` 主循环
- `codex-rs/core/src/session/handlers.rs` — `submission_loop()` 事件分发
- `codex-rs/core/src/session/mod.rs` — `Codex` struct + `Codex::spawn()`
- `codex-rs/core/src/state/service.rs` — `SessionServices` DI 容器
- `codex-rs/core/src/session/turn_context.rs` — `TurnContext` 不可变快照
- `codex-rs/core/src/tools/router.rs` — `ToolRouter` 工具分发
- `codex-rs/core/src/tools/orchestrator.rs` — `ToolOrchestrator` 审批-沙箱循环
- `codex-rs/core/src/tasks/regular.rs` — `RegularTask` turn 循环包装
- `codex-rs/config/src/loader/mod.rs` — `load_config_layers_state()` 配置加载链
- `codex-rs/config/src/state.rs` — `ConfigLayerStack`、`ConfigLayerEntry`
- `codex-rs/config/src/config_toml.rs` — `ConfigToml` schema
- `codex-rs/cli/src/main.rs` — CLI 入口
- `codex-rs/cli/src/lib.rs` — `MultitoolCli`、`Subcommand` 枚举
- `codex-rs/protocol/src/protocol.rs` — `Op`、`EventMsg` 协议枚举
- `codex-rs/protocol/src/items.rs` — `TurnItem` 枚举
- `codex-rs/app-server-protocol/src/protocol/` — `ClientRequest`、`ServerNotification`
- `codex-rs/core-api/src/lib.rs` — 公共 API facade
- `codex-rs/model-provider/src/lib.rs` — 存在性确认
- `codex-cli/bin/codex.js` — Node.js 薄壳入口

## 已复核 tests（确认存在，未逐条审读）

- `codex-rs/core/tests/` — 存在，core 测试目录
- `codex-rs/config/tests/` — 存在，config 测试目录

## 确认并写回的事实

- Cargo workspace 有 106 个显式 member + 2 个 path-only crate（chatgpt、windows-sandbox-rs），合计 108
- crate 按功能分为 9 域，`core` 是最大中枢（依赖 40+ 内部 crate）
- 进程模型有三种拓扑：交互式 TUI（内置 app-server）、非交互式、守护进程、MCP 模式
- 信息流 7 阶段：前端捕获 → JSON-RPC 翻译 → Codex::submit() 入队 → submission_loop 分发 → run_turn 核心循环 → 响应处理 → 事件回流
- 5 个持久化写入点：thread-store (SQLite)、rollout (JSONL)、memory (文件)、config (TOML)、analytics (HTTP)
- 8 条关键模块边界，均以 trait/channel/JSON-RPC 定义契约
- Node.js 薄壳只是一个平台检测 + spawn，真正的逻辑全部在 Rust 侧

## 已更新文档

- `_digested/02-system-architecture/01-system-landscape.md` — 新建
- `_digested/02-system-architecture/02-information-flow-and-boundaries.md` — 新建
- `_digested/02-system-architecture/README.md` — 更新文档状态

## 未覆盖风险 / 下一步

- [ ] TUI 渲染管线和事件循环细节 — 留给 Phase 7
- [ ] model-provider trait 的具体定义 — 留给 Phase 4
- [ ] Bazel BUILD 与 Cargo.toml 的一致性 — 低优先级，暂不检查
- [ ] Phase 2：03-agent-core（4 篇文档）— 下一个切片
