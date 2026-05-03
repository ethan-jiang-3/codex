---
title: "Phase 0 — Git 基线锁定 + Crate 依赖图扫描"
date: "2026-05-03"
status: "completed"
scope: "Git 基线确认 + 106 workspace crate 分区 + 关键入口定位"
---

# Phase 0 — Git 基线锁定 + Crate 依赖图扫描

目的：在开始写任何 `_digested` 正文前，先建立工作基础——确认 git 基线锚点、理清 crate 依赖关系、定位关键入口文件。

## 1) Git 基线锚点（2026-05-03）

| 项目 | 值 |
|------|-----|
| 当前分支 | `ethan` |
| `ethan` HEAD | `229fef4eb` — Add 10-phase progressive production plan for _digested owner docs |
| `main` HEAD | `35aaa5d9f` — Bound websocket request sends with idle timeout (#20751) |
| `origin` URL | `git@github.com:ethan-jiang-3/codex.git` |
| `upstream` | 未配置 |
| `main...origin/main` | `0 / 0`（已对齐） |
| `ethan` ahead of `main` | 2 commits（均为 `_digested/` + `_tmp_tracking/` 文档提交） |
| 结论 | `main` 已对齐 `origin/main`，`ethan` 仅领先文档层，无源码分叉 |

## 2) Workspace 成员统计

`codex-rs/Cargo.toml` 显式声明 **106 个** workspace member。

另有 2 个 crate 以 path dependency 形式存在（cargo 自动发现）：
- `chatgpt/` — ChatGPT API 适配器
- `windows-sandbox-rs/` — Windows 沙箱实现

合计：**108 个 Rust crate**（均位于 `codex-rs/` 下）。

> 注：还有 `codex-rs/codex-server/` 可能未被 workspace 引用；`codex-rs/memories/read` 和 `codex-rs/memories/write` 是以两级路径作为 member 的 crate。

## 3) Crate 按功能域分组（9 域）

### A. Agent 核心（agent-core）— 6 crates

| Crate | 路径 | 角色 |
|-------|------|------|
| `core` | `codex-rs/core/` | 主循环、对话生命周期、工具分发中枢 |
| `core-api` | `codex-rs/core-api/` | 核心 API 类型定义 |
| `core-plugins` | `codex-rs/core-plugins/` | 内置插件实现 |
| `core-skills` | `codex-rs/core-skills/` | 内置技能实现 |
| `agent-identity` | `codex-rs/agent-identity/` | Agent 身份定义（SOUL.md 等） |
| `agent-graph-store` | `codex-rs/agent-graph-store/` | Agent 图存储 |

**关键入口**：`codex-rs/core/src/lib.rs`

### B. 工具与执行（tools-execution）— 14 crates

| Crate | 路径 | 角色 |
|-------|------|------|
| `tools` | `codex-rs/tools/` | 工具实现与注册 |
| `exec` | `codex-rs/exec/` | 命令执行核心 |
| `exec-server` | `codex-rs/exec-server/` | 命令执行服务端 |
| `execpolicy` | `codex-rs/execpolicy/` | 执行策略 |
| `execpolicy-legacy` | `codex-rs/execpolicy-legacy/` | 旧版执行策略 |
| `shell-command` | `codex-rs/shell-command/` | Shell 命令抽象 |
| `shell-escalation` | `codex-rs/shell-escalation/` | Shell 权限提升 |
| `process-hardening` | `codex-rs/process-hardening/` | 进程加固 |
| `sandboxing` | `codex-rs/sandboxing/` | 沙箱抽象层 |
| `linux-sandbox` | `codex-rs/linux-sandbox/` | Linux Seatbelt 沙箱 |
| `windows-sandbox-rs` | `codex-rs/windows-sandbox-rs/` | Windows 沙箱（不在 workspace members 中，path-only） |
| `apply-patch` | `codex-rs/apply-patch/` | 文件 patch 应用 |
| `file-system` | `codex-rs/file-system/` | 文件系统操作 |
| `file-search` | `codex-rs/file-search/` | 文件搜索 |

**关键入口**：`codex-rs/tools/src/lib.rs`、`codex-rs/exec/src/lib.rs`、`codex-rs/sandboxing/src/lib.rs`

### C. 模型与提供商（models-providers）— 10 crates

| Crate | 路径 | 角色 |
|-------|------|------|
| `model-provider` | `codex-rs/model-provider/` | Provider 抽象 trait |
| `model-provider-info` | `codex-rs/model-provider-info/` | 模型元信息 |
| `models-manager` | `codex-rs/models-manager/` | 模型管理/路由/选择 |
| `backend-client` | `codex-rs/backend-client/` | Codex 后端客户端 |
| `codex-api` | `codex-rs/codex-api/` | Codex API 定义 |
| `codex-client` | `codex-rs/codex-client/` | Codex 客户端 |
| `ollama` | `codex-rs/ollama/` | Ollama 适配器 |
| `lmstudio` | `codex-rs/lmstudio/` | LM Studio 适配器 |
| `chatgpt` | `codex-rs/chatgpt/` | ChatGPT 适配器（不在 workspace members 中，path-only） |
| `codex-backend-openapi-models` | `codex-rs/codex-backend-openapi-models/` | 后端 OpenAPI 模型 |

**关键入口**：`codex-rs/model-provider/src/lib.rs`、`codex-rs/models-manager/src/lib.rs`

### D. 技能与插件（skills-plugins）— 4 crates

| Crate | 路径 | 角色 |
|-------|------|------|
| `skills` | `codex-rs/skills/` | 技能系统 |
| `plugin` | `codex-rs/plugin/` | 插件 trait 与注册 |
| `utils/plugins` | `codex-rs/utils/plugins/` | 插件工具函数 |

> `core-skills` 和 `core-plugins` 也属于本域，但已归入 agent-core（它们在 core 层被引用）。

**关键入口**：`codex-rs/skills/src/lib.rs`、`codex-rs/plugin/src/lib.rs`

### E. 状态与记忆（state-memory）— 6 crates

| Crate | 路径 | 角色 |
|-------|------|------|
| `state` | `codex-rs/state/` | 状态管理 |
| `thread-store` | `codex-rs/thread-store/` | 会话/线程存储 |
| `thread-manager-sample` | `codex-rs/thread-manager-sample/` | 线程管理器示例 |
| `memories/read` | `codex-rs/memories/read/` | 记忆读取 |
| `memories/write` | `codex-rs/memories/write/` | 记忆写入 |
| `external-agent-sessions` | `codex-rs/external-agent-sessions/` | 外部 agent 会话 |

**关键入口**：`codex-rs/state/src/lib.rs`、`codex-rs/thread-store/src/lib.rs`

### F. TUI / CLI / 集成面（tui-cli-integration）— 21 crates

| Crate | 路径 | 角色 |
|-------|------|------|
| `tui` | `codex-rs/tui/` | TUI 前端（ratatui） |
| `cli` | `codex-rs/cli/` | Rust CLI 入口 |
| `app-server` | `codex-rs/app-server/` | HTTP API 服务端 |
| `app-server-transport` | `codex-rs/app-server-transport/` | 传输层 |
| `app-server-client` | `codex-rs/app-server-client/` | 客户端 SDK |
| `app-server-protocol` | `codex-rs/app-server-protocol/` | 协议类型 |
| `app-server-test-client` | `codex-rs/app-server-test-client/` | 测试客户端 |
| `debug-client` | `codex-rs/debug-client/` | 调试客户端 |
| `protocol` | `codex-rs/protocol/` | 通用协议 |
| `codex-mcp` | `codex-rs/codex-mcp/` | Codex MCP 集成 |
| `mcp-server` | `codex-rs/mcp-server/` | MCP server 实现 |
| `rmcp-client` | `codex-rs/rmcp-client/` | Rust MCP 客户端 |
| `realtime-webrtc` | `codex-rs/realtime-webrtc/` | WebRTC 实时通信 |
| `responses-api-proxy` | `codex-rs/responses-api-proxy/` | Responses API 代理 |
| `connectors` | `codex-rs/connectors/` | 外部连接器 |
| `stdio-to-uds` | `codex-rs/stdio-to-uds/` | stdio→UDS 桥接 |
| `uds` | `codex-rs/uds/` | Unix Domain Socket |
| `terminal-detection` | `codex-rs/terminal-detection/` | 终端检测 |
| `login` | `codex-rs/login/` | 登录认证 |
| `code-mode` | `codex-rs/code-mode/` | 代码模式 |

**关键入口**：`codex-rs/tui/src/main.rs`、`codex-rs/app-server/src/main.rs`、`codex-rs/cli/src/lib.rs`

### G. 配置与安全（config-security）— 10 crates

| Crate | 路径 | 角色 |
|-------|------|------|
| `config` | `codex-rs/config/` | 配置加载链 |
| `features` | `codex-rs/features/` | 功能开关 |
| `secrets` | `codex-rs/secrets/` | 密钥管理 |
| `keyring-store` | `codex-rs/keyring-store/` | Keyring 存储 |
| `device-key` | `codex-rs/device-key/` | 设备密钥 |
| `aws-auth` | `codex-rs/aws-auth/` | AWS 认证 |
| `network-proxy` | `codex-rs/network-proxy/` | 网络代理 |
| `rollout` | `codex-rs/rollout/` | 灰度发布 |
| `rollout-trace` | `codex-rs/rollout-trace/` | 灰度追踪 |
| `install-context` | `codex-rs/install-context/` | 安装上下文 |

**关键入口**：`codex-rs/config/src/lib.rs`

### H. 基础设施/工具（infra-utils）— 18 crates

| Crate | 路径 | 角色 |
|-------|------|------|
| `analytics` | `codex-rs/analytics/` | 分析/遥测 |
| `otel` | `codex-rs/otel/` | OpenTelemetry 集成 |
| `feedback` | `codex-rs/feedback/` | 用户反馈 |
| `hooks` | `codex-rs/hooks/` | Hook 系统 |
| `response-debug-context` | `codex-rs/response-debug-context/` | 响应调试上下文 |
| `collaboration-mode-templates` | `codex-rs/collaboration-mode-templates/` | 协作模式模板 |
| `external-agent-migration` | `codex-rs/external-agent-migration/` | 外部 agent 迁移 |
| `git-utils` | `codex-rs/git-utils/` | Git 工具 |
| `cloud-requirements` | `codex-rs/cloud-requirements/` | 云端需求 |
| `cloud-tasks` | `codex-rs/cloud-tasks/` | 云端任务 |
| `cloud-tasks-client` | `codex-rs/cloud-tasks-client/` | 云端任务客户端 |
| `cloud-tasks-mock-client` | `codex-rs/cloud-tasks-mock-client/` | 云端任务 mock |
| `ansi-escape` | `codex-rs/ansi-escape/` | ANSI 转义处理 |
| `arg0` | `codex-rs/arg0/` | 参数处理 |
| `test-binary-support` | `codex-rs/test-binary-support/` | 测试二进制支持 |
| `codex-experimental-api-macros` | `codex-rs/codex-experimental-api-macros/` | 实验性 API 宏 |
| `v8-poc` | `codex-rs/v8-poc/` | V8 引擎 POC |
| `async-utils` | `codex-rs/async-utils/` | 异步工具 |

### I. 通用工具库（utils）— 20 crates

全部位于 `codex-rs/utils/` 下：

| Crate | 角色 |
|-------|------|
| `absolute-path` | 绝对路径抽象 |
| `approval-presets` | 批准预设 |
| `cache` | 缓存 |
| `cargo-bin` | Cargo 二进制工具 |
| `cli` | CLI 工具函数 |
| `elapsed` | 耗时追踪 |
| `fuzzy-match` | 模糊匹配 |
| `home-dir` | 主目录 |
| `image` | 图像处理 |
| `json-to-toml` | JSON→TOML 转换 |
| `oss` | OSS 相关 |
| `output-truncation` | 输出截断 |
| `path-utils` | 路径工具 |
| `pty` | PTY 抽象 |
| `readiness` | 就绪检查 |
| `rustls-provider` | Rustls 提供商 |
| `sandbox-summary` | 沙箱摘要 |
| `sleep-inhibitor` | 睡眠抑制 |
| `stream-parser` | 流解析器 |
| `string` | 字符串工具 |
| `template` | 模板引擎 |

## 4) 核心依赖关系（关键链）

```
app-server ──→ core ──→ tools
   │            │         │
   │            ├──→ model-provider
   │            ├──→ sandboxing
   │            ├──→ exec-server
   │            ├──→ state
   │            ├──→ thread-store
   │            ├──→ plugin
   │            └──→ config
   │
   ├──→ tui ──→ app-server-client (JSON-RPC)
   │
   └──→ mcp-server
```

- **`core`** 是最大的中枢 crate（依赖 40+ 内部 crate），汇集所有子系统
- **`tui`** 通过 `app-server-client`（JSON-RPC）与后端通信，不直接依赖 `core`
- **`app-server`** 直接依赖 `core`，是 HTTP API 面的启动入口
- **`model-provider`** 定义 Provider trait；各适配器（ollama/lmstudio/chatgpt）实现该 trait
- **`sandboxing`** 定义抽象层；`linux-sandbox` 和 `windows-sandbox-rs` 是具体后端

## 5) 关键入口文件速查

| 入口 | 路径 |
|------|------|
| Agent 核心循环 | `codex-rs/core/src/lib.rs` |
| TUI 主入口 | `codex-rs/tui/src/main.rs` |
| App Server 主入口 | `codex-rs/app-server/src/main.rs` |
| CLI 主入口 | `codex-rs/cli/src/lib.rs` |
| 工具注册 | `codex-rs/tools/src/lib.rs` |
| 沙箱抽象 | `codex-rs/sandboxing/src/lib.rs` |
| 模型 Provider trait | `codex-rs/model-provider/src/lib.rs` |
| 技能系统 | `codex-rs/skills/src/lib.rs` |
| 插件系统 | `codex-rs/plugin/src/lib.rs` |
| 状态管理 | `codex-rs/state/src/lib.rs` |
| Thread 存储 | `codex-rs/thread-store/src/lib.rs` |
| 配置加载 | `codex-rs/config/src/lib.rs` |
| MCP Server | `codex-rs/mcp-server/src/lib.rs` |
| Workspace 定义 | `codex-rs/Cargo.toml` |

## 6) 后续 Phase 的阅读顺序建议

基于依赖深度，Phase 顺序与生产计划一致：

1. **Phase 1** → `config`、`core`（架构全景）
2. **Phase 2** → `core`、`core-api`、`agent-identity`、`agent-graph-store`（agent 核心）
3. **Phase 3** → `tools`、`exec`、`sandboxing`、`shell-command`（工具与执行）
4. **Phase 4** → `model-provider`、`models-manager`、各适配器（模型）
5. **Phase 5** → `skills`、`plugin`（技能与插件）
6. **Phase 6** → `state`、`thread-store`、`memories/`（状态与记忆）
7. **Phase 7** → `tui`、`app-server`、`mcp-server`、`realtime-webrtc`（交互面）
8. **Phase 8** → 入门文档（依赖以上所有知识）

## 7) Phase 0 未覆盖项

- [ ] `codex-cli/`（Node.js TypeScript 前端）的源码定位与模块划分 — 留到 Phase 7
- [ ] 各 crate 的测试文件路径枚举 — 每个 Phase 内按需扫描
- [ ] Bazel BUILD 文件与 Cargo.toml 的依赖声明一致性校验 — 低优先级
- [ ] `chatgpt` / `windows-sandbox-rs` 是否为有意排除 workspace member 还是 cargo 自动发现 — 无需深究
