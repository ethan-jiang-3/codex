---
title: "Phase 2 Upstream Sync — 03-agent-core"
date: "2026-05-03"
status: "completed"
scope: "Phase 2 of progressive production plan: agent loop, tool dispatch, identity, context&compression"
---

# Phase 2 Sync Log — 03-agent-core

## 对齐锚点

- **baseline_ref**: `origin/main`
- **baseline_commit**: `35aaa5d9f` — Bound websocket request sends with idle timeout (#20751)
- **ethan_commit**: 本 Phase 开始前 `ad15f32bc`（Phase 1）

## 已复核源码路径

### Agent 循环与生命周期
- `core/src/session/mod.rs` — `Codex` struct (line 367), `Codex::spawn()` (423), `spawn_internal()` (447), `submit()` (673), `shutdown_and_wait()` (716), `next_event()` (727), channel 创建 (472-473), `build_initial_context()` (2531)
- `core/src/session/session.rs` — `Session` struct (12), `Session::new()` (330), `SessionConfiguration` (37)
- `core/src/session/handlers.rs` — `submission_loop()` (962), `user_input_or_turn()` (112), `shutdown()` 处理 (872-926)
- `core/src/session/turn.rs` — `run_turn()` (137), `build_prompt()` (944), `create_router()` (~1242)
- `core/src/session/turn_context.rs` — `TurnContext` 不可变快照
- `core/src/codex_thread.rs` — `CodexThread` (94), `shutdown_and_wait()` (127)
- `core/src/thread_manager.rs` — `ThreadManager` (210), `start_thread()` (534), `remove_thread()` (692), `shutdown_all_threads_bounded()` (699), `finalize_thread_spawn()` (1158)
- `core/src/tasks/mod.rs` — `SessionTask` trait (183), `spawn_task()` (292), `abort_all_tasks()` (473), `maybe_start_turn_for_pending_work()` (438)
- `core/src/tasks/regular.rs` — `RegularTask` (19), `RegularTask::run()` (40)
- `core/src/state/service.rs` — `SessionServices` DI 容器 (37)
- `core/src/agent/control.rs` — `shutdown_live_agent()` (702), `close_agent()` (722), `shutdown_agent_tree()` (736)
- `core/src/agent/registry.rs` — `release_spawned_thread()` (99)
- `core/src/agent/mailbox.rs` — agent 间通信

### 工具分发
- `core/src/tools/mod.rs` — 模块结构
- `core/src/tools/router.rs` — `ToolRouter::from_config()` (56), `build_tool_call()` (176), `dispatch_tool_call_with_code_mode_result()` (270), `model_visible_specs()` (110), `filter_deferred_dynamic_tool_spec()` (300)
- `core/src/tools/spec.rs` — `build_specs_with_discoverable_tools()` (71)
- `core/src/tools/registry.rs` — `dispatch_any()` (265), `ToolHandler` trait (88), `ToolRegistryBuilder` (530)
- `core/src/tools/orchestrator.rs` — `ToolOrchestrator::run()` (126), approval-sandbox-retry loop
- `core/src/tools/parallel.rs` — `ToolCallRuntime::handle_tool_call_with_source()` (83)
- `core/src/tools/context.rs` — `ToolInvocation` (48), `ToolPayload`, `ToolCallSource`
- `core/src/tools/sandboxing.rs` — `ExecApprovalRequirement`, `SandboxAttempt`
- `core/src/stream_events_utils.rs` — `handle_output_item_done()` (220)
- `core/src/tools/handlers/apply_patch.rs` — `intercept_apply_patch()` (467)
- `core/src/tools/code_mode/` — code mode 工具包装
- `tools/src/tool_registry_plan.rs` — `build_tool_registry_plan()` (72)
- `tools/src/tool_config.rs` — `ToolsConfig` 全部特性标志
- `code-mode/src/description.rs` — `is_code_mode_nested_tool()` (248)

### Agent Identity & System Prompt
- `agent-identity/src/lib.rs` — `AgentIdentityJwtClaims` (66), `decode_agent_identity_jwt()` (147), `generate_agent_key_material()` (262), `build_abom()` (334), `authorization_header_for_agent_task()` (106)
- `core/src/agents_md.rs` — `AgentsMdManager` (47), `user_instructions()` (82), `agents_md_paths()` (213)
- `core/src/context/fragment.rs` — `ContextualUserFragment` trait (40), `FragmentRegistration` (9)
- `core/src/context/` — ~25 fragment 实现模块（permissions, skills, plugins, environment, etc.）
- `core/src/context_manager/history.rs` — `ContextManager` (34), `reference_context_item` (50), `drop_last_n_user_turns()` (237)
- `core/src/context_manager/updates.rs` — `build_settings_update_items()` (204), `build_developer_update_item()` (178)
- `core/src/client.rs` — `prompt_cache_key` (880), `window_generation` (349), `advance_window_generation()` (356)
- `protocol/src/prompts/base_instructions/default.md` — 282 行 base instructions 模板
- `protocol/src/openai_models.rs` — `ModelInfo::get_model_instructions()` (329), `auto_compact_token_limit()` (310)

### Context & Compression
- `core/src/context_manager/history.rs` — `estimate_token_count()` (135), token estimation heuristics
- `core/src/context_manager/normalize.rs` — 三条规范化 pass
- `core/src/compact.rs` — `run_compact_task_inner_impl()` (151), `build_compacted_history()` (441), `COMPACT_USER_MESSAGE_MAX_TOKENS` (44), compaction retry loop (196-238)
- `core/src/compact_remote.rs` — remote compaction path, `trim_function_call_history_to_fit_context_window()` (327)
- `core/src/session/turn.rs` — pre-turn compact (155, 727), mid-turn compact (467-501), model-downshift compact (747)
- `protocol/src/error.rs` — `ContextWindowExceeded` (82), `is_retryable` (185)
- `core/src/client.rs` — `WebsocketSession` (241), `set_window_generation()`

## 确认并写回的事实

- Agent 实例 = `Codex`(门面) + `Session`(状态机) + 3 个 tokio channel + `submission_loop`(后台任务)
- 三种 teardown 语义有明确差异：`Op::Shutdown` 不清理 ThreadManager 记录；`shutdown_live_agent` 清理但不持久化 Closed 边；`close_agent` 全部清理 + 持久化 + 递归子树
- 没有字面常量 `AGENT_LOOP_TOOLS`；工具拦截通过统一 dispatch 管线 + code_mode 包装 + apply_patch 拦截 + unavailable tool 占位实现
- 25 个工具可用性 gate 覆盖配置级、运行时、审批级三个层面
- 没有 "SOUL.md" 概念；agent identity 是 JWT + Ed25519 密码学身份
- System prompt 通过 20+ `ContextualUserFragment` 实现，首次 turn 完整注入，稳态 turn 只做 diff
- `reference_context_item` 就是"冻结快照"：第一个 turn 建立，后续 diff，rollback/compaction 时清除
- Compaction 触发条件 = token 使用量 ≥ 90% context window；Memento 策略 = 模型自己总结 → 替换历史
- Compaction 失败回退三层：删最旧 item 重试 → 指数退避重试 → 最终失败 turn 退出但不丢 session
- Prompt caching key = `conversation_id`（整个 conversation 不变），compaction 通过 `advance_window_generation()` 使缓存失效

## 已更新文档

- `_digested/03-agent-core/01-agent-loop-and-lifecycle.md` — 新建
- `_digested/03-agent-core/02-tool-dispatch-and-execution.md` — 新建
- `_digested/03-agent-core/03-agent-identity-and-system-prompt.md` — 新建
- `_digested/03-agent-core/04-context-and-compression.md` — 新建
- `_digested/03-agent-core/README.md` — 更新文档状态

## 未覆盖风险 / 下一步

- [ ] 多 agent（multi-agent / collab）的详细机制 — 留给 Phase 5 的 skill/plugin 文档
- [ ] Guardian review 的内部 prompt 和 LLM 选择 — 当前只覆盖了它在 dispatch 链中的位置
- [ ] Goal runtime 的状态追踪细节 — 低优先级
- [ ] Remote compaction 的 protocol 细节 — 当前只说明了与 inline 的差异
- [ ] Phase 3：04-tools-execution-and-sandboxing（5 篇文档）— 下一个切片
