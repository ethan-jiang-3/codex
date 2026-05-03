---
title: "01 — Agent 循环与生命周期（Agent Loop & Lifecycle）"
doc_type: "owner"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解 Agent 从创建到销毁的完整生命周期的人"
purpose: "给出 Agent 实例组成、主循环结构、三种 teardown 语义和完整的生命周期链路"
owns: "Agent 生命周期：创建（spawn）、运行（submission_loop + run_turn）、销毁（shutdown / release / close）三个阶段的源码事实"
update_when:
  - "Session / ThreadManager / Codex 的生命周期管理方式变化时"
  - "Op / Submission / Event 协议类型变化时"
  - "teardown 路径新增或语义变化时"
out_of_scope:
  - "工具分发细节（见 02-tool-dispatch-and-execution.md）"
  - "system prompt 装配（见 03-agent-identity-and-system-prompt.md）"
  - "TUI/CLI 前端如何创建 session（见 08-tui-cli-and-integration）"
---

# 01 — Agent 循环与生命周期（Agent Loop & Lifecycle）

> [!IMPORTANT]
> 这一篇解释 Agent 怎么"活着"——从创建到运行到销毁的完整链路。

> 读完这篇你应该能回答：Agent 实例由哪些组件组成、主循环怎么转、shutdown / release / close 三种退出方式区别在哪。

## 1) 先记住三件事

1. **Agent 实例 = Session + channels + submission_loop**。创建时产生一对 channel（`tx_sub`/`rx_sub` 用于入队命令，`tx_event`/`rx_event` 用于回流事件），一个 tokio 后台任务跑 `submission_loop`。
2. **主循环是双层嵌套的**：外层 `submission_loop` 分发 Op，内层 `run_turn` 执行单轮模型交互（工具调用→模型→工具调用→...）。
3. **退出有三种语义**：`Op::Shutdown`（协议级关闭）、`shutdown_live_agent`（"release"，软删除）、`close_agent`（硬关闭，递归终止子树）。

## 2) Agent 实例的组成

```
CodexThread (公共句柄)
  ├── Codex (公共门面)
  │     ├── tx_sub: Sender<Submission>    // 命令入队
  │     ├── rx_event: Receiver<Event>     // 事件回流
  │     ├── agent_status: watch::Receiver<AgentStatus>
  │     ├── session: Arc<Session>         // 内部状态机
  │     └── session_loop_termination     // 后台循环结束信号
  │
  └── Session (内部状态机)
        ├── SessionState (可变状态：config、history、agent_status)
        ├── SessionServices (Arc 共享依赖注入容器)
        │     ├── AuthManager, ModelsManager
        │     ├── McpConnectionManager, SkillsManager, PluginsManager
        │     ├── UnifiedExecManager, GuardianReviewSessionManager
        │     ├── ToolApprovalsStore, ExecPolicyManager
        │     ├── NetworkProxyService, StateDbBridge
        │     └── ModelClient, EnvironmentManager, HooksManager
        ├── ActiveTurn (当前任务追踪 + CancellationToken)
        └── tx_event: Sender<Event>
```

| 组件 | 角色 | 源码路径 |
|------|------|---------|
| `Codex` | 公共门面：submit / next_event / shutdown_and_wait | `session/mod.rs:367` |
| `CodexThread` | Thread 级句柄：包装 Codex + session_source + rollout_path | `codex_thread.rs:94` |
| `Session` | 内部状态机：持有全部可变状态和服务容器 | `session/session.rs:12` |
| `SessionServices` | DI 容器：27+ Arc 共享服务 | `state/service.rs:37` |
| `ThreadManager` | 顶层编排器：创建/查找/关闭线程 | `thread_manager.rs:210` |
| `TurnContext` | 每 turn 的配置不可变快照（model、sandbox、approval 等） | `session/turn_context.rs` |

## 3) 完整生命周期（7 阶段）

```
Phase A: 线程创建
  ThreadManager::start_thread()
    → ThreadManagerState::spawn_thread_with_source()
      → 加载 plugins、skills、MCP connections
      → Codex::spawn(CodexSpawnArgs { services })
        → 创建 channels (tx_sub/rx_sub, tx_event/rx_event)
        → Session::new(configuration, services)  // 初始化全部子服务
        → tokio::spawn(submission_loop(session, config, rx_sub))
        → 返回 Codex { tx_sub, rx_event, agent_status, session, ... }
      → finalize_thread_spawn()
        → 等待 SessionConfigured 事件
        → 插入 ThreadManager.threads HashMap

Phase B: 主循环启动
  submission_loop() (handlers.rs:962)
    → loop { rx_sub.recv().await → dispatch(sub.op) }
    → 所有非 Shutdown 的 Op 返回 false，循环继续
    → Op::Shutdown 返回 true，循环退出

Phase C: 用户输入到达
  Op::UserTurn / Op::UserInput → user_input_or_turn() (handlers.rs:112)
    → 创建 TurnContext (配置快照)
    → Session::spawn_task(turn_context, RegularTask)

Phase D: Turn 执行
  RegularTask::run() (tasks/regular.rs:40)
    → loop {
        run_turn(session, turn_context, input, ...)
        if has_pending_input() → continue loop
        else → return
      }

Phase E: run_turn 内部 (turn.rs:137)
  1. Pre-turn auto-compaction (ctx window 管理)
  2. 更新 context (skills, MCP tools, mentions)
  3. 构建 ToolRouter (工具列表)
  4. 组装 Prompt (system + history + user input)
  5. ModelClientSession::stream(prompt) → 模型
  6. for event in stream: 文本增量 / 工具调用 / TurnComplete

Phase F: Turn 结束
  → 发送 TurnComplete 事件
  → 持久化 turn history (SQLite + JSONL rollout)
  → 检查 pending_input → 可能需要下一轮

Phase G: 销毁 (见 §5)
```

## 4) 主循环源码锚点

| 想看什么 | 函数 | 文件:行号 |
|----------|------|----------|
| 线程创建入口 | `ThreadManager::start_thread()` | `thread_manager.rs:534` |
| Codex 创建 + channel 初始化 | `Codex::spawn_internal()` | `session/mod.rs:447` |
| Session 构造 | `Session::new()` | `session/session.rs:330` |
| 命令入队 | `Codex::submit()` | `session/mod.rs:673` |
| 事件出队 | `Codex::next_event()` | `session/mod.rs:727` |
| 事件分发循环 | `submission_loop()` | `session/handlers.rs:962` |
| 用户输入处理 | `user_input_or_turn()` | `session/handlers.rs:112` |
| 单轮执行入口 | `run_turn()` | `session/turn.rs:137` |
| RegularTask 循环 | `RegularTask::run()` | `tasks/regular.rs:40` |
| Turn 创建 | `Session::spawn_task()` | `tasks/mod.rs:292` |
| 任务中断 | `Session::abort_all_tasks()` | `tasks/mod.rs:473` |
| Prompt 组装 | `build_prompt()` | `session/turn.rs:944` |
| ToolRouter 创建 | `create_router()` | `session/turn.rs:1242` |

## 5) 三种 Teardown 语义

| | Op::Shutdown | shutdown_live_agent ("release") | close_agent ("close") |
|---|---|---|---|
| **触发方** | 用户/前端直接 | Agent 子系统 | Agent 子系统 |
| **实现位置** | `handlers.rs:872` | `agent/control.rs:702` | `agent/control.rs:722` |
| **中断活动 task** | 是 | 是（通过 Op::Shutdown） | 是（通过 Op::Shutdown） |
| **终止后台进程** | 是 | 是 | 是 |
| **关闭 MCP 连接** | 是 | 是 | 是 |
| **发送 ShutdownComplete** | 是 | 是 | 是 |
| **从 ThreadManager 移除** | 否（调用者负责） | 是 | 是 |
| **释放 AgentRegistry 计数** | 否 | 是 | 是 |
| **持久化 Closed 边状态** | 否 | 否 | **是** |
| **递归关闭子树** | 否 | 否 | **是** |

**Shutdown 内部步骤**（`handlers.rs:872-926`）：
1. `abort_all_tasks(Interrupted)` — 中断当前 task
2. 关闭 realtime conversation manager
3. `terminate_all_processes()` — 终止所有后台进程
4. MCP connection manager begin_shutdown + await
5. Guardian review session shutdown
6. `live_thread.shutdown()` — 刷新并关闭持久化
7. 发送 `EventMsg::ShutdownComplete`
8. 退出 submission_loop

## 6) Channel 架构

```
外部 (TUI/CLI/app-server)
   │
   │  Codex::submit(Submission { op })
   ▼
tx_sub ────→ rx_sub ────→ submission_loop()
  (bounded 512)              │
                             │ 处理 Op
                             ▼
                        Session 内部
                             │
                             │ 产生 Event
                             ▼
tx_event ←────────────────────────────────
  (unbounded)        │
                     ▼
              rx_event ────→ Codex::next_event()
                                 │
                                 ▼
                            外部消费 (TUI 渲染)
```

- **`tx_sub/rx_sub`**：容量 512 的有界 channel，防止命令生产者无限堆积
- **`tx_event/rx_event`**：无界 channel，events 必须被消费但不能阻塞 agent 循环
- **`agent_status`**：tokio watch channel，用于非阻塞地查询 agent 状态

## 7) 源码抓手

| 想看什么 | 从哪里开始 |
|----------|-----------|
| 完整生命周期追踪 | `thread_manager.rs:534` → `session/mod.rs:447` → `session/handlers.rs:962` → `session/turn.rs:137` |
| Channel 初始化 | `session/mod.rs:472-473` |
| Shutdown 全部步骤 | `session/handlers.rs:872-926` |
| Release vs Close 差异 | `agent/control.rs:700-733` |
| Bulk shutdown（批量关闭） | `thread_manager.rs:699-745` |
| Session 状态结构 | `session/session.rs:12` |
| SessionServices（所有依赖） | `state/service.rs:37` |

## 8) 相关文档

- 工具分发链：`02-tool-dispatch-and-execution.md`
- Identity 与 System Prompt：`03-agent-identity-and-system-prompt.md`
- 上下文与压缩：`04-context-and-compression.md`

---

> 读完这篇，你应该能追踪 Agent 从 `ThreadManager::start_thread()` → `submission_loop` → `run_turn` → `Op::Shutdown` 的完整链路。如果想了解工具调用怎么在 turn 内分发，继续看 `02-tool-dispatch-and-execution.md`。
