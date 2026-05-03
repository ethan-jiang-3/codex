---
title: "02 — 信息流与模块边界（Information Flow & Boundaries）"
doc_type: "owner"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解 Codex 内部数据怎么流动、模块怎么通信的人"
purpose: "给出 Codex 的数据面/控制面划分、从用户输入到 agent 响应的完整信息流、模块边界和持久化写入点"
owns: "Codex 的信息流路径、数据面/控制面划分、模块边界通信方式和持久化写入点"
update_when:
  - "Op/Event/Submission 等核心协议类型变化时"
  - "新增持久化写入点或模块通信方式变化时"
  - "信息流路径出现新的重要分支时"
out_of_scope:
  - "具体工具的执行细节（见 04-tools-execution-and-sandboxing）"
  - "模型 API 适配细节（见 06-models-and-providers）"
  - "TUI 渲染细节（见 08-tui-cli-and-integration）"
---

# 02 — 信息流与模块边界（Information Flow & Boundaries）

> [!IMPORTANT]
> 这一篇解释 Codex 内部的数据流转——模块之间怎么说话，边界怎么画。

> 读完这篇你应该能回答：用户输入触发了几层处理、中间经过了哪些队列/通道、最终结果怎么回到前端。

## 1) 先分清三件事：入口、协议、核心

Codex 的所有外部交互通过 **4 个入口面** 进入，每一面都翻译为同一套内部协议（`Op` / `EventMsg`），再交给 `core` 处理。

```
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│   TUI   │  │   CLI   │  │  HTTP   │  │   MCP   │
│(ratatui)│  │ (clap)  │  │ (axum)  │  │(stdio)  │
└────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘
     │            │            │            │
     │  JSON-RPC  │  直接调用   │  JSON-RPC  │  MCP协议
     │            │            │            │
     └────────────┼────────────┼────────────┘
                  │            │
          ┌───────┴────────────┴───────────┐
          │    app-server-protocol         │
          │  ClientRequest → Op            │
          │  EventMsg → ServerNotification │
          └───────────────┬───────────────┘
                          │
                  ┌───────┴────────┐
                  │   codex-core   │
                  │  submission_   │
                  │  loop          │
                  └────────────────┘
```

**关键翻译点**：

| 方向 | 入口格式 | 内部格式 | 翻译位置 |
|------|---------|---------|---------|
| 请求 → | `ClientRequest` (JSON-RPC method) | `Op` (内部枚举) | `app-server-protocol` |
| 请求 → | CLI args | `Op` | 各子命令模块 |
| ← 响应 | `EventMsg` (内部枚举) | `ServerNotification` (JSON-RPC) | `core-api/src/lib.rs` → `item_event_to_server_notification()` |

## 2) 数据面 vs 控制面

```
┌─────────────────────────────────────────────────────────┐
│                      控制面 (Control Plane)               │
│                                                         │
│  ThreadManager.start_thread() / fork() / resume()       │
│  Session.configure() / shutdown()                       │
│  TurnContext (model, sandbox, approval 等配置快照)        │
│  ConfigLayerStack 加载与合并                              │
│  Guardian auto-review 策略判断                            │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                      数据面 (Data Plane)                  │
│                                                         │
│  UserInput → run_turn() → Prompt → LLM → stream         │
│  ToolCall → ToolRouter → ToolInvocation → exec          │
│  EventMsg 流 (AgentMessage, CommandExecDelta, 等)        │
│  ThreadItem 历史记录 (user/agent/tool messages)           │
│  rollout JSONL 文件写入                                   │
│  SQLite thread-store 持久化                               │
└─────────────────────────────────────────────────────────┘
```

**数据面**处理"用户说了什么、模型回复了什么、工具执行了什么"。走的是低延迟的 tokio channel + stream。

**控制面**处理"用什么配置跑、什么安全策略、什么时候压缩上下文"。走的是结构化的配置快照、Arc 共享状态、以及 `TurnContext` 不可变快照。

## 3) 从用户输入到 Agent 响应：完整信息流

以下追踪一条用户输入在 Codex 内部经过的所有处理节点。

```
用户输入 "帮我重构这个函数"
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│ 1. 前端捕获                                              │
│                                                         │
│ TUI: ratatui 事件循环捕获键盘输入                          │
│ → 构造 ClientRequest::ThreadStart 或直接提交输入           │
│ → 打包为 JSON-RPC，通过 app-server-transport 发送         │
│                                                         │
│ CLI: MultitoolCli::parse() → 提取 config_overrides       │
│ → 调用对应子命令模块                                       │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│ 2. JSON-RPC → 内部 Op 转换                               │
│                                                         │
│ app-server-protocol 将 ClientRequest 转为内部 Op:         │
│   ThreadStart → ThreadManager.start_thread()            │
│   用户消息 → Op::UserTurn { input, turn_context }        │
│   继续输入 → Op::UserInput { items }                     │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│ 3. Codex::submit() — 入队                                │
│                                                         │
│ let submission = Submission { id, op, trace }           │
│ tx_sub.send(submission)  // → 进入 submission_loop       │
│                                                         │
│ 这个 channel 是控制面的入口。所有 Op 都通过它排队。         │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│ 4. submission_loop — 事件分发                             │
│                                                         │
│ session/handlers.rs: submission_loop()                  │
│   match op {                                            │
│     Op::UserTurn { .. } => user_input_or_turn()        │
│     Op::Interrupt => interrupt_current_turn()          │
│     Op::Shutdown => break                              │
│   }                                                     │
│                                                         │
│ → 创建 TurnContext（配置不可变快照）                       │
│ → 选择合适的 SessionTask（RegularTask/ReviewTask/等）     │
│ → tokio::spawn(task)                                    │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│ 5. run_turn() — 核心循环                                 │
│                                                         │
│ session/turn.rs: run_turn(session, turn_context, input) │
│                                                         │
│ 5a. Pre-turn 预处理：                                    │
│   - 检查是否需要 auto-compaction（上下文管理）             │
│   - 刷新 MCP 工具列表                                    │
│   - 解析 mentions（@plugin、@skill、@connector）          │
│   - 加载 context fragments（system prompt 模块）         │
│                                                         │
│ 5b. 构建 ToolRouter：                                    │
│   - 内置工具 + MCP 工具 + 可发现工具 + 动态工具             │
│   - 根据 TurnContext 过滤可用工具                          │
│                                                         │
│ 5c. 组装 Prompt：                                        │
│   - System：agent identity + tool guidance + skills     │
│     + plugin injection + memory + context files         │
│   - Messages：压缩后的历史消息 + 当前用户输入               │
│                                                         │
│ 5d. 模型调用：                                           │
│   ModelClientSession::stream(prompt)                    │
│   → HTTP 流式请求 → SSE/WebSocket → ResponseEvent stream │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│ 6. 响应处理循环                                           │
│                                                         │
│ for event in response_stream {                          │
│   match event {                                         │
│     ResponseEvent::Message => 文本增量                    │
│       → tx_event.send(EventMsg::AgentMessageDelta)      │
│                                                         │
│     ResponseEvent::ToolCall => 工具调用                   │
│       → ToolRouter.route(tool_name, args)               │
│       → ToolInvocation (session, turn, call_id)         │
│       → ToolOrchestrator.execute(invocation)            │
│         → Approval check (guardian or user prompt)      │
│         → Sandbox selection                             │
│         → ToolRuntime execution                         │
│       → Result → ResponseInputItem                      │
│                                                         │
│     ResponseEvent::TurnComplete => 完成                   │
│       → tx_event.send(EventMsg::TurnComplete)           │
│   }                                                     │
│ }                                                       │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│ 7. 事件回流                                               │
│                                                         │
│ tx_event channel → CodexThread.next_event()             │
│ → item_event_to_server_notification()                   │
│ → ServerNotification (JSON-RPC) → app-server-transport  │
│ → 推送到前端（TUI / HTTP client）                         │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                    ┌─────────┐
                    │  前端    │
                    │ 渲染更新  │
                    └─────────┘
```

## 4) 持久化写入点

Codex 有 **5 个持久化写入面**，按写入时机区分：

| 写入点 | 存储 | 写入时机 | 关键模块 |
|--------|------|---------|---------|
| Thread state | SQLite（`thread-store`） | Thread 创建、每 turn 结束、配置变更 | `thread-store`、`state_db_bridge` |
| Turn history | SQLite + JSONL rollout | 每 turn 完成后追加 | `rollout.rs` → JSONL 文件 |
| Memory | 文件系统（MEMORY.md 等） | Session 边界或显式触发 | `memories/write` |
| User config | TOML 文件 | 用户显式修改或程序写入 | `config` |
| Analytics | 远端（HTTP） | 事件发生后异步上报 | `analytics` |

**SQLite schema 核心表**（位于 `thread-store` crate 的 migration）：

| 表 | 内容 |
|----|------|
| `threads` | Thread ID、名称、状态、创建/更新时间 |
| `turns` | 每 turn 的 ID、序号、模型、状态、token 用量 |
| `turn_items` | Turn 内的结构化 item（user message / agent message / tool call / 等） |
| `sessions` | Session 元数据（配置快照、环境信息） |

**JSONL rollout**：`$CODEX_HOME/rollouts/<thread_id>/` 下按 turn 追加 JSON 行，每行一个 `TurnItem`。与 SQLite 形成**双写**：SQLite 用于结构化查询和 resume，JSONL 用于调试回放和外部消费。

## 5) 模块边界与通信方式

### 5.1 进程内通信（IPC 替代方案）

Codex 的所有 Rust 组件运行在一个进程内。各模块之间通过以下方式通信：

| 通信方式 | 适用场景 | 示例 |
|---------|---------|------|
| **tokio mpsc channel** | 跨任务异步消息 | `tx_sub/rx_sub`（Op 入队）、`tx_event/rx_event`（事件回传） |
| **Arc + RwLock/Mutex** | 共享只读/读写状态 | `SessionState`、`ThreadManagerState`、`SessionServices` |
| **tokio oneshot channel** | 请求-响应（一问一答） | Approval 请求：发送 `ExecApproval` Op，等待用户响应 |
| **trait 抽象** | 插件式替换 | `ModelProvider` trait、`ToolRuntime` trait、`ThreadConfigLoader` trait |
| **watch channel** | 状态变更通知 | `AgentStatus` watch（agent 运行/中断/完成） |
| **tokio broadcast** | 一对多广播 | Config 热重载通知 |

### 5.2 关键模块边界（Cross-Boundary Contracts）

```
┌──────────────────┬───────────────────────────────────────┐
│      边界         │              契约                      │
├──────────────────┼───────────────────────────────────────┤
│ TUI ↔ app-server │ JSON-RPC 2.0，ClientRequest /         │
│                  │ ServerNotification，stdio 或 WebSocket │
├──────────────────┼───────────────────────────────────────┤
│ app-server ↔     │ Op (input) / EventMsg (output)，      │
│ core             │ 通过 Codex::submit() 和 next_event()   │
├──────────────────┼───────────────────────────────────────┤
│ core ↔ tools     │ ToolRouter.route() → ToolInvocation   │
│                  │ → ToolRuntime trait 实现                │
├──────────────────┼───────────────────────────────────────┤
│ core ↔ model-    │ ModelClientSession::stream()          │
│ provider         │ → ResponseEvent stream                │
├──────────────────┼───────────────────────────────────────┤
│ core ↔ sandboxing│ ExecRequest → execute_env()           │
│                  │ → sandboxed command → subprocess      │
├──────────────────┼───────────────────────────────────────┤
│ core ↔ state/    │ state_db_bridge (SQLite)、            │
│ thread-store     │ rollout.rs (JSONL)                    │
├──────────────────┼───────────────────────────────────────┤
│ core ↔ config    │ ConfigLayerStack → Config (deserialize)│
│                  │ → SessionConfiguration                │
├──────────────────┼───────────────────────────────────────┤
│ core ↔ MCP       │ McpConnectionManager                  │
│                  │ → list_tools() → tool specs            │
│                  │ → call_tool() → result                  │
├──────────────────┼───────────────────────────────────────┤
│ core ↔ skills/   │ SkillsManager::injections_for_turn()  │
│ plugins          │ → system prompt 文本注入               │
└──────────────────┴───────────────────────────────────────┘
```

### 5.3 安全边界（Trust Boundaries）

```
      ┌──────────────────────────────┐
      │      Untrusted 输入           │
      │  用户 prompt / 文件内容 /      │
      │  MCP 工具结果 / Web 内容       │
      └─────────────┬────────────────┘
                    │
    ┌───────────────┴────────────────┐
    │         Trust 边界 #1           │
    │  Guardian (LLM 审查)            │
    │  + execpolicy (用户定义规则)     │
    │  + AskForApproval (策略决策)     │
    └───────────────┬────────────────┘
                    │
    ┌───────────────┴────────────────┐
    │         Trust 边界 #2           │
    │  沙箱隔离 (OS-level)             │
    │  Landlock / Seatbelt /          │
    │  AppContainer                   │
    └───────────────┬────────────────┘
                    │
            ┌───────┴───────┐
            │   Subprocess   │
            │ (受约束执行)     │
            └───────────────┘
```

## 6) 源码抓手（Source Code Handholds）

| 想看什么 | 从哪里开始 |
|----------|-----------|
| 完整信息流追踪 | `codex-rs/core/src/session/turn.rs` → `run_turn()` (line ~137) |
| Op/Event 协议定义 | `codex-rs/protocol/src/protocol.rs` |
| JSON-RPC 请求转换 | `codex-rs/app-server-protocol/src/protocol/` |
| 事件到通知映射 | `codex-rs/core-api/src/lib.rs` → `item_event_to_server_notification()` |
| 事件分发循环 | `codex-rs/core/src/session/handlers.rs` → `submission_loop()` |
| Session 创建 | `codex-rs/core/src/session/mod.rs` → `Codex::spawn()` (line ~423) |
| Turn 上下文字段 | `codex-rs/core/src/session/turn_context.rs` |
| SessionServices（DI 容器） | `codex-rs/core/src/state/service.rs` → `SessionServices` (line ~37) |
| 持久化桥接 | `codex-rs/core/src/state_db_bridge.rs` |
| Rollout JSONL 写入 | `codex-rs/core/src/rollout.rs` |
| Thread store schema | `codex-rs/thread-store/src/` → migration files |
| Channel 定义 | `codex-rs/core/src/session/mod.rs` → `tx_sub`/`rx_event` 等字段 |

## 7) 相关文档

- 系统全景：`01-system-landscape.md`
- Agent 核心循环：`../03-agent-core/01-agent-loop-and-lifecycle.md`（待生产）
- 工具分发：`../04-tools-execution-and-sandboxing/01-tool-registry-and-dispatch.md`（待生产）

---

> 读完这篇，你应该能追踪一条用户输入从 TUI/CLI → Op → submission_loop → run_turn → EventMsg → 前端渲染的完整路径。如果想深挖 agent 循环的具体实现，继续看 `03-agent-core/`。
