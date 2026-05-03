---
title: "02 — 命令执行与 Shell（Exec & Shell）"
doc_type: "owner"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解 Codex 如何执行命令、shell 集成和权限提升机制的人"
purpose: "给出命令执行的三种模式、shell 检测与环境快照、Zsh fork 权限提升、以及 exec-server 的进程管理协议"
owns: "命令执行面：三种 shell 模式、shell snapshot 机制、shell escalation、exec-server 协议"
update_when:
  - "新增 shell 执行模式或 ShellCommandBackend 变化时"
  - "shell snapshot 策略变化时"
  - "exec-server 协议新增 method 时"
out_of_scope:
  - "沙箱隔离（见 03-sandboxing.md）"
  - "后台进程管理（见 05-background-processes.md）"
  - "ToolOrchestrator 审批流（见 03-agent-core/02-tool-dispatch-and-execution.md）"
---

# 02 — 命令执行与 Shell（Exec & Shell）

> [!IMPORTANT]
> 这一篇解释 Codex 如何把"执行这个命令"翻译成操作系统的 subprocess。

> 读完这篇你应该能回答：三种 shell 模式有什么区别、shell 环境怎么检测和快照、Zsh fork 怎么实现权限提升。

## 1) 先记住三件事

1. **三种 Shell 模式**：`Default`（普通 array-based shell）、`UnifiedExec`（PTY + 后台进程管理）、`ShellCommand`（字符串命令，支持 Classic 和 ZshFork 两种后端）。由 `ToolsConfig.shell_type` 控制。
2. **Shell Snapshot** 是加速机制：第一次启动 shell 时捕获 `.zshrc`/`.bashrc` 的环境状态（别名、函数、export），后续命令直接注入快照，跳过重复初始化。
3. **Zsh fork** 是 Unix 专属的权限提升路径：通过 patched zsh + execve wrapper + escalation server 的 Unix domain socket 通信，实现按需提升沙箱权限。

## 2) 三种 Shell 模式对比

| | Default (Shell) | UnifiedExec | ShellCommand (Classic) | ShellCommand (ZshFork) |
|---|---|---|---|---|
| **工具名** | `shell` | `exec_command` + `write_stdin` | `shell_command` | `shell_command` |
| **命令格式** | `["ls", "-la"]` | `"ls -la"` (字符串) | `"ls -la"` (字符串) | `"ls -la"` (字符串) |
| **PTY** | 否 | 是 | 否 | 是（zsh fork） |
| **stdin 写入** | 否 | 是（`write_stdin`） | 否 | 否 |
| **后台运行** | 否 | 是 | 否 | 否 |
| **平台** | 全平台 | 全平台 | 全平台 | Unix only |
| **Handler** | `ShellHandler` | `UnifiedExecHandler` | `ShellCommandHandler` | `ShellCommandHandler` |

## 3) Shell 检测与环境快照

### Shell 检测（`core/src/shell_detect.rs`）

`detect_shell_type(path)` 按以下顺序解析：
1. 精确匹配二进制名：`zsh`, `sh`, `cmd`, `bash`, `pwsh`, `powershell`
2. 回退到 `file_stem()` 提取
3. 最终回退：`/bin/sh`（Unix）或 `cmd.exe`（Windows）

### Shell Snapshot（`core/src/shell_snapshot.rs`）

**生命周期**：
1. **捕获**：`start_snapshotting()` → 后台 tokio task 运行 snapshot 脚本
2. **脚本内容**（以 zsh 为例）：
   - `source $ZDOTDIR/.zshrc`
   - 捕获 `functions`、`alias -L`、`setopt`、`export -p`
3. **验证**：snapshot 文件写入后，`set -e` source 测试
4. **注入**：`maybe_wrap_shell_lc_with_snapshot()` 将 `["zsh", "-lc", "cmd"]` 改写为 `["zsh", "-c", ". SNAPSHOT; exec zsh -c cmd"]`
5. **过期**：3 天自动清理（`SNAPSHOT_RETENTION`）

**支持**：Zsh、Bash、Sh、PowerShell（各自不同的捕获脚本）

### 命令包装链

在实际执行前，命令经过多层包装：
1. **Shell snapshot 注入**：`. SNAPSHOT_PATH; exec ...`
2. **UTF-8 编码前缀**：PowerShell 加 `[Console]::OutputEncoding = ...`
3. **沙箱包装**：`sandbox-exec` / `bwrap` / `codex-linux-sandbox` / Windows restricted token
4. **沙箱参数**：`SandboxCommand { program, args, cwd, env, additional_permissions }`

## 4) Shell Escalation：Zsh Fork 权限提升

**Unix only**（`shell-escalation/src/unix/`）。

### 架构

```
EscalateServer
  ├── patched zsh (拦截所有 exec() 调用)
  ├── execve wrapper (通信代理)
  └── escalation policy (决定 Run / Escalate / Deny)

Flow:
  Server启动 → fork zsh → zsh拦截exec → execve wrapper
    → Unix socket通信 → EscalateServer裁决
    → Run (直接执行) / Escalate (服务端代理) / Deny
```

### 协议（`escalate_protocol.rs`）

| 消息 | 方向 | 含义 |
|------|------|------|
| `EscalateRequest { file, argv, workdir, env }` | Wrapper → Server | 被拦截的命令 |
| `EscalateResponse { action }` | Server → Wrapper | 裁决结果 |
| `EscalateAction::Run` | | 直接 exec（无沙箱提升） |
| `EscalateAction::Escalate` | | 服务端代理执行（提权） |
| `EscalateAction::Deny { reason }` | | 拒绝执行 |

### EscalationPolicy（trait）

```rust
async fn determine_action(&self, file, argv, workdir)
    -> EscalationDecision;
```

返回：
- `Run` — 允许 wrappper 直接 exec
- `Escalate(EscalationExecution)` — 服务端代理，可指定 `Unsandboxed`、`TurnDefault` 或 `Permissions(...)`
- `Deny { reason }` — 拒绝执行

## 5) Exec-Server 协议

`exec-server` crate 提供 JSON-RPC 2.0 协议（WebSocket 或 stdin/stdout）：

### 进程管理方法

| Method | 方向 | 含义 |
|--------|------|------|
| `process/start` | Client → Server | 启动进程 |
| `process/read` | Client → Server | 读取输出 |
| `process/write` | Client → Server | 写入 stdin |
| `process/terminate` | Client → Server | 终止进程 |
| `process/output` | Server → Client | 推送输出 chunk |
| `process/exited` | Server → Client | 推送退出通知 |
| `process/closed` | Server → Client | 推送关闭通知 |

### 文件系统方法

| Method | 含义 |
|--------|------|
| `fs/readFile` | 读文件 |
| `fs/writeFile` | 写文件 |
| `fs/createDirectory` | 创建目录 |
| `fs/getMetadata` | 文件元数据 |
| `fs/readDirectory` | 读目录 |
| `fs/remove` | 删除文件/目录 |
| `fs/copy` | 复制文件/目录 |

### 环境抽象

`EnvironmentManager` 管理两种执行环境：
- **Local**（`LOCAL_ENVIRONMENT_ID = "local"`）：使用 `LocalProcess` + `LocalFileSystem`，始终可用
- **Remote**（`REMOTE_ENVIRONMENT_ID = "remote"`）：通过 WebSocket 连接远端 exec-server

## 6) 进程加固（Process Hardening）

`process-hardening/src/lib.rs` 在 `pre_main()` 阶段应用（`#[ctor::ctor]`）：

| 平台 | 加固措施 |
|------|---------|
| **Linux** | `prctl(PR_SET_DUMPABLE, 0)` — 禁 ptrace；`setrlimit(RLIMIT_CORE, 0)` — 禁 core dump；清除 `LD_*` 环境变量 |
| **macOS** | `ptrace(PT_DENY_ATTACH, ...)` — 防调试器挂载；禁 core dump；清除 `DYLD_*` 环境变量 |
| **BSD** | 禁 core dump；清除 `LD_*` 环境变量 |
| **Windows** | 当前为 no-op |

## 7) exec CLI 工具

`exec/src/lib.rs` 提供 `codex exec` 子命令：
- 非交互式单轮执行
- 通过内置 `InProcessAppServerClient` 驱动
- 支持 `--json` 输出 JSONL 事件流
- 事件类型：`ThreadEvent::{ThreadStarted, TurnStarted, TurnCompleted, ItemStarted, ItemUpdated, ItemCompleted, Error}`

## 8) 源码抓手

| 想看什么 | 从哪里开始 |
|----------|-----------|
| Shell handler 实现 | `core/src/tools/handlers/shell.rs:231` — `ShellHandler::handle()` |
| UnifiedExec handler | `core/src/tools/handlers/unified_exec.rs:179` — `UnifiedExecHandler::handle()` |
| Shell 检测 | `core/src/shell_detect.rs` — `detect_shell_type()` |
| Shell snapshot 捕获 | `core/src/shell_snapshot.rs:108` — `try_new()` |
| Snapshot 注入 | `core/src/tools/runtimes/mod.rs:89` — `maybe_wrap_shell_lc_with_snapshot()` |
| Escalation server | `shell-escalation/src/unix/escalate_server.rs:127` — `EscalateServer` |
| Escalation policy trait | `shell-escalation/src/unix/escalation_policy.rs:7` |
| Exec-server 协议 | `exec-server/src/protocol.rs` |
| 进程 spawn | `core/src/spawn.rs` — `spawn_child_async()` |
| Process hardening | `process-hardening/src/lib.rs:13` — `pre_main_hardening()` |
| Exec CLI 入口 | `exec/src/lib.rs:221` — `run_main()` |

## 9) 相关文档

- 工具注册：`01-tool-registry-and-discovery.md`
- 沙箱机制：`03-sandboxing.md`
- 后台进程：`05-background-processes.md`

---

> 读完这篇，你应该能追踪一条 `shell` 命令从工具调用 → ShellHandler → ShellRuntime → shell snapshot 注入 → 沙箱包装 → subprocess spawn 的完整链路。
