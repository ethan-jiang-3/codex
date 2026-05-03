---
title: "03 — Agent 身份与 System Prompt（Identity & System Prompt）"
doc_type: "owner"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解 Agent 身份从哪里来、system prompt 怎么组装、以及 prompt caching 怎么影响上下文注入的人"
purpose: "给出 Agent identity 的建立机制、AGENTS.md 发现与加载、system prompt 的 11 层装配顺序、以及 reference context 的 diff 机制和缓存失效触发条件"
owns: "Agent identity 来源、AGENTS.md 管理、system prompt 装配层次、context fragment 系统和 reference context diff 机制"
update_when:
  - "Agent identity 机制变化（JWT / key 管理）"
  - "新增 context fragment 类型"
  - "system prompt 装配顺序变化"
  - "prompt caching 策略变化"
out_of_scope:
  - "JWT 加密细节（属于安全面）"
  - "具体 context fragment 的内容（属于各子系统）"
  - "compaction 机制（见 04-context-and-compression.md）"
---

# 03 — Agent 身份与 System Prompt（Identity & System Prompt）

> [!IMPORTANT]
> 这一篇解释 Agent "知道自己是谁"以及"向模型说什么话"这两个关键问题的源码真相。

> 读完这篇你应该能回答：Agent 身份怎么建立的、AGENTS.md 从哪里发现、system prompt 各块按什么顺序注入、"冻结快照"怎么 diff、什么时候会触发完整重新注入。

## 1) 先分清三件事

1. **Agent 身份（identity）** 是加密层面的，用 JWT + Ed25519 密钥对来绑定 `agent_runtime_id` 和用户账号。它不是"SOUL.md"式的角色描述。
2. **Agent "人格"（personality / base instructions）** 来自 `protocol/src/prompts/base_instructions/default.md`，是 282 行的 Markdown 模板，定义了行为规则、引用规则、工具使用指南和语调。
3. **System prompt 装配** 通过 20+ 个 `ContextualUserFragment` 实现拼接。每个 fragment 是带 XML 标记（如 `<permissions instructions>...</permissions instructions>`）的独立消息块。

## 2) Agent 身份链路

```
┌─────────────────────────────────────────────────────────────┐
│                   Agent Identity Flow                        │
│                                                              │
│  1. 服务端签发 JWT                                             │
│     AgentIdentityJwtClaims {                                 │
│       agent_runtime_id, agent_private_key,                  │
│       account_id, chatgpt_user_id, email, plan_type          │
│     }                                                        │
│     → 通过 JWKS 验证签名                                       │
│                                                              │
│  2. 客户端生成本地密钥                                          │
│     generate_agent_key_material() → Ed25519 密钥对           │
│     (agent-identity/src/lib.rs:262)                          │
│                                                              │
│  3. Task 级授权                                               │
│     authorization_header_for_agent_task()                   │
│     → 签名 "{agent_runtime_id}:{task_id}:{timestamp}"       │
│     → AgentAssertion HTTP header                             │
│                                                              │
│  4. Agent Bill of Materials                                  │
│     build_abom() → agent_version, harness_id, location      │
└─────────────────────────────────────────────────────────────┘
```

**关键类型**（`agent-identity/src/lib.rs`）：

| 类型 | 行号 | 角色 |
|------|------|------|
| `AgentIdentityJwtClaims` | 66 | JWT claims 结构 |
| `GeneratedAgentKeyMaterial` | — | Ed25519 公私钥对 |
| `AgentAssertionEnvelope` | — | HTTP auth header 载体 |
| `AgentBillOfMaterials` | — | 客户端环境描述 |

## 3) AGENTS.md 发现与加载

`AgentsMdManager`（`core/src/agents_md.rs:47`）负责三个来源的指令合并：

**发现算法**（`agents_md_paths()`，line 213）：
1. 从 cwd 向上走到 project root（默认以 `.git` 为标记）
2. 在 project root 到 cwd 的每个目录中，按优先级找：
   - `AGENTS.override.md`（最高优先）
   - `AGENTS.md`
   - 额外的 `project_doc_fallback_filenames`（可配置）
3. 所有文件用 `"\n\n"` 拼接，受 `project_doc_max_bytes` 预算限制

**用户指令组装**（`user_instructions()`，line 82）：
```
Config::user_instructions (如果有)
  + "\n\n--- project-doc ---\n\n"
  + project AGENTS.md 内容
  + hierarchical agents message (如果 Feature::ChildAgentsMd 开启)
```

**注意**：`CLAUDE.md` 在 core 中不作为 agent identity 源使用。它只出现在 `app-server/src/config/external_agent_config.rs:38` 的外部 agent 迁移路径中。

## 4) System Prompt 装配：11 层顺序

所有上下文注入通过 `Session::build_initial_context()`（`session/mod.rs:2531`）完成。

**Developer message（单条，11 个 section 合并）**：

| # | Fragment | Start Marker | 触发条件 |
|---|----------|-------------|---------|
| 1 | Model switch instructions | `<model_switch>` | 模型发生了变化 |
| 2 | Permissions instructions | `<permissions instructions>` | 始终注入 |
| 3 | Developer instructions override | 无（裸文本） | 非 guardian subagent |
| 4 | Memory tool developer instructions | — | `Feature::MemoryTool` |
| 5 | Collaboration mode instructions | `<collaboration_mode>` | 协作模式活跃 |
| 6 | Realtime instructions | `<realtime_conversation>` | Realtime 状态变化 |
| 7 | Personality spec | `<personality_spec>` | 人格 != 默认 |
| 8 | Apps/Connectors instructions | `<apps_instructions>` | Apps 可用 |
| 9 | Available skills instructions | `<skills_instructions>` | 有已加载 skill |
| 10 | Available plugins instructions | `<plugins_instructions>` | 有已启用 plugin |
| 11 | Git commit trailer instruction | — | `Feature::CodexGitCommit` |

**User message（单条，2 个 section 合并）**：

| # | Fragment | Start Marker | 触发条件 |
|---|----------|-------------|---------|
| 1 | User instructions (AGENTS.md) | `# AGENTS.md instructions for` | AGENTS.md 存在 |
| 2 | Environment context | `<environment_context>` | 始终注入 |

**最终 Prompt item 列表**：
```
[0] Developer message (11 sections merged)
[1] Multi-agent v2 usage hint (如果有)
[2] User message (AGENTS.md + environment)
[3] Guardian policy (如果 separate_guardian_developer_message)
```

## 5) Reference Context：冻结快照与 Diff 机制

`ContextManager`（`context_manager/history.rs:34`）维护一个 `reference_context_item: Option<TurnContextItem>` 作为**基线快照**。

**首次 turn**（无基线）：
- `reference_context_item` 为 `None`
- 调用 `build_initial_context()` → **完整输出**所有 context fragment

**稳态 turn**（有基线）：
- 调用 `build_settings_update_items()` → **只输出变化**的部分
- 检测变化：sandbox 策略、approval 策略、collaboration mode、realtime 状态、personality、model slug、cwd、网络域
- 大部分 turn 这些不变 → 零 overhead

**Rollback 使基线失效**：
- `drop_last_n_user_turns()` 切掉了建立基线的那轮 turn
- → 清除 `reference_context_item = None`
- → 下一轮 turn 重新做完整注入

**Compaction 重新建立基线**：
- Compaction 替换了 history
- → `insert_initial_context_before_last_real_user_or_summary()` 重新注入完整 context

这等价于 "frozen snapshot" 机制：基线在第一个真实 turn 时冻结，后续以 diff 方式维护，直到 rollback/compaction 导致"解冻"。

## 6) Prompt Caching 与缓存失效

**Cache key**：`prompt_cache_key = conversation_id`（thread ID），在整个 conversation 中不变。（`client.rs:880`）

**缓存失效触发条件**：

| 触发条件 | 机制 | 位置 |
|---------|------|------|
| Reference context 被清除 | 重新完整注入 context | `session/mod.rs:2779` |
| Settings diff 产生新 item | 新 developer/user message 插入 | `context_manager/updates.rs:204` |
| History rollback | 可能清除 reference context | `history.rs:237` |
| History compaction | 替换 history → 重新注入 context | `compact.rs` |
| Window generation 递增 | 新 `x-codex-window-id` header → 缓存 bust | `client.rs:356` |

**Compaction 总是导致缓存失效**：`advance_window_generation()` 递增计数器，`x-codex-window-id` 从 `conversation_id:3` 变成 `conversation_id:4`，模型侧缓存在新 key 下重新开始。

## 7) ContextualUserFragment trait

所有 context fragment 实现 `ContextualUserFragment` trait（`context/fragment.rs:40`）：

```rust
trait ContextualUserFragment {
    const ROLE: &'static str;          // "user" | "developer"
    const START_MARKER: &'static str;  // e.g. "<permissions instructions>"
    const END_MARKER: &'static str;    // e.g. "</permissions instructions>"
    fn body(&self) -> String;          // 标记之间的内容
    fn render(&self) -> String;        // = START_MARKER + body() + END_MARKER
    fn into(self) -> ResponseItem;     // 转换为可发送的 message
}
```

`FragmentRegistration` 是类型擦除层：通过 `matches_text(&self, text: &str) -> bool` 识别"这条消息是否属于某个 context fragment"。用于 rollback/trim 时区分 auto-generated 内容 vs 真实用户内容。

## 8) 源码抓手

| 想看什么 | 从哪里开始 |
|----------|-----------|
| Agent identity JWT | `agent-identity/src/lib.rs:66,147,262,334` |
| AGENTS.md 发现 | `core/src/agents_md.rs:47,82,213` |
| System prompt 完整装配 | `core/src/session/mod.rs:2531` — `build_initial_context()` |
| Context fragment trait | `core/src/context/fragment.rs:9,40` |
| Permissions 指令 | `core/src/context/permissions_instructions.rs:57` |
| Environment 上下文 | `core/src/context/environment_context.rs:9` |
| Diff 更新逻辑 | `core/src/context_manager/updates.rs:204` |
| Reference context 管理 | `core/src/context_manager/history.rs:34,50` |
| Prompt cache key | `core/src/client.rs:880` |
| Base instructions 默认值 | `protocol/src/prompts/base_instructions/default.md` |
| Base instructions 解析优先级 | `core/src/session/mod.rs:542` → `protocol/src/openai_models.rs:329` |

## 9) 相关文档

- Agent 循环：`01-agent-loop-and-lifecycle.md`
- 工具分发：`02-tool-dispatch-and-execution.md`
- 上下文与压缩：`04-context-and-compression.md`

---

> 读完这篇，你应该能回答 Agent 身份是谁赋予的、system prompt 每块按什么顺序拼出来的、以及为什么大多数 turn 不会重新发送完整 context（diff 机制）。Context 超了怎么办？看下一篇。
