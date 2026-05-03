---
title: "01 — 状态管理与持久化（State Management & Persistence）"
doc_type: "owner"
status: "current"
branch: "ethan"
updated: "2026-05-04"
audience: "需要理解 Codex 如何存、查、恢复对话状态的人"
purpose: "给出 SQLite 双库持久化、ThreadStore trait 抽象、LocalThreadStore 的 rollout JSONL + SQLite 双写、LiveThread 生命周期、以及 agent-graph-store 的父子拓扑"
owns: "状态持久化面：StateRuntime、ThreadStore trait、LocalThreadStore、LiveThread、agent-graph-store"
update_when:
  - "SQLite schema 版本号或数据库结构变化时"
  - "ThreadStore trait 新增方法时"
  - "rollout 文件格式或目录结构变化时"
  - "线程生命周期管理方式变化时"
out_of_scope:
  - "记忆系统（见 02-memory-system.md）"
  - "会话管理（见 ../03-agent-core/01-agent-loop-and-lifecycle.md）"
---

# 01 — 状态管理与持久化（State Management & Persistence）

> [!IMPORTANT]
> 这一篇解释 Codex 如何让对话"关掉重开还能继续"——从 SQLite 双库设计到 rollout JSONL 文件格式，从 `ThreadStore` trait 抽象到 `LiveThread` 生命周期。

> 读完这篇你应该能回答：thread 数据存在哪里（两个 SQLite + JSONL 文件）、`ThreadStore` trait 提供哪些操作、`LiveThread` 从创建到 shutdown 的全生命周期、以及 agent 父子关系怎么存。

## 1) 先记住三件事

1. **三层存储，两个目的**：rollout JSONL 文件（权威重放历史）+ SQLite `state_N.sqlite`（快速元数据索引）+ SQLite `logs_N.sqlite`（tracing 日志）。JSONL 是真相源，SQLite 是索引。
2. **SQLite 版本号嵌入文件名**：`state_5.sqlite`、`logs_2.sqlite`。版本号变了就新建数据库，旧文件在下次 init 时自动清理。
3. **ThreadStore 是抽象边界**：`ThreadStore` trait 解耦了 session 代码与存储后端——本地用 `LocalThreadStore`（文件 + SQLite），远端用 `RemoteThreadStore`（gRPC），测试用 `InMemoryThreadStore`。

## 2) StateRuntime：SQLite 双库

`state/src/runtime.rs:86`：

```rust
pub struct StateRuntime {
    codex_home: PathBuf,
    default_provider: String,
    pool: Arc<sqlx::SqlitePool>,           // state DB 连接池（最多 5 连接）
    logs_pool: Arc<sqlx::SqlitePool>,       // logs DB 连接池
    thread_updated_at_millis: Arc<AtomicI64>, // 单调时间戳分配器
}
```

### 两个 SQLite 数据库

| 数据库 | 文件名 | 内容 |
|--------|--------|------|
| State DB | `state_5.sqlite` | `threads`、`thread_goals`、`agent_jobs`、`thread_spawn_edges`、`thread_dynamic_tools`、`device_key_bindings`、`remote_control_enrollments`、`memories_stage1`、`backfill_state` |
| Logs DB | `logs_2.sqlite` | `logs` 表 — tracing 事件日志 |

两者均使用 WAL 日志模式、`NORMAL` synchronous、5 秒 busy timeout、增量 auto-vacuum。

**数据库位置**：`$CODEX_HOME`（可通过 `CODEX_SQLITE_HOME` 环境变量覆盖）。

### 日志保留策略

- 按 partition（thread_id 或 process UUID）限制：**10 MiB + 1,000 行**
- 按时间限制：启动时删除 **10 天前**的日志

### 单调时间戳

`allocate_thread_updated_at()` 使用 `AtomicI64` 分配进程内唯一的毫秒时间戳。热路径写入无需查询 SQLite 即可获得唯一 ms 时间戳。回填时间戳保留原值不变。同一秒内的时间戳被 +1ms 递增以避免冲突。

## 3) ThreadStore Trait：存储抽象

`thread-store/src/store.rs:21`：

```rust
#[async_trait]
pub trait ThreadStore: Any + Send + Sync {
    // 实时线程生命周期
    async fn create_thread(&self, params: CreateThreadParams) -> ThreadStoreResult<()>;
    async fn resume_thread(&self, params: ResumeThreadParams) -> ThreadStoreResult<()>;
    async fn append_items(&self, params: AppendThreadItemsParams) -> ThreadStoreResult<()>;
    async fn persist_thread(&self, thread_id: ThreadId) -> ThreadStoreResult<()>;
    async fn flush_thread(&self, thread_id: ThreadId) -> ThreadStoreResult<()>;
    async fn shutdown_thread(&self, thread_id: ThreadId) -> ThreadStoreResult<()>;
    async fn discard_thread(&self, thread_id: ThreadId) -> ThreadStoreResult<()>;

    // 读操作
    async fn load_history(&self, params: LoadThreadHistoryParams) -> ThreadStoreResult<StoredThreadHistory>;
    async fn read_thread(&self, params: ReadThreadParams) -> ThreadStoreResult<StoredThread>;
    async fn list_threads(&self, params: ListThreadsParams) -> ThreadStoreResult<ThreadPage>;

    // 变更操作
    async fn update_thread_metadata(&self, params: UpdateThreadMetadataParams) -> ThreadStoreResult<StoredThread>;
    async fn archive_thread(&self, params: ArchiveThreadParams) -> ThreadStoreResult<()>;
    async fn unarchive_thread(&self, params: ArchiveThreadParams) -> ThreadStoreResult<StoredThread>;
}
```

### 关键参数和返回类型

**CreateThreadParams** (`types.rs:43`)：`thread_id`、`forked_from_id`、`source: SessionSource`、`base_instructions`、`dynamic_tools`、`metadata: ThreadPersistenceMetadata`、`event_persistence_mode`

**StoredThread** (`types.rs:184`)：统一的读/列表响应。包含所有 thread 元数据 + 可选的 `history: Option<StoredThreadHistory>`

**ListThreadsParams** (`types.rs:148`)：`page_size`、`cursor`（不透明游标）、`sort_key`、`sort_direction`、`allowed_sources`、`model_providers`、`cwd_filters`、`archived`、`search_term`（使用 SQLite `instr()` 子串匹配）、`use_state_db_only`

**ThreadMetadataPatch** (`types.rs:249`)：`name: Option<String>`、`memory_mode: Option<MemoryMode>`、`git_info: Option<GitInfoPatch>`

## 4) LocalThreadStore：文件系统 + SQLite

`thread-store/src/local/mod.rs`：

```rust
pub struct LocalThreadStore {
    config: LocalThreadStoreConfig,
    live_recorders: Arc<Mutex<HashMap<ThreadId, RolloutRecorder>>>,
    state_db: Arc<OnceCell<StateDbHandle>>,  // 惰性初始化的 StateRuntime
}
```

### Rollout JSONL 文件

每个 thread 对应一个 JSONL 文件：

- **活跃**：`{codex_home}/sessions/{YYYY}/{MM}/{DD}/rollout-{TIMESTAMP}-{UUID}.jsonl`
- **已归档**：`{codex_home}/archived_sessions/rollout-{TIMESTAMP}-{UUID}.jsonl`

每行是一个 JSON 对象，包含 `type`、`timestamp`、`payload` 字段。关键行类型：`session_meta`、`turn_context`、`event_msg`（子类型：`user_message`、`token_count`、`thread_name_updated`）、`response_item`、`compacted`。

### SQLite threads 表

维护与 JSONL 镜像的元数据，用于快速查询（按 source、model_provider、cwd、search_term 过滤和排序）：

- `ThreadMetadata` (`state/src/model/thread_metadata.rs:59`)：id、rollout_path、created_at、updated_at、source、agent_nickname、model_provider、model、reasoning_effort、cwd、cli_version、sandbox_policy、approval_mode、title、first_user_message、tokens_used、archived_at、git_sha、git_branch、git_origin_url

### 状态提取

`state/src/extract.rs` — `apply_rollout_item()` 将 rollout 中的每行 JSON 转换为 SQLite 元数据更新：
- `SessionMeta` → 设置 id、source、agent info、model_provider、cwd、git info
- `TurnContext` → 设置 cwd、model、reasoning_effort、sandbox_policy、approval_mode
- `EventMsg::TokenCount` → 更新 `tokens_used`
- `EventMsg::UserMessage` → 设置 `first_user_message` 和 `title`
- `EventMsg::ThreadNameUpdated` → 覆盖 `title`

### 列表与搜索

`list_threads()` 支持两种模式：
- **Rollout 扫描** (`use_state_db_only: false`)：扫描 `sessions/` 目录树找 JSONL 文件
- **SQLite 模式** (`use_state_db_only: true`)：直接查询 `threads` 表，支持 source、model_provider、cwd 过滤和 title 文本搜索

分页通过不透明 cursor 实现（序列化的 sort key + direction + anchor timestamp）。

## 5) LiveThread：线程生命周期

`thread-store/src/live_thread.rs:26`：

```rust
pub struct LiveThread {
    store: Arc<dyn ThreadStore>,
    thread_id: ThreadId,
}
```

### 生命周期流程

```
Create:
  LiveThread::create(store, params)
    → store.create_thread() → 创建 RolloutRecorder → 开始写 JSONL
    → LiveThreadInitGuard 包装（失败时自动 discard）

运行中:
  append_items(items)  → 缓冲 rollout items
  persist()            → 物化延迟写入
  flush()              → fsync 确保持久化

关闭:
  shutdown()           → 最终 flush + 关闭 writer
  discard()            → 丢弃 writer（失败初始化时用）
```

**LiveThreadInitGuard** (line 36)：如果 session 初始化失败，drop 时自动调用 `discard()` 防止写入不完整的 rollout。调用 `commit()` 将所有权转移给调用方。

### 会话恢复

重新打开已有 thread：
1. `ThreadStore::read_thread()` 先从 SQLite（`StateRuntime::get_thread()`）查找
2. 如果 rollout path 有效，读取 JSONL 获取完整历史
3. `ThreadStore::resume_thread()` 在已有文件上打开新的 `RolloutRecorder`（追加模式）
4. 新的 items 追加到已有 rollout

## 6) 归档与取消归档

**归档**：`fs::rename()` 将 rollout 从 `sessions/YYYY/MM/DD/` 移到 `archived_sessions/`，SQLite 写入 `archived_at`

**取消归档**：`fs::rename()` 移回 `sessions/YYYY/MM/DD/`，更新文件 mtime，SQLite 清除 `archived_at`

## 7) AgentGraphStore：父子拓扑

`agent-graph-store/src/store.rs:12`：

```rust
#[async_trait]
pub trait AgentGraphStore: Send + Sync {
    async fn upsert_thread_spawn_edge(parent, child, status) -> Result<()>;
    async fn set_thread_spawn_edge_status(child, status) -> Result<()>;
    async fn list_thread_spawn_children(parent, status_filter) -> Result<Vec<ThreadId>>;
    async fn list_thread_spawn_descendants(root, status_filter) -> Result<Vec<ThreadId>>;
}
```

`LocalAgentGraphStore` 包装 `Arc<StateRuntime>`，直接操作 `thread_spawn_edges` 表（`parent_thread_id`、`child_thread_id`、`status`）。

**后代查询**使用 SQLite 递归 CTE，广度优先：
```sql
WITH RECURSIVE subtree(child_thread_id, depth) AS (
    SELECT child_thread_id, 1 FROM thread_spawn_edges WHERE parent_thread_id = ?
    UNION ALL
    SELECT edge.child_thread_id, subtree.depth + 1
    FROM thread_spawn_edges AS edge
    JOIN subtree ON edge.parent_thread_id = subtree.child_thread_id
)
SELECT child_thread_id FROM subtree ORDER BY depth ASC, child_thread_id ASC
```

## 8) Log 数据库 Tracer

`state/src/log_db.rs:94` — `LogDbLayer` 实现 `tracing_subscriber::Layer`：
- 从 tracing span 捕获事件
- 从 span 字段提取 `thread_id`
- 格式化 `feedback_log_body`（span context + event fields）
- 发送到 bounded `mpsc` channel（容量 512）
- 后台 task 批量插入（128 per batch）+ 2 秒 flush 间隔

## 9) ThreadGoal：目标与用量追踪

`state/src/model/thread_goal.rs:52`：

```rust
pub struct ThreadGoal {
    pub thread_id: ThreadId,
    pub goal_id: GoalId,
    pub objective: String,
    pub status: ThreadGoalStatus,  // Active | Paused | BudgetLimited | Complete
    pub token_budget: i64,
    pub tokens_used: i64,
    pub time_used_seconds: f64,
}
```

用量核算 `account_thread_goal_usage()`：累加 `tokens_used` 和 `time_used_seconds`。当 `tokens_used >= token_budget` 时自动转换状态为 `BudgetLimited`。

## 10) 源码抓手

| 想看什么 | 从哪里开始 |
|----------|-----------|
| StateRuntime | `state/src/runtime.rs:86` |
| StateRuntime::init | `state/src/runtime.rs:101` |
| SQLite 双库初始化 | `state/src/runtime.rs:230-311` |
| ThreadStore trait | `thread-store/src/store.rs:21` |
| CreateThreadParams | `thread-store/src/types.rs:43` |
| StoredThread | `thread-store/src/types.rs:184` |
| ListThreadsParams | `thread-store/src/types.rs:148` |
| LocalThreadStore | `thread-store/src/local/mod.rs` |
| LiveThread | `thread-store/src/live_thread.rs:26` |
| LiveThreadInitGuard | `thread-store/src/live_thread.rs:36` |
| RolloutRecorder | `thread-store/src/local/live_writer.rs` |
| Thread 创建 | `thread-store/src/local/create_thread.rs` |
| Thread 恢复 | `thread-store/src/local/live_writer.rs` |
| Thread 读取 | `thread-store/src/local/read_thread.rs` |
| Thread 列表 | `thread-store/src/local/list_thread.rs` |
| 归档/取消归档 | `thread-store/src/local/archive_thread.rs` |
| ThreadMetadata | `state/src/model/thread_metadata.rs:59` |
| 状态提取 (apply_rollout_item) | `state/src/extract.rs:15` |
| AgentGraphStore trait | `agent-graph-store/src/store.rs:12` |
| 递归 CTE 后代查询 | `agent-graph-store/src/local.rs` |
| Thread CRUD | `state/src/runtime/threads.rs` |
| ThreadGoal | `state/src/model/thread_goal.rs:52` |
| ThreadGoal 用量核算 | `state/src/runtime/goals.rs:316` |
| LogDbLayer（tracing 捕获） | `state/src/log_db.rs:94` |
| 日志保留策略 | `state/src/runtime.rs:83-84` |

## 11) 相关文档

- 记忆系统：`02-memory-system.md`
- Agent 循环与生命周期：`../03-agent-core/01-agent-loop-and-lifecycle.md`
- 上下文与压缩：`../03-agent-core/04-context-and-compression.md`

---

> 读完这篇，你应该能追踪一个 thread 从 `create_thread` → JSONL rollout 写入 → SQLite 元数据镜像 → `shutdown` → 下次 `resume_thread` 恢复追加的完整持久化链路。
