---
title: "02 — 记忆系统（Memory System）"
doc_type: "owner"
status: "current"
branch: "ethan"
updated: "2026-05-04"
audience: "需要理解 Codex 如何从对话中提取、整合和注入长期记忆的人"
purpose: "给出两阶段记忆管线（Phase 1 单 rollout 提取 + Phase 2 全局整合）、read path 的 developer instruction 注入、citation 生命周期、thread memory mode、以及 workspace git baseline 机制"
owns: "记忆系统面：两阶段 write pipeline、read path 注入、memory artifact 文件、citation 追踪、memory mode、git baseline workspace"
update_when:
  - "记忆 artifact 文件格式或语义变化时"
  - "Phase 1/2 管线或模型默认值变化时"
  - "read path 模板或注入策略变化时"
  - "MemoriesConfig 字段变化时"
out_of_scope:
  - "状态持久化 SQLite 细节（见 01-state-management-and-persistence.md）"
  - "Claude Code 的 auto-memory 文件系统（那是 Claude Code 的功能，非 codex 仓库）"
---

# 02 — 记忆系统（Memory System）

> [!IMPORTANT]
> 这一篇解释 Codex 的记忆系统——如何从过去的对话中提取长期记忆、整合到 `~/.codex/memories/` 下的 Markdown 文件中、并在每次 turn 注入给模型。

> 读完这篇你应该能回答：`~/.codex/memories/` 下有哪 6 种 artifact 文件、Phase 1 和 Phase 2 分别做什么、read path 怎么把记忆摘要注入 system prompt、citation 怎么被解析和追溯、以及 thread 的 memory mode 生命周期。

## 1) 先记住三件事

1. **读写分离**：`codex-memories-read` 负责每个 turn 读取 `memory_summary.md` 注入 prompt。`codex-memories-write` 负责后台异步的两阶段管线（Phase 1 提取 + Phase 2 整合）。
2. **Phase 1 是单 rollout，Phase 2 是全局**：Phase 1 从每个有意义的对话里用 `gpt-5.4-mini` 提取 raw_memory + rollout_summary。Phase 2 收集 Phase 1 的产出，用 `gpt-5.4` 整合成最终的 `MEMORY.md` 和 `memory_summary.md`。
3. **记忆注入不是自动的**：需要 `Feature::MemoryTool` 启用 + `config.memories.use_memories` = true + `memory_summary.md` 非空。三个条件缺一个就跳过。

## 2) 记忆 Artifact 文件

全部存在 `~/.codex/memories/`（"memory root"）：

| 文件 | 用途 | 生产者 | 消费者 |
|------|------|--------|--------|
| `memory_summary.md` | 紧凑摘要 + 偏好 + 索引。**每个 turn 注入 developer instructions** | Phase 2 agent | Read path |
| `MEMORY.md` | 可搜索手册。按任务分组，含 rollout 引用、关键词、偏好 | Phase 2 agent | Agent 运行时 grep |
| `raw_memories.md` | Phase 1 产出的合并 raw memories，按 thread_id 升序 | Phase 2 sync code | Phase 2 agent（临时输入） |
| `rollout_summaries/*.md` | 每 rollout 一个 recap（证据、偏好信号、可复用知识、失败） | Phase 2 sync code | Phase 2 agent |
| `skills/<name>/SKILL.md` | 可复用的斜杠命令包 | Phase 2 agent | Agent |
| `extensions/ad_hoc/notes/<ts>-<slug>.md` | 用户临时笔记 | Agent（经 read path 提示） | Phase 2 agent |

### 额外路径

- `extensions/ad_hoc/instructions.md` — 启动时 seed 的扩展指令
- `extensions/<ext>/resources/*.md` — 扩展资源文件，**7 天后自动清理**
- `phase2_workspace_diff.md` — 生成的 git-style diff，在 baseline reset 前删除

## 3) Read Path：记忆注入

`memories/read/src/prompts.rs:28` — `build_memory_tool_developer_instructions()`：

1. 从 `~/.codex/memories/memory_summary.md` 读取
2. 截断到 **5,000 tokens**（`MEMORY_TOOL_DEVELOPER_INSTRUCTIONS_SUMMARY_TOKEN_LIMIT`）
3. 如果为空或无文件 → 返回 `None`
4. 渲染模板 `memories/read/templates/memories/read_path.md`，填入 `base_path` 和 `memory_summary`

**核心注入点** (`core/src/session/mod.rs:2589`)：
```rust
if turn_context.features.enabled(Feature::MemoryTool)
    && turn_context.config.memories.use_memories
    && let Some(memory_prompt) =
        build_memory_tool_developer_instructions(&turn_context.config.codex_home).await
{
    developer_sections.push(memory_prompt);
}
```

**模板内容**告诉模型：
- 记忆文件夹布局（各文件的位置和用途）
- 何时使用记忆（跳过琐碎查询，有 workspace/path 提及才查）
- "Quick memory pass" 协议：扫摘要 → 关键词搜 MEMORY.md → 必要时打开 1-2 个 rollout summary/skill
- Citation 要求：回复末尾附加 `<oai-mem-citation>` 块
- 临时笔记：只有用户显式要求才能写 `extensions/ad_hoc/notes/`

## 4) Citation 生命周期

### 模型侧

模型在回复末尾附加隐藏的 citation 块：
```xml
<oai-mem-citation>
  <citation_entries>
    MEMORY.md:234-236|note=[responsesapi citation extraction code pointer]
  </citation_entries>
  <rollout_ids>uuid-1, uuid-2</rollout_ids>
</oai-mem-citation>
```

### 解析

`memories/read/src/citations.rs:6` — `parse_memory_citation()` 提取 `<citation_entries>` 和 `<rollout_ids>` / `<thread_ids>`。

`core/src/stream_events_utils.rs:74` — `strip_hidden_assistant_markup_and_parse_memory_citation()` 从可见文本中剥离 citation 并返回解析后的 `MemoryCitation`。

### 追溯

`record_stage1_output_usage_and_detect_memory_citation()` (`stream_events_utils.rs:176`)：
- 在每次 response item 完成时检查
- 发现 citation → 调用 `db.record_stage1_output_usage()` 递增 `usage_count` 并设置 `last_usage`

### Protocol 类型

`protocol/src/memory_citation.rs:1`：
```rust
pub struct MemoryCitation {
    pub entries: Vec<MemoryCitationEntry>,
    pub rollout_ids: Vec<String>,
}
pub struct MemoryCitationEntry {
    pub path: String,
    pub line_start: u32,
    pub line_end: u32,
    pub note: String,
}
```

附加在 `AgentMessageItem.memory_citation` 字段上 (`protocol/src/items.rs:102`)。

## 5) Write Path：两阶段管线

`memories/write/src/start.rs:22` — `start_memories_startup_task()` 在 session 启动时作为后台 tokio task 触发。

**跳过条件**：ephemeral session、`Feature::MemoryTool` 未启用、子 agent session、state DB 不可用。

### Phase 1：单 Rollout 提取

`memories/write/src/phase1.rs`：

1. **Claim rollouts** (line 75)：`claim_stage1_jobs_for_startup()` 扫描 SQLite 中 `memory_mode = 'enabled'` 的活跃 thread，要求：
   - `max_rollout_age_days` 以内（默认 10 天）
   - 空闲至少 `min_rollout_idle_hours`（默认 6 小时）
   - 未被其他 job claim
   - 最多 `max_rollouts_per_startup` 个（默认 2）

2. **构建请求** (line 185)：模型默认 `gpt-5.4-mini`，`ReasoningEffort::Low`

3. **处理每个 rollout** (line 199，并发上限 8)：
   - 从 JSONL 加载 rollout items
   - 过滤：移除 developer 角色消息、AGENTS.md 片段、`<skill>` 块；保留环境上下文和子 agent 通知
   - 截断到模型有效上下文窗口的 70%
   - 使用 Phase 1 模型提取：严格 JSON schema — `raw_memory`（详细）、`rollout_summary`（紧凑）、`rollout_slug`（可选）
   - 输出 secret 脱敏
   - 结果存入 SQLite `memories_stage1` 表

**产出**：`Stage1Output { thread_id, rollout_path, source_updated_at, raw_memory, rollout_summary, rollout_slug, cwd, git_branch, generated_at }`

### Phase 2：全局整合

`memories/write/src/phase2.rs`：

1. **Claim 全局锁** (line 57)：`try_claim_global_phase2_job()` 序列化 Phase 2——整个系统同时只能有一个整合在跑

2. **准备 workspace** (line 66)：`prepare_memory_workspace()` 将 memory root 初始化为 git baseline 目录（`~/.codex/memories/.git`）

3. **锁定 agent config** (line 79)：
   - `ephemeral = true`（整合线程不会反馈到下一轮 Phase 1）
   - `generate_memories = false`、`use_memories = false`
   - `AskForApproval::Never`
   - 无 collab、plugins、apps、MCP
   - Sandbox: `WorkspaceWrite` 仅 memory root 可写，无网络访问

4. **加载 Phase 2 输入** (line 93)：选择最多 `max_raw_memories_for_consolidation`（默认 256）条 Phase 1 输出，按 `usage_count DESC` 排序，`thread_id ASC` 稳定顺序

5. **Sync workspace** (line 114)：
   - 写入 `rollout_summaries/*.md`（每个保留的 memory 一个文件）
   - 写入 `raw_memories.md`（全部 raw memory 合并）
   - 清理旧的 extension resource 文件（>7 天）

6. **检查变更** (line 127)：`memory_workspace_diff()` 用 git 计算 diff。无变更则标记成功退出

7. **Spawn 整合 agent** (line 170)：有变更时，写入 `phase2_workspace_diff.md`，启动内部 agent

8. **Monitor + heartbeat** (line 352)：轮询 agent 状态，每 90 秒续约全局 job lease

9. **完成** (line 375)：reset git baseline（先删除 diff 文件），标记 job 成功

### 模型默认值

| 阶段 | 模型 | Reasoning | 并行度 | Lease |
|------|------|-----------|--------|-------|
| Phase 1 | `gpt-5.4-mini` | Low | 8 | 3600s |
| Phase 2 | `gpt-5.4` | Medium | 1 (全局锁) | 3600s |

## 6) Thread Memory Mode 生命周期

`threads` 表的 `memory_mode` 列有三种值：

| 值 | 含义 | 设置者 |
|----|------|--------|
| `enabled` | Phase 1/2 可处理 | Session 创建时（`config.memories.generate_memories = true`） |
| `disabled` | 从不处理 | Session 创建时（`generate_memories = false`）或 Op::SetThreadMemoryMode |
| `polluted` | 涉及外部上下文，需要遗忘 | `mark_thread_memory_mode_polluted()` — 当 web search 或 MCP 工具与 `disable_on_external_context = true` 一起使用时 |

### Pollution 检测

`core/src/stream_events_utils.rs:158`：每次 response item 完成时检查是否为 `ToolSearchCall`、`ToolSearchOutput` 或 `WebSearchCall`。如果是 → 标记 `polluted`（需 `config.memories.disable_on_external_context = true`）。

MCP 工具调用同样触发 (`core/src/mcp_tool_call.rs:613`)。

## 7) Workspace Git Baseline 机制

`memories/write/src/workspace.rs`：

Memory root 作为 git 仓库管理（仅用于 diff，不用作版本控制）：

1. `prepare_memory_workspace()` — 创建目录，清理旧 diff 文件，确保 git baseline 存在
2. `memory_workspace_diff()` — 从上次 baseline 计算 diff
3. `write_workspace_diff()` — 写入 `phase2_workspace_diff.md`（最大 4 MB）
4. `reset_memory_workspace_baseline()` — Phase 2 成功后删除 diff 文件，reset git 仓库到当前状态作为新 baseline

diff 文件在所有操作前和 reset 前都会被删除，防止旧 diff 被当作记忆内容。

## 8) MemoriesConfig

`config/src/types.rs:267`：

| 字段 | 默认 | 说明 |
|------|------|------|
| `disable_on_external_context` | `false` | MCP/web search 后标记 polluted |
| `generate_memories` | `true` | 启用 Phase 1/2 write pipeline |
| `use_memories` | `true` | 注入记忆到 developer prompt |
| `max_raw_memories_for_consolidation` | `256` | Phase 2 输入上限 |
| `max_unused_days` | `30` | 超过此天数未使用的记忆被丢弃 |
| `max_rollout_age_days` | `10` | Phase 1 处理的 rollout 最大年龄 |
| `max_rollouts_per_startup` | `2` | 每次启动最多处理几个 rollout |
| `min_rollout_idle_hours` | `6` | rollout 最短空闲时间 |
| `min_rate_limit_remaining_percent` | `25` | 低于此则跳过 |
| `extract_model` | `None` | Phase 1 模型（默认 gpt-5.4-mini） |
| `consolidation_model` | `None` | Phase 2 模型（默认 gpt-5.4） |

## 9) 完整生命周期

```
启动 Session
  │
  ├── live_thread.memory_mode = Enabled/Disabled
  │
  └── start_memories_startup_task() 后台运行
        ├── Seed ad_hoc extension instructions
        ├── Prune 过期 stage1_outputs
        ├── 检查 rate limits
        ├── Phase 1: claim rollouts → 提取 → 存 SQLite
        └── Phase 2: claim 全局锁 → 加载 → sync workspace
              → diff → 如有变更: spawn agent
              → agent 编辑 MEMORY.md, memory_summary.md, skills/
              → reset baseline → 成功

每次 Turn
  │
  ├── Build developer instructions:
  │     ├── Feature::MemoryTool + use_memories + memory_summary.md 非空?
  │     └── Yes → 读 memory_summary.md → 截断到 5000 tokens → 注入 prompt
  │
  ├── Agent 使用记忆（按 read_path 指引）:
  │     ├── Quick memory pass: 扫摘要 → grep MEMORY.md → 打开 rollout summaries
  │     └── 回复末尾附加 <oai-mem-citation>
  │
  └── 处理模型输出:
        ├── Strip citation markup → 解析 MemoryCitation
        ├── Record usage: 递增 cited thread 的 usage_count
        └── 检查 pollution: web search/MCP → mark polluted
```

## 10) 源码抓手

| 想看什么 | 从哪里开始 |
|----------|-----------|
| Read path prompt builder | `memories/read/src/prompts.rs:28` |
| Read path 模板 | `memories/read/templates/memories/read_path.md` |
| Citation 解析 | `memories/read/src/citations.rs:6` |
| Citation 剥离与追踪 | `core/src/stream_events_utils.rs:74` |
| Developer instructions 注入 | `core/src/session/mod.rs:2589` |
| Write path 启动 | `memories/write/src/start.rs:22` |
| Phase 1 实现 | `memories/write/src/phase1.rs` |
| Phase 1 系统 prompt | `memories/write/templates/memories/stage_one_system.md` |
| Phase 2 实现 | `memories/write/src/phase2.rs` |
| Phase 2 整合 prompt | `memories/write/templates/memories/consolidation.md` |
| Workspace/diff 管理 | `memories/write/src/workspace.rs` |
| Artifact sync | `memories/write/src/storage.rs` |
| Extension seed/prune | `memories/write/src/extensions/ad_hoc.rs` |
| Stage1Output 类型 | `state/src/model/memories.rs:12` |
| MemoryCitation 类型 | `protocol/src/memory_citation.rs:1` |
| ThreadMemoryMode 枚举 | `protocol/src/protocol.rs:811` |
| MemoriesToml (TOML) | `config/src/types.rs:236` |
| MemoriesConfig (effective) | `config/src/types.rs:267` |
| Feature::MemoryTool | `features/src/lib.rs:135` |
| Memory mode pollution | `core/src/stream_events_utils.rs:158` |
| SetThreadMemoryMode op | `core/src/session/handlers.rs:781` |

## 11) 相关文档

- 状态管理与持久化：`01-state-management-and-persistence.md`
- Agent 循环与生命周期：`../03-agent-core/01-agent-loop-and-lifecycle.md`
- 上下文与压缩：`../03-agent-core/04-context-and-compression.md`

---

> 读完这篇，你应该能追踪一段对话从 rollout JSONL → Phase 1 提取 → SQLite `memories_stage1` → Phase 2 全局整合 → `MEMORY.md` + `memory_summary.md` → 下次 turn 的 developer instructions 注入 → 模型 citation → usage 追溯的完整记忆链路。
