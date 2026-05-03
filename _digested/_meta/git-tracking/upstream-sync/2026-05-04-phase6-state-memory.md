---
title: "Phase 6 Upstream Sync — 07-state-memory-and-sessions"
date: "2026-05-04"
status: "completed"
scope: "Phase 6 of progressive production plan: state management & persistence, memory system"
---

# Phase 6 Sync Log — 07-state-memory-and-sessions

## 对齐锚点

- **baseline_ref**: `origin/main`
- **baseline_commit**: `35aaa5d9f` — Bound websocket request sends with idle timeout (#20751)
- **ethan_commit**: 本 Phase 开始前 `dd22e8ae0`（Phase 5）

## 已复核源码路径

### 状态持久化
- `state/src/runtime.rs:86` — `StateRuntime` 结构体
- `state/src/runtime.rs:101` — `StateRuntime::init()` SQLite 双库初始化
- `state/src/runtime.rs:230-311` — 旧版 DB 文件清理
- `state/src/runtime.rs:83-84` — 日志保留策略（10 MiB / 1000 rows per partition, 10 day retention）
- `state/src/model/thread_metadata.rs:59` — `ThreadMetadata` 完整字段
- `state/src/model/thread_metadata.rs:108` — `ThreadMetadataBuilder`
- `state/src/model/thread_goal.rs:52` — `ThreadGoal`（5 种状态、token_budget、用量核算）
- `state/src/model/agent_job.rs:74` — `AgentJob` 批量 agent 任务
- `state/src/model/log.rs:4` — `LogEntry`/`LogRow`
- `state/src/model/memories.rs:12` — `Stage1Output`/`Stage1JobClaim`
- `state/src/model/backfill_state.rs:8` — `BackfillState`
- `state/src/model/graph.rs` — `ThreadSpawnEdge` 父子边
- `state/src/runtime/threads.rs:7-927` — 完整 Thread CRUD（get/list/upsert/insert/delete/archive/touch）
- `state/src/runtime/goals.rs:24-406` — ThreadGoal CRUD + 用量核算（BudgetLimited 自动转换）
- `state/src/runtime/logs.rs:11-427` — 日志 batch insert + 查询 + feedback log 合并
- `state/src/runtime/memories.rs` — Stage1/Stage2 memory jobs
- `state/src/extract.rs:15` — `apply_rollout_item()` 从 rollout JSON → SQLite metadata
- `state/src/log_db.rs:94` — `LogDbLayer` tracing-subscriber Layer
- `state/src/migrations.rs:14` — 迁移容忍新 schema

### Thread-Store 抽象
- `thread-store/src/store.rs:21` — `ThreadStore` trait（13 方法）
- `thread-store/src/types.rs:31-256` — 全部参数/响应类型（CreateThreadParams、StoredThread、ListThreadsParams、ThreadMetadataPatch 等）
- `thread-store/src/live_thread.rs:26` — `LiveThread` handle
- `thread-store/src/live_thread.rs:36` — `LiveThreadInitGuard`
- `thread-store/src/local/mod.rs` — `LocalThreadStore` 结构
- `thread-store/src/local/live_writer.rs` — RolloutRecorder 管理
- `thread-store/src/local/create_thread.rs` — 新 thread rollout 创建
- `thread-store/src/local/read_thread.rs:29` — 双路径读取（SQLite → rollout）
- `thread-store/src/local/list_thread.rs` — 列表（rollout 扫描 + SQLite 模式）
- `thread-store/src/local/archive_thread.rs` — 归档
- `thread-store/src/local/unarchive_thread.rs` — 取消归档
- `thread-store/src/local/update_thread_metadata.rs` — 元数据 patch
- `thread-store/src/in_memory.rs` — InMemoryThreadStore
- `thread-store/src/remote/mod.rs` — RemoteThreadStore (gRPC)

### Agent Graph Store
- `agent-graph-store/src/store.rs:12` — `AgentGraphStore` trait
- `agent-graph-store/src/local.rs` — `LocalAgentGraphStore`（递归 CTE 后代查询）

### 记忆系统
- `memories/read/src/prompts.rs:28` — `build_memory_tool_developer_instructions()`
- `memories/read/src/prompts.rs:16` — `MEMORY_TOOL_DEVELOPER_INSTRUCTIONS_SUMMARY_TOKEN_LIMIT` (5000)
- `memories/read/templates/memories/read_path.md` — Read path 模板
- `memories/read/src/citations.rs:6` — `parse_memory_citation()`
- `memories/read/src/usage.rs:28` — `memories_usage_kinds_from_command()`
- `memories/write/src/start.rs:22` — `start_memories_startup_task()`
- `memories/write/src/phase1.rs` — Phase 1 实现（claim → 过滤 → 提取 → store）
- `memories/write/src/phase1.rs:53` — `StageOneOutput` 模型输出 schema
- `memories/write/src/phase2.rs` — Phase 2 实现（lock → load → sync → diff → agent → baseline）
- `memories/write/src/workspace.rs` — Git baseline workspace 管理
- `memories/write/src/storage.rs` — Artifact 文件 sync
- `memories/write/src/extensions/ad_hoc.rs` — Extension seed
- `memories/write/src/extensions/prune.rs` — 7 天过期清理
- `memories/write/src/guard.rs` — 速率限制守卫
- `memories/write/src/prompts.rs` — Phase 1/2 prompt 装配
- `memories/write/src/runtime.rs` — 模型 client 和 agent spawn
- `memories/write/src/lib.rs:78` — Phase 1/2 模型默认值
- `memories/write/templates/memories/stage_one_system.md` — Phase 1 系统 prompt
- `memories/write/templates/memories/stage_one_input.md` — Phase 1 用户 prompt
- `memories/write/templates/memories/consolidation.md` — Phase 2 整合 prompt
- `memories/write/templates/extensions/ad_hoc/instructions.md` — Extension 指令

### Core 集成
- `core/src/session/mod.rs:2589` — memory developer instruction 注入点
- `core/src/stream_events_utils.rs:74` — `strip_hidden_assistant_markup_and_parse_memory_citation()`
- `core/src/stream_events_utils.rs:176` — `record_stage1_output_usage_and_detect_memory_citation()`
- `core/src/stream_events_utils.rs:158` — pollution 检测
- `core/src/session/handlers.rs:781` — `Op::SetThreadMemoryMode`
- `core/src/session/session.rs:403` — thread memory mode 初始设置
- `core/src/mcp_tool_call.rs:613` — MCP 工具调用触发 pollution
- `core/src/state/service.rs:66` — `SessionServices` 中的 StateDbHandle/LiveThread/ThreadStore

### Protocol & Config
- `protocol/src/memory_citation.rs:1` — `MemoryCitation` 类型
- `protocol/src/items.rs:102` — `AgentMessageItem.memory_citation`
- `protocol/src/protocol.rs:811` — `ThreadMemoryMode` 枚举
- `config/src/types.rs:236` — `MemoriesToml` (TOML)
- `config/src/types.rs:267` — `MemoriesConfig` (effective)
- `features/src/lib.rs:135` — `Feature::MemoryTool`

## 产出文档

1. `07-state-memory-and-sessions/01-state-management-and-persistence.md`
2. `07-state-memory-and-sessions/02-memory-system.md`

## 复核统计

- 源码文件：70+ paths across 12 crates
- 涉及 crate：state, thread-store, agent-graph-store, memories/read, memories/write, core, protocol, config, features
