---
title: "05 — 后台进程管理（Background Processes）"
doc_type: "owner"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解 Codex 如何管理长时间运行的后台进程的人"
purpose: "给出 UnifiedExec 后台进程系统的完整架构：进程生命周期、输出缓冲（HeadTailBuffer）、异步监控、进程修剪策略和网络审批监控"
owns: "后台进程管理面：UnifiedExecProcess 生命周期、HeadTailBuffer、输出流监控、进程修剪、网络审批联动"
update_when:
  - "UnifiedExec 进程管理模式变化时"
  - "HeadTailBuffer 策略或容量限制变化时"
  - "新增后台进程通知/监控机制时"
out_of_scope:
  - "exec 工具的 handler 实现（见 02-exec-and-shell.md）"
  - "沙箱隔离（见 03-sandboxing.md）"
---

# 05 — 后台进程管理（Background Processes）

> [!IMPORTANT]
> 这一篇解释 Codex 的 UnifiedExec 系统如何管理从 spawn 到 terminate 的完整进程生命周期。

> 读完这篇你应该能回答：后台进程怎么创建、输出怎么缓冲和流式推送、进程太多怎么办、网络审批怎么联动终止进程。

## 1) 先记住三件事

1. **UnifiedExec 是 PTY 级进程管理**：支持 TTY 分配、yield、stdin 写入、后台运行，区别于普通 shell 工具的一次性执行。
2. **HeadTailBuffer 是核心数据结构**：50% 头 + 50% 尾的容量分配，丢弃中间省略部分，限 1 MiB 总容量。
3. **进程数有硬上限**：最多 64 个进程，满时先清已退出进程（LRU），仍不够则清最旧非保护进程。

## 2) 系统架构

```
UnifiedExecProcessManager (Mutex<ProcessStore>)
  ├── ProcessStore { processes: HashMap<i32, ProcessEntry>, reserved_ids: HashSet<i32> }
  │
  └── UnifiedExecProcess (per process)
        ├── HeadTailBuffer (output retention, 1 MiB cap)
        ├── broadcast::channel(64) (output streaming to consumers)
        ├── watch::channel<ProcessState> (state observation)
        ├── CancellationToken (exit signal)
        ├── Notify (output_drained signal)
        └── Arc<AtomicBool> (output_closed flag)
```

## 3) 关键常量和限制

| 常量 | 值 | 含义 |
|------|----|------|
| `UNIFIED_EXEC_OUTPUT_MAX_BYTES` | 1 MiB | 单进程输出缓冲上限 |
| `MAX_UNIFIED_EXEC_PROCESSES` | 64 | 最大并发进程数 |
| `WARNING_UNIFIED_EXEC_PROCESSES` | 60 | 警告阈值 |
| `MIN_YIELD_TIME_MS` | 250ms | 最短产出等待 |
| `MIN_EMPTY_YIELD_TIME_MS` | 5000ms | 空输出的最短等待 |
| `MAX_YIELD_TIME_MS` | 30000ms | 最长产出等待 |
| `DEFAULT_MAX_BACKGROUND_TERMINAL_TIMEOUT_MS` | 5min | 后台终端默认超时 |
| `UNIFIED_EXEC_OUTPUT_DELTA_MAX_BYTES` | 8192 | 单次 delta 事件最大字节 |
| `EARLY_EXIT_GRACE_PERIOD` | 150ms | 进程立即退出的优雅期 |

## 4) 进程生命周期

```
1. allocate_process_id()
   → 随机 ID (1000..100000)，冲突重试，加入 reserved_ids

2. open_session_with_sandbox()
   → ToolOrchestrator 审批 → 沙箱选择 → 创建 subprocess

3. UnifiedExecProcess::new()
   → 分配 HeadTailBuffer, Notify, CancellationToken, watch channel

4. from_spawned() / from_exec_server_started()
   → 本地 PTY：spawn_local_output_task() 泵取输出
   → 远程 exec-server：spawn_exec_server_output_task() 轮询 process.read()
   → 检查早期退出（150ms 优雅期）

5. start_streaming_output()
   → 后台 task 从 broadcast::channel 读取输出 chunk
   → split_valid_utf8_prefix() 按 UTF-8 边界分片
   → 发送 ExecCommandOutputDeltaEvent 事件

6. exec_command() 收集初始输出
   → collect_output_until_deadline() 等待 yield_time_ms

7. 运行中
   → write_stdin() 发送输入
   → 网络审批监控 task 运行中

8. 进程退出
   → spawn_exit_watcher() 检测 exit_token + output_drained
   → 发送 ExecCommandEnd 事件

9. 清理
   → Drop impl 调用 terminate()
   → 取消 token, 终止 subprocess, 释放 process_id
```

## 5) HeadTailBuffer：输出缓冲策略

容量分配：
- **Head（50%）**：保留最早输出的字节
- **Tail（50%）**：保留最新输出的字节
- **中间省略**：超容量时丢弃，记录 omitted_bytes

方法：
- `push_chunk()` — 先填 head，溢出后写 tail（最旧 tail 被驱逐）
- `snapshot_chunks()` — head→tail 顺序，返回 `Vec<Vec<u8>>`
- `drain_chunks()` — 消费全部，重置状态

**设计目的**：模型需要看到命令输出的**开头**（判断是否正常启动）和**结尾**（判断是否成功），中间部分可以省略。

## 6) 流式输出监控（`async_watcher.rs`）

`start_streaming_output()` 后台 task：
1. 从 `broadcast::Receiver` 接收 output chunks
2. 在 `pending: Vec<u8>` 中累积
3. 用 `split_valid_utf8_prefix()` 沿 UTF-8 边界切割
4. 追加到 `transcript: HeadTailBuffer`
5. 发送 `ExecCommandOutputDeltaEvent`（单次最多 `MAX_EXEC_OUTPUT_DELTAS_PER_CALL` 个 delta）
6. 进程退出后：等待 `TRAILING_OUTPUT_GRACE` (100ms) 捕获尾部输出 → 发 `output_drained` 信号

`split_valid_utf8_prefix()`：从 max_bytes 向下尝试 UTF-8 切割点，4 字节内找不到合法边界则逐字节发送以避免卡死。

## 7) 进程修剪（Pruning）

`prune_processes_if_needed()`：
- 触发：进程数达到 `MAX_UNIFIED_EXEC_PROCESSES` (64)
- 策略：
  1. 先清理已退出进程（按 `last_used` LRU 排序）
  2. 保护最近使用的 8 个进程
  3. 仍不够 → 清理非保护进程中最旧的

## 8) 网络审批联动

UnifiedExec 支持延迟（deferred）网络审批：

1. `UnifiedExecRuntime::network_approval_spec()` 返回 `NetworkApprovalMode::Deferred`
2. 进程先启动，网络权限稍后裁决
3. `terminate_process_on_network_denial()` 后台 task race：
   - 网络审批被拒 → `process.fail(denial_message)` + 释放 process_id
   - 进程自己退出 → 无操作

这允许进程快速启动而不必等待用户的网络审批决策。

## 9) 源码抓手

| 想看什么 | 从哪里开始 |
|----------|-----------|
| ProcessManager 主体 | `core/src/unified_exec/process_manager.rs` |
| exec_command 流程 | `core/src/unified_exec/process_manager.rs:369` |
| write_stdin 流程 | `core/src/unified_exec/process_manager.rs:597` |
| 进程创建 | `core/src/unified_exec/process.rs:101` — `UnifiedExecProcess::new()` |
| 进程状态 | `core/src/unified_exec/process_state.rs` |
| HeadTailBuffer | `core/src/unified_exec/head_tail_buffer.rs` |
| 流式输出 | `core/src/unified_exec/async_watcher.rs:40` — `start_streaming_output()` |
| 退出监控 | `core/src/unified_exec/async_watcher.rs:107` — `spawn_exit_watcher()` |
| UTF-8 分片 | `core/src/unified_exec/async_watcher.rs:284` — `split_valid_utf8_prefix()` |
| 进程修剪 | `core/src/unified_exec/process_manager.rs:1197` — `prune_processes_if_needed()` |
| 全部终止 | `core/src/unified_exec/process_manager.rs:1244` — `terminate_all_processes()` |
| 网络审批联动 | `core/src/unified_exec/process_manager.rs:308` — `terminate_process_on_network_denial()` |
| 模块常量 | `core/src/unified_exec/mod.rs:60-71` |

## 10) 相关文档

- Exec 与 Shell：`02-exec-and-shell.md`
- 沙箱机制：`03-sandboxing.md`
- Agent 工具分发：`../03-agent-core/02-tool-dispatch-and-execution.md`

---

> 读完这篇，你应该能追踪一个 UnifiedExec 进程从 `allocate_process_id` → `exec_command` → HeadTailBuffer 缓冲 → 流式推送 delta events → 退出监控 → `terminate_all_processes` 的路。
