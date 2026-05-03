---
title: "02 — 工具分发与执行（Tool Dispatch & Execution）"
doc_type: "owner"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解工具如何被发现、如何分发给 handler、以及有哪些可用性闸门的人"
purpose: "给出工具 schema 的三条来源路径、ToolRouter 的组装逻辑、从模型输出到 handler 执行的完整 dispatch 链、以及 25 个可用性过滤点"
owns: "工具系统的分发面：schema 来源、registry 注册、dispatch 链路、可用性闸门"
update_when:
  - "新增工具类型或工具注册机制变化时"
  - "dispatch 链路新增环节时"
  - "可用性过滤条件变化时"
out_of_scope:
  - "具体工具的 handler 实现细节"
  - "沙箱/exec 的底层机制（见 04-tools-execution-and-sandboxing）"
  - "MCP 协议细节（见 08-tui-cli-and-integration）"
---

# 02 — 工具分发与执行（Tool Dispatch & Execution）

> [!IMPORTANT]
> 这一篇解释 Codex 如何把"模型说我要调用某个工具"翻译成"真的跑了某个 handler 并产生了结果"。

> 读完这篇你应该能回答：工具列表怎么来的、一个 ToolCall 经过哪些环节才执行、为什么某些工具在某些场景下不可用。

## 1) 先记住三件事

1. **工具 schema 有三条来源**：内置工具（config-gated）、MCP 工具（server 发现）、动态工具（turn 上下文注入）。ToolRouter 每 turn 重新组装。
2. **Dispatch 是 7 阶段链路**：模型输出解析 → ToolCall 构造 → ToolCallRuntime 并发控制 → Invocation 包装 → Registry 查表 → Orchestrator 审批-沙箱循环 → Handler 执行。
3. **没有字面常量 `AGENT_LOOP_TOOLS`**。替代机制是：统一 dispatch 管线 + code_mode 工具包装 + apply_patch 拦截 + unavailable tool 占位。

## 2) 工具 Schema 的三条来源

```
┌──────────────────────────────────────────────────────────┐
│                    ToolRouter::from_config()               │
│                                                           │
│  输入:                                                     │
│  ┌─────────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ built-in tools   │  │  MCP tools   │  │dynamic tools │ │
│  │ (config-gated)   │  │ (server 发现) │  │ (turn 上下文) │ │
│  └────────┬────────┘  └──────┬───────┘  └──────┬───────┘ │
│           │                  │                  │          │
│           ▼                  ▼                  ▼          │
│     build_tool_registry_plan()  +  deferred MCP tools     │
│           │                                              │
│           ▼                                              │
│     build_specs_with_discoverable_tools()                │
│           │                                              │
│           ▼                                              │
│     ToolRegistry + model_visible_specs                    │
└──────────────────────────────────────────────────────────┘
```

**来源 1 — 内置工具（built-in）**

定义在 `tools/src/tool_registry_plan.rs:72` 的 `build_tool_registry_plan()`。每个工具由 `ToolsConfig` 中的布尔标志控制开关。示例：

| 工具 | 控制开关 |
|------|---------|
| `shell` / `local_shell` / `unified_exec` | `ToolsConfig.shell_type` |
| `apply_patch` | `ToolsConfig.apply_patch_tool_type` |
| `view_image` | `ToolsConfig.has_environment` |
| `web_search` | `ToolsConfig.web_search_mode` |
| `spawn_agent` / `collab_*` | `ToolsConfig.collab_tools` / `multi_agent_v2` |
| `exec` + `wait` (code mode) | `ToolsConfig.code_mode_enabled` |

**来源 2 — MCP 工具（外部发现）**

通过 `McpConnectionManager` 从已连接的 MCP server 获取工具列表。传入 `ToolRouterParams` 时分为两类：
- `mcp_tools: HashMap<String, ToolInfo>` — 直接可用
- `deferred_mcp_tools: HashMap<String, ToolInfo>` — 延迟加载（通过 `tool_search` 触发）

MCP 工具被转换为 `ResponsesApiTool` 并按 server namespace 分组（`ToolSpec::Namespace`）。

**来源 3 — 动态工具（turn 级注入）**

从 `turn_context.dynamic_tools: Vec<DynamicToolSpec>` 传入。支持 `defer_loading: true` 来对模型隐藏（只通过 `tool_search` 暴露）。

## 3) ToolRouter 组装逻辑

每 turn 构建一个新的 `ToolRouter`（`turn.rs:1242`，`create_router()`）：

1. **构建 plan**：`build_tool_registry_plan()` 根据 `ToolsConfig` 逐工具决定是否添加 spec + handler。
2. **Code mode 包装**：如果 `code_mode_enabled`，先构建嵌套工具的内部 plan，然后包装进 `exec` 工具的 freeform description 中。
3. **MCP namespace 处理**：如果 `namespace_tools = false`，剥离所有 `ToolSpec::Namespace`。
4. **构建 registry**：`build_specs_with_discoverable_tools()` 把 plan 转为 `ToolRegistry` + `Vec<ConfiguredToolSpec>`。
5. **过滤 model_visible_specs**：两个过滤器 —
   - **Code-mode-only filter**：`code_mode_only_enabled` 时，只暴露 `exec` 和 `wait`，隐藏全部嵌套工具
   - **Deferred-dynamic-tool filter**：隐藏 `defer_loading: true` 的 dynamic tools

6. **Unavailable tool 占位**：当 `Feature::UnavailableDummyTools` 开启时，为模型调用了但不可用的工具创建假的 spec + `UnavailableToolHandler`。

## 4) Dispatch 链路（7 阶段）

```
Phase A: 模型输出解析
  stream_events_utils.rs:220 — handle_output_item_done()
    ResponseItem::FunctionCall / CustomToolCall / LocalShellCall / ToolSearchCall
    → ToolRouter::build_tool_call(session, item)

Phase B: ToolCall 构造
  router.rs:176 — build_tool_call()
    匹配 ResponseItem 变体 → 解析 namespace → 构造 ToolPayload + ToolCall

Phase C: 并发控制
  parallel.rs:83 — ToolCallRuntime::handle_tool_call_with_source()
    → 检查 router.tool_supports_parallel(&call) → 决定 read/write 锁
    → tokio::select! { call, cancellation } 并发模型

Phase D: Invocation 包装
  router.rs:270 — dispatch_tool_call_with_code_mode_result()
    → 构造 ToolInvocation { session, turn, cancellation_token, call_id, tool_name, source, payload }
    → registry.dispatch_any(invocation)

Phase E: Registry 查表
  registry.rs:265 — dispatch_any()
    1. 按 tool_name 查 handler
    2. 验证 handler.matches_kind(&payload)
    3. 执行 pre-tool-use hooks
    4. 检查 is_mutating → 等待 tool_call_gate (文件监控静默窗口)
    5. handler.handle_any(invocation)
    6. 执行 post-tool-use hooks

Phase F: Orchestrator 审批-沙箱循环
  orchestrator.rs:126 — ToolOrchestrator::run()
    1. 评估 ExecApprovalRequirement (Skip / NeedsApproval / Forbidden)
    2. 需要审批 → Guardian 自动审查 或 用户弹窗
    3. 首次尝试：sandboxing.select_initial() → tool.run()
    4. 被拒且 escalate_on_failure → 申请无沙箱模式 → 重试

Phase G: Result → ResponseInputItem
  parallel.rs:74 — 转换为模型可消费的 tool result
```

## 5) 可用性过滤：25 个 Gate

### 配置级 Gate（ToolRouter 构建时）

| # | Gate | 位置 | 影响 |
|---|------|------|------|
| 1 | `has_environment` | `tool_registry_plan.rs:138` | 无环境 → 禁用 shell/exec/apply_patch 等 |
| 2 | `shell_type` | `tool_registry_plan.rs:139-184` | 选择 shell/local_shell/unified_exec/Disabled |
| 3 | `shell_type = Disabled` | 同上:173 | 所有 shell 禁用 |
| 4 | `apply_patch_tool_type` | 同上:328-346 | 控制 patch 类型 |
| 5 | `search_tool` + deferred | 同上:274-308 | 有延迟工具才启用 tool_search |
| 6 | `tool_suggest` | 同上:310-325 | 控制 request_plugin_install |
| 7 | `namespace_tools` | 同上:601-604 | false → 剥离 MCP namespace |
| 8 | `code_mode_enabled` | 同上:79-136 | 控制 exec/wait 工具 |
| 9 | `code_mode_only_enabled` | `router.rs:82-86` | 只暴露 exec/wait |
| 10 | `goal_tools` | `tool_registry_plan.rs:221-240` | 控制 goal 工具 |
| 11 | `collab_tools` | 同上:407-494 | 控制多 agent 工具 |

### 运行时 Gate（ToolRouter 构建时 / Handler 执行时）

| # | Gate | 位置 | 影响 |
|---|------|------|------|
| 12 | `multi_agent_v2` | `tool_registry_plan.rs:408,454` | v1 vs v2 schema |
| 13 | `agent_jobs_tools` | 同上:497-512 | 批处理任务工具 |
| 14 | `image_gen_tool` | 同上:388-395 | 图像生成 |
| 15 | `web_search_mode` | 同上:376-386 | 网络搜索 |
| 16 | `experimental_supported_tools` | 同上:350-374 | list_dir / test_sync_tool |
| 17 | `request_permissions_tool_enabled` | 同上:254-261 | request_permissions |
| 18 | MCP 连接状态 | `turn.rs:1206-1214` | 无连接 → 无 MCP 工具 |
| 19 | `defer_loading` | `router.rs:300-331` | 对模型隐藏 |
| 20 | `UnavailableDummyTools` feature | `turn.rs:1218` | 不可用工具占位 |
| 21 | `AskForApproval` 策略 | `orchestrator.rs:148-213` | 控制是否需要审批 |
| 22 | `exec_approval_requirement()` | `sandboxing.rs:308` | 每工具的审批覆盖 |
| 23 | 文件系统 sandbox 策略 | `orchestrator.rs:146-149` | 限制模式下触发审批 |
| 24 | MCP `supports_parallel_tool_calls` | `router.rs:165-172` | 控制并行 |
| 25 | `tool_call_gate` (文件监控) | `registry.rs:389-393` | 修改工具等待文件稳定 |

## 6) AGENT_LOOP_TOOLS 概念澄清

Codex 代码库中**没有 `AGENT_LOOP_TOOLS` 字面常量**。工具拦截通过以下机制实现：

| 机制 | 说明 |
|------|------|
| **统一 dispatch 管线** | 所有工具调用走同一条路径：`handle_output_item_done` → `ToolRouter` → `ToolRegistry`，没有特殊 bypass |
| **Code mode 工具包装** | `code_mode_only_enabled` 时，模型只看到 `exec` + `wait`；嵌套工具调用由 code_mode runtime 内部发出（标记 `ToolCallSource::CodeMode`） |
| **apply_patch 拦截** | Shell/unified_exec handler 内部调用 `intercept_apply_patch()` 检测 patch 命令并重路由 |
| **Unavailable tool 占位** | 模型调用了不存在的工具 → `UnavailableToolHandler` 返回友好错误 |

## 7) 源码抓手

| 想看什么 | 从哪里开始 |
|----------|-----------|
| 内置工具注册计划 | `tools/src/tool_registry_plan.rs:72` — `build_tool_registry_plan()` |
| ToolRouter 构建 | `core/src/tools/router.rs:56` — `ToolRouter::from_config()` |
| Spec + Registry 构建 | `core/src/tools/spec.rs:71` — `build_specs_with_discoverable_tools()` |
| 模型输出→ToolCall | `core/src/tools/router.rs:176` — `build_tool_call()` |
| Invocation→Handler | `core/src/tools/router.rs:270` — `dispatch_tool_call_with_code_mode_result()` |
| Registry 查表 | `core/src/tools/registry.rs:265` — `dispatch_any()` |
| 审批+沙箱循环 | `core/src/tools/orchestrator.rs:126` — `ToolOrchestrator::run()` |
| 并发控制 | `core/src/tools/parallel.rs:83` — `handle_tool_call_with_source()` |
| Prompt 包含工具列表 | `core/src/session/turn.rs:952` — `build_prompt()` |
| 模型输出解析 | `core/src/stream_events_utils.rs:220` — `handle_output_item_done()` |
| Handler trait 定义 | `core/src/tools/registry.rs:88` — `ToolHandler::handle()` |

## 8) 相关文档

- Agent 循环：`01-agent-loop-and-lifecycle.md`
- Identity 与 System Prompt：`03-agent-identity-and-system-prompt.md`
- 沙箱机制：`../04-tools-execution-and-sandboxing/03-sandboxing.md`（待生产）
- Exec 工具细节：`../04-tools-execution-and-sandboxing/02-exec-and-shell.md`（待生产）

---

> 读完这篇，你应该能追踪一个工具调用从模型的 `FunctionCall` → `ToolRouter` → `ToolRegistry` → `ToolOrchestrator` → handler 执行 → 结果返回模型的完整链路。25 个可用性 gate 解释了"为什么这个工具看不见/不让用"。
