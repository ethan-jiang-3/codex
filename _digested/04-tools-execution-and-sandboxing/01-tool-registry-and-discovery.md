---
title: "01 — 工具注册与分发（Tool Registry & Dispatch）"
doc_type: "owner"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解工具如何注册、发现、按平台/配置过滤的人"
purpose: "给出工具的注册机制、ToolRegistryPlan 构建流程、toolset/tools 分类、平台差异化、以及 discovery 延迟加载机制"
owns: "工具系统的注册面：ToolRegistryPlan 构建、ToolsConfig 配置门控、平台工具集、tool discovery"
update_when:
  - "新增/移除工具类别或 ToolsConfig 字段变化时"
  - "工具注册计划构建逻辑变化时"
  - "新增 discovery 机制时"
out_of_scope:
  - "dispatch 链路（见 03-agent-core/02-tool-dispatch-and-execution.md）"
  - "具体工具 handler 实现细节"
---

# 01 — 工具注册与发现（Tool Registry & Discovery）

> [!IMPORTANT]
> 这一篇解释 Codex 工具系统的注册面——工具列表怎么生成的、按什么条件开关、跨平台怎么差异化。

> 读完这篇你应该能回答：有哪些工具、每个工具由什么配置控制、工具注册计划的 20 步构建流程、以及延迟发现机制怎么工作。

## 1) 先记住三件事

1. **工具注册是集中的**：`build_tool_registry_plan()` 是唯一的工具注册入口，接收一个 `ToolsConfig`，产出一个 `ToolRegistryPlan`（specs + handlers）。
2. **没有 "toolset" 抽象**：平台差异化通过 `ToolsConfig` 字段 + `cfg!(unix/windows)` 条件 + Feature flags + `experimental_supported_tools` 实现，不是显式的 toolset 概念。
3. **三种工具来源**：内置（config-gated）、MCP（外部 server 发现）、动态（turn 级注入）。后两种支持延迟加载（`tool_search` 触发）。

## 2) 核心类型

| 类型 | 位置 | 角色 |
|------|------|------|
| `ToolRegistryPlan` | `tools/src/tool_registry_plan_types.rs:52` | 注册计划输出：`specs: Vec<ConfiguredToolSpec>` + `handlers: Vec<ToolHandlerSpec>` |
| `ToolHandlerKind` | 同上:12 | 扁平枚举，包含所有 handler 变体（Shell, UnifiedExec, ApplyPatch, Mcp, Goal, 多 agent 等） |
| `ToolHandlerSpec` | 同上:46 | `{ name: ToolName, kind: ToolHandlerKind }` 映射 |
| `ToolSpec` | `tools/src/tool_spec.rs:22` | JSON 序列化工具定义，tagged enum（Function/Namespace/ToolSearch/LocalShell/Freeform/等） |
| `ConfiguredToolSpec` | 同上:132 | `ToolSpec` + `supports_parallel_tool_calls: bool` |
| `ToolsConfig` | `tools/src/tool_config.rs:86` | 全部工具开关的配置结构 |
| `DiscoverableTool` | `tools/src/tool_discovery.rs:67` | 可发现的 MCP connector 或 plugin |

## 3) 工具注册计划：20 步构建流程

`build_tool_registry_plan()`（`tool_registry_plan.rs:72`）按以下顺序构建：

| 步骤 | 工具 | 控制开关 | Handler |
|------|------|---------|---------|
| 1 | `exec` + `wait` (code mode) | `code_mode_enabled` | `CodeModeExecute` + `CodeModeWait` |
| 2 | `shell` / `local_shell` / `exec_command` / `shell_command` | `shell_type` (Default/Local/UnifiedExec/ShellCommand/Disabled) | `Shell` / `ShellCommand` |
| 3 | MCP Resources | MCP tools 存在 | `McpResource` |
| 4 | `update_plan` | 始终 | `Plan` |
| 5 | `get_goal` / `create_goal` / `update_goal` | `goal_tools` | `Goal` |
| 6 | `request_user_input` | 始终 | `RequestUserInput` |
| 7 | `request_permissions` | `request_permissions_tool_enabled` | `RequestPermissions` |
| 8 | `tool_search` | `search_tool` + 有延迟 MCP/dynamic tools | `ToolSearch` |
| 9 | `request_plugin_install` | `tool_suggest` + discoverable | `RequestPluginInstall` |
| 10 | `apply_patch` (Freeform 或 Function) | `has_environment` + `apply_patch_tool_type` | `ApplyPatch` |
| 11 | `list_dir` | `experimental_supported_tools` 含 `"list_dir"` | `ListDir` |
| 12 | `test_sync_tool` | `experimental_supported_tools` 含 `"test_sync_tool"` | `TestSync` |
| 13 | `web_search` | `web_search_mode` 非 disabled | — |
| 14 | `image_gen` | `image_gen_tool` | — |
| 15 | `view_image` | `has_environment` | `ViewImage` |
| 16 | 多 agent V1 或 V2 | `collab_tools` + `multi_agent_v2` | SpawnAgent/SendInput/WaitAgent/等 |
| 17 | Agent jobs | `agent_jobs_tools` | `AgentJobs` |
| 18 | MCP tools (按 namespace) | MCP server 提供 | `Mcp` |
| 19 | Dynamic tools | turn 上下文注入 | `DynamicTool` |
| 20 | Namespace 清理 | `namespace_tools = false` → 剥离 namespace | — |

## 4) 平台差异化（Platform Toolset 等价物）

没有显式的 "toolset" 类型。平台差异通过以下机制实现：

| 机制 | 示例 |
|------|------|
| `cfg!(unix)` / `cfg!(windows)` | `local_tool.rs` 中 shell 描述切换 PowerShell vs bash 语法 |
| `ShellCommandBackendConfig` | `Classic` vs `ZshFork`（仅 Unix 下有 zsh fork） |
| `UnifiedExecShellMode::ZshFork` | Unix-only，需要 zsh path + execve wrapper |
| `conpty_supported()` | Windows ConPTY 检测 |
| `ExperimentalSupportedTools` | 模型级工具白名单（`list_dir`、`test_sync_tool`） |
| Feature flags | `ShellZshFork`、`UnifiedExec`、`ShellTool` 等 bitmask |

## 5) 工具发现（Discovery）

`tools/src/tool_discovery.rs` 实现了延迟加载的 MCP 工具搜索：

**`tool_search` 工具**：模型可以调用它来搜索更多工具。
- 名称：`"tool_search"`
- 默认返回 8 条结果（`TOOL_SEARCH_DEFAULT_LIMIT`）
- 搜索来源：MCP servers + discoverable connectors + discoverable plugins
- 结果按 `ToolSearchResultSource` 分类（server metadata、type、name、description）

**`request_plugin_install` 工具**：模型可以建议安装新 MCP server/plugin。
- 来源：`DiscoverableTool::Connector(AppInfo)` 或 `DiscoverableTool::Plugin(DiscoverablePluginInfo)`

**延迟加载工具**（`defer_loading: true`）：
- 在 `model_visible_specs` 中被隐藏
- 只有通过 `tool_search` 才能发现
- 发现后动态注入到后续 turn 的 tool list

## 6) Code Mode 工具包装

当 `code_mode_enabled = true` 时：
1. 先构建一个内部 plan（`code_mode_enabled = false`, `code_mode_only_enabled = false`），收集所有非 code-mode 工具
2. 把所有嵌套工具的定义写入 `exec` 工具的 freeform description 中
3. 通过 `augment_tool_spec_for_code_mode()` 改写描述为 code-mode 执行示例
4. 如果 `code_mode_only_enabled = true`：只暴露 `exec` + `wait` 给模型，其余全部隐藏

## 7) 源码抓手

| 想看什么 | 从哪里开始 |
|----------|-----------|
| 工具注册计划 | `tools/src/tool_registry_plan.rs:72` — `build_tool_registry_plan()` |
| ToolsConfig 全部字段 | `tools/src/tool_config.rs:86` |
| Shell type 解析 | `tools/src/tool_config.rs:169-188` |
| ToolSpec 定义 | `tools/src/tool_spec.rs:22` |
| ToolHandlerKind 枚举 | `tools/src/tool_registry_plan_types.rs:12` |
| Code mode 包装 | `tools/src/code_mode.rs:13` — `augment_tool_spec_for_code_mode()` |
| 工具发现 | `tools/src/tool_discovery.rs` |
| 延迟工具过滤 | `core/src/tools/router.rs:300` — `filter_deferred_dynamic_tool_spec()` |

## 8) 相关文档

- Dispatch 链路：`../03-agent-core/02-tool-dispatch-and-execution.md`
- Exec 与 Shell：`02-exec-and-shell.md`
- 沙箱机制：`03-sandboxing.md`

---

> 读完这篇，你应该能回答"这个工具为什么出现/为什么不出现"——去 `ToolsConfig` 找对应的布尔开关，或者查 `build_tool_registry_plan()` 的步骤顺序。
