---
title: "04 — 上下文管理与压缩（Context & Compression）"
doc_type: "owner"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解 Codex 如何管理 context window、何时触发压缩、压缩策略和失败回退的人"
purpose: "给出 ContextManager 的结构、compaction 触发条件与策略、prompt caching 交互、以及 context window 超限的处理路径"
owns: "上下文管理：ContextManager 结构、compaction 触发/策略/失败回退、与 prompt caching 的交互"
update_when:
  - "compaction 策略或触发条件变化时"
  - "context window 管理方式变化时"
  - "新增 compaction 模式时"
out_of_scope:
  - "system prompt 装配（见 03-agent-identity-and-system-prompt.md）"
  - "具体 context fragment 内容"
  - "模型 API 层面的 token 计数细节"
---

# 04 — 上下文管理与压缩（Context & Compression）

> [!IMPORTANT]
> 这一篇解释 Codex 怎么在 context window 有限的情况下管理越来越长的对话历史。

> 读完这篇你应该能回答：什么时候触发压缩、怎么压缩（策略叫 Memento）、压缩失败了怎么办、prompt caching 和压缩怎么交互。

## 1) 先记住三件事

1. **Context 由 ContextManager 管理**，是一个内存中的 `Vec<ResponseItem>` 列表（最老在前），加上 `history_version`（单调递增版本号）和 `reference_context_item`（diff 基线）。
2. **Compaction 触发条件是 token 阈值**：当前使用量 ≥ 模型 context window 的 90%。在 pre-turn 和 mid-turn 两个位置各检查一次。
3. **Compaction 策略叫 Memento**：让模型自己总结进度、决策、上下文和下一步，用总结替换历史中的工具调用和中间消息。

## 2) ContextManager 结构

```
ContextManager (context_manager/history.rs:34)
  ├── items: Vec<ResponseItem>           // 最老→最新
  ├── history_version: u64               // 每次 compaction/rollback 递增
  ├── token_info: Option<TokenUsageInfo> // 上次 API 响应的 token 统计
  └── reference_context_item: Option<TurnContextItem>  // diff 基线
```

**`history_version`** 的语义：单调递增计数器，当 history 被 compaction / rollback / `replace` / `remove_last_item` / `replace_last_turn_images` 改写时递增。外部通过 `history_version()` 检测 history 是否被外部修改。

**`reference_context_item`** 的语义：第一个真实 turn 建立的"冻结"上下文快照。后续 turn 只做 diff（`build_settings_update_items`）。当 rollback 切掉基线所在的 turn 时被清除 → 下一轮完整注入。

## 3) Compaction 触发条件

### 两个检查点

| 检查点 | 位置 | 时机 |
|--------|------|------|
| Pre-turn | `session/turn.rs:727` | `run_turn()` 开始后、任何 context 更新前 |
| Mid-turn | `session/turn.rs:467-468` | 每次 model 采样返回后，`needs_follow_up = true` 时 |

### Token 阈值计算

`ModelInfo::auto_compact_token_limit()`（`protocol/src/openai_models.rs:310`）：

```
auto_compact_limit = min(
    config.model_auto_compact_token_limit,  // 用户可配
    context_window * 9 / 10                  // 默认：90%
)
```

**额外触发**：模型降级（切换到更小 context window 的模型）
- `maybe_run_previous_model_inline_compact()`（`turn.rs:747`）
- 当 `total_usage_tokens > new_auto_compact_limit && old_context_window > new_context_window`
- 使用**旧模型**执行 compaction（因为新模型窗口装不下当前历史）

### Manual 触发

用户输入 `/compact` → `CompactionTrigger::Manual`。

## 4) Compaction 策略：Memento

**策略名称**：`CompactionStrategy::Memento`（`compact.rs:329`）

### 过程（inline 路径）

```
输入：当前完整 history

步骤：
1. 克隆全部 history 项
2. 记录 compaction 输入 item
3. 规范化（去掉不支持模态的内容，去掉图片）
4. 组装 Prompt：
   - system: base_instructions + compaction instructions
   - user: "summarize progress, decisions, context, next steps"
5. 发送给模型（标准 Responses API 流式请求）
6. 收集模型助理回复 → 成为"总结后缀"
7. 构建压缩后 history：
   a. 从原始 history 收集 user messages（过滤掉之前的总结消息）
   b. 选最近的 user messages，最高 20,000 tokens（COMPACT_USER_MESSAGE_MAX_TOKENS）
   c. 最新优先，最老的截断
   d. 每个 user msg + SUMMARY_PREFIX + 模型总结
8. 注入初始 context（mid-turn: BeforeLastUserMessage, pre-turn: DoNotInject）
9. 替换 ContextManager.history_items
10. history_version += 1
11. advance_window_generation() → prompt cache bust
```

### `InitialContextInjection` 两种模式

| 模式 | 使用场景 | 含义 |
|------|---------|------|
| `BeforeLastUserMessage` | Mid-turn compaction | 模型训练预期总结作为最后一条 item；context 注入到最后一个 user msg 之前 |
| `DoNotInject` | Pre-turn / Manual compaction | 清除 `reference_context_item`，由下一轮正常 turn 完整注入 |

## 5) Compaction 失败与回退

Compaction 失败有三条路径，在重试循环中处理（`compact.rs:196-238`）：

| 错误 | 处理 | 重试？ |
|------|------|--------|
| `ContextWindowExceeded` | 删除最旧的一条 item（保留 prefix cache），然后重试 | 是（直到只剩 1 条 item） |
| `Interrupted` | 立即传播 | 否 |
| 其他错误 | 指数退避重试 `stream_max_retries()` 次 | 是（有限次） |

**如果最终失败**：
- Pre-turn：`run_turn()` 返回 `None`，turn 不开始
- Mid-turn：`run_turn()` 返回 `None`，turn 正常结束（用户可以继续下一轮）
- 前端的 `TokenCount` 事件会反映 token 使用量已达上限

## 6) Remote vs Inline Compaction

| | Inline | Remote |
|------|--------|--------|
| **路径** | 本地模型流式请求 | `POST /responses/compact` |
| **触发条件** | 默认 | `provider.supports_remote_compaction()` 返回 true |
| **预处理** | 无 | `trim_function_call_history_to_fit_context_window()` — 先去掉旧 codex 生成 item |
| **输出过滤** | 无 | 去掉 developer 消息、去掉非用户内容的 user 消息、保留 assistant 消息和用户消息 |
| **失败诊断** | 标准 error | 额外记录 `all_history_items_model_visible_bytes`、`estimated_tokens` 等详细诊断 |

**路由逻辑**（`turn.rs:788`）：
```rust
if should_use_remote_compact_task(provider.info()) {
    remote compact
} else {
    inline compact
}
```

## 7) Context Window 超限处理

`CodexErr::ContextWindowExceeded`（`protocol/src/error.rs:82`）是**不可重试**的错误（`is_retryable` 返回 false，line 185）：

**在模型采样中**（`turn.rs:1042`）：
- 调用 `set_total_tokens_full()` 标记 token 用量为 100%
- 错误向上传播，turn 结束

**在 compaction 中**（`compact.rs:204`）：
- 不是报错结束，而是**删除最旧 item → 重试**
- 这是"last-ditch fallback"：宁可丢掉早期 history 也要让 compaction 成功
- 删除从**最旧的 item**开始（prefix 方向），保留剩余 item 的 prompt cache

## 8) Prompt Caching 与 Compaction 的交互

**Cache key**：`prompt_cache_key = conversation_id`（整个 conversation 不变）

**Compaction 导致 cache bust**：
- `advance_window_generation()` 递增计数器（`client.rs:356`）
- `x-codex-window-id` header 从 `conversation_id:3` → `conversation_id:4`
- 模型侧视为新 cache key → 重新缓存

**Compaction 重试中的 cache 保留**：
- 删除 item 从**最老**开始（prefix-based）
- 保留剩余 item 的 prefix 结构 → 保留对应部分的 prompt cache

## 9) 源码抓手

| 想看什么 | 从哪里开始 |
|----------|-----------|
| ContextManager 定义 | `core/src/context_manager/history.rs:34` |
| Token 估算 | `core/src/context_manager/history.rs:135` — `estimate_token_count()` |
| History 规范化 | `core/src/context_manager/normalize.rs` |
| Context diff 更新 | `core/src/context_manager/updates.rs:204` |
| Rollback | `core/src/context_manager/history.rs:237` — `drop_last_n_user_turns()` |
| Auto-compact 阈值 | `protocol/src/openai_models.rs:310` |
| Pre-turn compact | `core/src/session/turn.rs:155,727` |
| Mid-turn compact | `core/src/session/turn.rs:467-501` |
| Inline compaction | `core/src/compact.rs:151` — `run_compact_task_inner_impl()` |
| Remote compaction | `core/src/compact_remote.rs` |
| Compaction 后 history 构建 | `core/src/compact.rs:441` — `build_compacted_history()` |
| Prompt cache key | `core/src/client.rs:880` |
| Window generation | `core/src/client.rs:349-358` |
| set_total_tokens_full | `core/src/session/mod.rs:2897` |

## 10) 相关文档

- Agent 循环：`01-agent-loop-and-lifecycle.md`
- Identity 与 System Prompt：`03-agent-identity-and-system-prompt.md`

---

> 读完这篇，你应该能回答 context window 超了以后 Codex 怎么应对——先压缩（Memento 策略），压缩失败就删最老消息，再失败 turn 结束但不丢 session。
