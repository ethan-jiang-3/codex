---
title: "Phase 3 Upstream Sync — 04-tools-execution-and-sandboxing"
date: "2026-05-03"
status: "completed"
scope: "Phase 3 of progressive production plan: tool registry, exec & shell, sandboxing, file operations, background processes"
---

# Phase 3 Sync Log — 04-tools-execution-and-sandboxing

## 对齐锚点

- **baseline_ref**: `origin/main`
- **baseline_commit**: `35aaa5d9f` — Bound websocket request sends with idle timeout (#20751)
- **ethan_commit**: 本 Phase 开始前 `94e630790`（Phase 2）

## 已复核源码路径

### 工具注册与发现
- `tools/src/tool_registry_plan.rs:72` — `build_tool_registry_plan()` 20 步构建
- `tools/src/tool_config.rs:86` — `ToolsConfig` 全部字段定义
- `tools/src/tool_config.rs:169-188` — `shell_type` 解析
- `tools/src/tool_spec.rs:22` — `ToolSpec` tagged enum 定义
- `tools/src/tool_registry_plan_types.rs:12-52` — `ToolHandlerKind` + `ToolRegistryPlan`
- `tools/src/code_mode.rs:13` — `augment_tool_spec_for_code_mode()`
- `tools/src/tool_discovery.rs:67` — `DiscoverableTool` + `tool_search` 机制
- `core/src/tools/router.rs:300` — `filter_deferred_dynamic_tool_spec()`

### 命令执行与 Shell
- `core/src/tools/handlers/shell.rs:231` — `ShellHandler::handle()`
- `core/src/tools/handlers/unified_exec.rs:179` — `UnifiedExecHandler::handle()`
- `core/src/shell_detect.rs` — `detect_shell_type()` 检测逻辑
- `core/src/shell_snapshot.rs:108` — `try_new()` snapshot 捕获
- `core/src/tools/runtimes/mod.rs:89` — `maybe_wrap_shell_lc_with_snapshot()` 注入
- `shell-escalation/src/unix/escalate_server.rs:127` — `EscalateServer` 架构
- `shell-escalation/src/unix/escalation_policy.rs:7` — `EscalationPolicy` trait
- `exec-server/src/protocol.rs` — JSON-RPC 2.0 进程管理 + 文件系统方法
- `core/src/spawn.rs` — `spawn_child_async()`
- `process-hardening/src/lib.rs:13` — `pre_main_hardening()` 平台加固
- `exec/src/lib.rs:221` — `run_main()` exec CLI 入口

### 沙箱隔离
- `sandboxing/src/manager.rs:22` — `SandboxType` 枚举
- `sandboxing/src/manager.rs:139` — `select_initial()` 选择逻辑
- `sandboxing/src/manager.rs:168` — `transform()` 命令包装
- `sandboxing/src/seatbelt.rs:603` — `create_seatbelt_command_args()` macOS sbpl 策略
- `linux-sandbox/src/bwrap.rs` — bwrap filesystem 隔离
- `linux-sandbox/src/landlock.rs:169-267` — seccomp BPF + Landlock 回退
- `windows-sandbox-rs/src/token.rs` — Windows RestrictedToken
- `windows-sandbox-rs/src/allow.rs` — ACL 文件系统
- `windows-sandbox-rs/src/desktop.rs` — 私有桌面隔离
- `windows-sandbox-rs/src/wfp.rs`, `firewall.rs` — Windows 网络过滤
- `core/src/sandboxing/mod.rs:164` — `execute_env()` 核心集成
- `core/src/tools/orchestrator.rs:126` — `ToolOrchestrator::run()` escalation 重试
- `core/src/exec.rs:783` — `is_likely_sandbox_denied()` 拒绝检测

### 文件操作
- `file-system/src/lib.rs:134` — `ExecutorFileSystem` trait 8 方法
- `apply-patch/src/parser.rs:265` — `parse_one_hunk()` 状态机解析
- `apply-patch/src/invocation.rs` — `maybe_parse_apply_patch()` 调用检测
- `apply-patch/src/invocation.rs:269-313` — Tree-sitter bash heredoc 查询
- `apply-patch/src/seek_sequence.rs` — 4 遍模糊 Unicode 搜索
- `apply-patch/src/streaming_parser.rs` — 流式增量解析
- `apply-patch/src/lib.rs:261` — `apply_hunks_to_files()` 应用
- `core/src/tools/handlers/apply_patch.rs:467` — `intercept_apply_patch()`
- `core/src/tools/handlers/list_dir.rs:57` — `ListDirHandler::handle()` BFS 遍历
- `core/src/tools/handlers/shell.rs:196` — `is_mutating()` 并发控制

### 后台进程管理
- `core/src/unified_exec/process_manager.rs:369` — `exec_command()` 流程
- `core/src/unified_exec/process_manager.rs:597` — `write_stdin()` 流程
- `core/src/unified_exec/process_manager.rs:1197` — `prune_processes_if_needed()`
- `core/src/unified_exec/process_manager.rs:1244` — `terminate_all_processes()`
- `core/src/unified_exec/process_manager.rs:308` — `terminate_process_on_network_denial()`
- `core/src/unified_exec/process.rs:101` — `UnifiedExecProcess::new()`
- `core/src/unified_exec/process_state.rs` — 进程状态枚举
- `core/src/unified_exec/head_tail_buffer.rs` — HeadTailBuffer 实现
- `core/src/unified_exec/async_watcher.rs:40` — `start_streaming_output()`
- `core/src/unified_exec/async_watcher.rs:107` — `spawn_exit_watcher()`
- `core/src/unified_exec/async_watcher.rs:284` — `split_valid_utf8_prefix()`
- `core/src/unified_exec/mod.rs:60-71` — 模块常量定义

## 产出文档

1. `04-tools-execution-and-sandboxing/01-tool-registry-and-discovery.md`
2. `04-tools-execution-and-sandboxing/02-exec-and-shell.md`
3. `04-tools-execution-and-sandboxing/03-sandboxing.md`
4. `04-tools-execution-and-sandboxing/04-file-operations.md`
5. `04-tools-execution-and-sandboxing/05-background-processes.md`

## 复核统计

- 源码文件：60+ paths across 15+ crates
- 涉及 crate：tools, core, sandboxing, linux-sandbox, windows-sandbox-rs, shell-escalation, exec-server, process-hardening, exec, file-system, apply-patch
