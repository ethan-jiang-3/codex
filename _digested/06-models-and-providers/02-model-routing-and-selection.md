---
title: "02 — 模型路由与选择（Model Routing & Selection）"
doc_type: "owner"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解 Codex 如何发现、缓存、选择和切换模型的人"
purpose: "给出 ModelsManager 的模型目录管理、刷新策略、模型解析优先级、fallback 链、以及 turn 间的模型切换机制"
owns: "模型路由与选择面：ModelsManager trait、OpenAiModelsManager/StaticModelsManager、模型解析与 fallback、模型切换与上下文衔接"
update_when:
  - "ModelsManager trait 新增方法或刷新策略变化时"
  - "模型解析优先级或 fallback 链变化时"
  - "模型切换机制变化时"
out_of_scope:
  - "Provider trait 定义（见 01-model-provider-abstraction.md）"
  - "具体 provider API 适配（见 03-api-adapters.md）"
---

# 02 — 模型路由与选择（Model Routing & Selection）

> [!IMPORTANT]
> 这一篇解释 Codex 如何从"用户说了一个模型名"到"拿到完整 ModelInfo metadata"——中间的模型目录缓存、刷新策略、slug 解析、以及 turn 间模型切换。

> 读完这篇你应该能回答：模型目录从哪里来、`RefreshStrategy` 三种策略的区别、模型 slug 怎么解析到 `ModelInfo`、切换模型时上下文怎么衔接。

## 1) 先记住三件事

1. **模型目录有三个来源**：编译时嵌入的 `models.json`（静态 fallback）、`/models` 端点的远端响应（ETag 条件刷新）、磁盘缓存 `models_cache.json`（300s TTL）。
2. **模型解析是一个三级回退链**：最长前缀匹配 → 命名空间后缀匹配（如 `custom/gpt-5.3-codex`）→ 从 slug 构造 fallback `ModelInfo`（保守默认值）。
3. **模型切换触发两种行为**：切到更小上下文窗口时自动压缩历史（`maybe_run_previous_model_inline_compact`），每次切换都会注入 `<model_switch>` 开发者消息。

## 2) ModelsManager 架构

```
ModelsManager trait
  ├── OpenAiModelsManager    ← 动态 provider（OpenAI、自定义）
  │     ├── bundled models.json (编译时嵌入)
  │     ├── ModelsCacheManager (磁盘缓存, 300s TTL)
  │     ├── ModelsEndpointClient (远端 /models 端点)
  │     └── remote_models: RwLock<Vec<ModelInfo>>
  │
  └── StaticModelsManager    ← 静态 provider（Bedrock、config 指定 catalog）
        └── remote_models: Vec<ModelInfo> (内存中不变)
```

## 3) ModelsEndpointClient：模型端点抽象

`models-manager/src/manager.rs:30`：

```rust
#[async_trait]
pub trait ModelsEndpointClient: Debug + Send + Sync {
    fn has_command_auth(&self) -> bool;
    async fn uses_codex_backend(&self) -> bool;
    async fn list_models(&self, client_version: &str) -> CoreResult<(Vec<ModelInfo>, Option<String>)>;
}
```

由 provider 实现。返回模型列表和可选的 ETag。具体实现 `OpenAiModelsEndpoint`（`model-provider/src/models_endpoint.rs:36`）连接 provider 的 `/models` 端点，使用 5 秒超时。

## 4) RefreshStrategy：三种刷新策略

`models-manager/src/manager.rs:46`：

```rust
pub enum RefreshStrategy {
    Online,            // 始终从网络获取
    Offline,           // 仅使用缓存
    OnlineIfUncached,  // 缓存优先，无缓存时回退到网络
}
```

| 策略 | 行为 | 适用场景 |
|------|------|---------|
| `Online` | 始终 fetch，merge 到 bundled models | 用户主动刷新 |
| `Offline` | 只从 `models_cache.json` 加载 | 离线或启动快速路径 |
| `OnlineIfUncached` | 先检查缓存新鲜度，过期才联网 | 常规启动路径 |

**条件刷新**：`should_refresh_models()` 确保只有 Codex 后端或命令鉴权的 provider 会刷新模型——第三方 provider 的模型稳定，不会无谓访问其 `/models` 端点。

## 5) ModelsManager Trait

`models-manager/src/manager.rs:76`，关键方法：

| 方法 | 职责 |
|------|------|
| `list_models(refresh_strategy)` | 列出所有可用模型，按优先级排序，按鉴权/可见性过滤 |
| `raw_model_catalog(refresh_strategy)` | 返回活跃的原始 `ModelsResponse` |
| `get_remote_models()` | 返回内存中的远程模型（不刷新、不读缓存） |
| `try_get_remote_models()` | 非阻塞版本，使用 `try_read` |
| `get_default_model(model, refresh_strategy)` | 解析要使用的模型：有指定则直接用，否则选择预设中的默认模型 |
| `get_model_info(model, config)` | 查找模型元数据（含 config overrides） |
| `build_available_models(remote_models)` | 按优先级排序 → 转 `ModelPreset` → 按鉴权过滤 → 标记默认 |
| `list_collaboration_modes()` | 列出协作模式 preset |
| `refresh_if_new_etag(etag)` | 基于 ETag 的条件刷新 |

### OpenAiModelsManager 刷新流程

`models-manager/src/manager.rs:268` — `refresh_available_models()`：
1. 按 `RefreshStrategy` 决定路径
2. `fetch_and_update_models()` 调用 `endpoint_client.list_models()`，持久化缓存
3. `apply_remote_models()` 将远端模型按 slug upsert 到内存中的列表

### StaticModelsManager

`models-manager/src/manager.rs:190`：持有固定的 `Vec<ModelInfo>` 在内存中。无刷新、无缓存、无 ETag。用于 Amazon Bedrock 和 `config_model_catalog` 场景。

## 6) 模型解析优先级

`construct_model_info_from_candidates()` (`manager.rs:439`) 三级回退：

```
1. find_model_by_longest_prefix(model, candidates)
   → 按 slug 最长前缀匹配（如 "gpt-5.2-codex" 匹配 "gpt-5.2"）

2. find_model_by_namespaced_suffix(model, candidates)
   → 剥离前导 namespace（如 "custom/"）后重试匹配

3. model_info_from_slug(model)
   → 构造 fallback ModelInfo：
     - context_window: 272,000
     - used_fallback_model_metadata = true (标记为已知信息不足)
     - 基础指令使用保守默认值
     - 特殊 personality 消息用于 gpt-5.2-codex 和 exp-codex-personality
```

### 模型 Preset 排序

`build_available_models()` 创建的 preset 按以下顺序排列：
1. `is_default` 标记的模型优先
2. 按 `priority` 字段排序
3. 过滤条件：鉴权模式 + 可见性（`ModelVisibility`）

### Config Overrides

`with_config_overrides()` (`model_info.rs:23`) 允许用户覆盖模型元数据：
- `model_context_window` → 覆盖上下文窗口
- `model_auto_compact_token_limit` → 覆盖自动压缩触发阈值
- `tool_output_token_limit` → 覆盖工具输出 token 限制
- `base_instructions` → 覆盖基础指令
- `model_supports_reasoning_summaries` → 覆盖推理摘要支持
- `personality_enabled` → 是否注入 personality 模板

## 7) 模型切换机制

### TurnContext 与模型绑定

`core/src/session/turn_context.rs:51` — `TurnContext` 是每 turn 的不可变快照，携带：
- `model_info: ModelInfo` (line 57) — 已解析的模型元数据
- `provider: SharedModelProvider` (line 59) — 运行时 provider
- `tools_config: ToolsConfig` — 基于模型能力过滤的工具配置

`with_model(model, models_manager)` (line 141)：创建使用不同模型的新的 `TurnContext`：
- 解析新模型的 `ModelInfo`
- 调整 `reasoning_effort` 为新模型支持的中等水平
- 基于新模型能力重新构建 `ToolsConfig`

### 模型切换时的上下文衔接

**切到更小上下文窗口时**（`core/src/session/turn.rs:747`）：
- `maybe_run_previous_model_inline_compact()` 检测：`previous_model.slug != current_model.slug && old_context_window > new_context_window`
- 触发提前压缩，将历史缩小后再继续

**ModelSwitchInstructions**（`core/src/context/model_switch_instructions.rs:4`）：
- 每次模型切换时注入 `<model_switch>` 标签包裹的开发者消息
- 告诉新模型"你是从哪个模型切换过来的"
- 作为 `ContextualUserFragment` 注入到 system prompt 中

### Review 线程的模型选择

`core/src/session/review.rs:6` — `spawn_review_thread()`：
- 如果 `config.review_model` 已设置，使用该模型
- 否则继承 `parent_turn_context.model_info.slug`

## 8) 模型缓存

### ModelsCacheManager

`models-manager/src/cache.rs` — 管理 `models_cache.json` 的磁盘缓存。

`ModelsCache` 结构：
- `fetched_at: DateTime<Utc>` — 抓取时间戳
- `etag: Option<String>` — 条件刷新的 ETag
- `client_version: Option<String>` — 抓取时的客户端版本
- `models: Vec<ModelInfo>` — 缓存的模型列表

**新鲜度判断**：比较 `fetched_at + TTL`（300s）与 `Utc::now()`。`load_fresh()` 还验证缓存的 client version 与当前 client version 是否匹配，不匹配则视为过期。

### 编译时嵌入

`bundled_models_response()` (`models-manager/src/lib.rs:13`)：在编译时通过 `include_str!` 加载 `models.json` 作为兜底模型目录。

## 9) 源码抓手

| 想看什么 | 从哪里开始 |
|----------|-----------|
| ModelsManager trait | `models-manager/src/manager.rs:76` |
| OpenAiModelsManager | `models-manager/src/manager.rs:180` |
| StaticModelsManager | `models-manager/src/manager.rs:190` |
| RefreshStrategy 枚举 | `models-manager/src/manager.rs:46` |
| refresh_available_models | `models-manager/src/manager.rs:268` |
| construct_model_info_from_candidates | `models-manager/src/manager.rs:439` |
| find_model_by_longest_prefix | `models-manager/src/manager.rs:403` |
| model_info_from_slug | `models-manager/src/model_info.rs:66` |
| with_config_overrides | `models-manager/src/model_info.rs:23` |
| ModelsEndpointClient trait | `models-manager/src/manager.rs:30` |
| OpenAiModelsEndpoint | `model-provider/src/models_endpoint.rs:36` |
| ModelsCacheManager | `models-manager/src/cache.rs` |
| TurnContext | `core/src/session/turn_context.rs:51` |
| TurnContext::with_model | `core/src/session/turn_context.rs:141` |
| maybe_run_previous_model_inline_compact | `core/src/session/turn.rs:747` |
| ModelSwitchInstructions | `core/src/context/model_switch_instructions.rs:4` |
| default_model_from_available | `models-manager/src/manager.rs:394` |
| bundled_models_response | `models-manager/src/lib.rs:13` |

## 10) 相关文档

- Provider 抽象：`01-model-provider-abstraction.md`
- API 适配器：`03-api-adapters.md`
- 上下文与压缩：`../03-agent-core/04-context-and-compression.md`

---

> 读完这篇，你应该能追踪模型名从用户输入 → `get_default_model()` → `construct_model_info_from_candidates()` 三级回退 → `TurnContext` 绑定 → 模型切换时的压缩 + 指令注入的完整链路。
