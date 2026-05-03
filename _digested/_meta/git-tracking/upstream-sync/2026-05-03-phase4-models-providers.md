---
title: "Phase 4 Upstream Sync — 06-models-and-providers"
date: "2026-05-03"
status: "completed"
scope: "Phase 4 of progressive production plan: model provider abstraction, model routing & selection, API adapters"
---

# Phase 4 Sync Log — 06-models-and-providers

## 对齐锚点

- **baseline_ref**: `origin/main`
- **baseline_commit**: `35aaa5d9f` — Bound websocket request sends with idle timeout (#20751)
- **ethan_commit**: 本 Phase 开始前 `e21fc6810`（Phase 3）

## 已复核源码路径

### 模型提供商抽象
- `model-provider/src/provider.rs:73` — `ModelProvider` trait 完整定义
- `model-provider/src/provider.rs:145` — `ConfiguredModelProvider` 实现
- `model-provider/src/provider.rs:132` — `create_model_provider()` 工厂函数
- `model-provider/src/provider.rs:27` — `ProviderCapabilities` 结构
- `model-provider/src/auth.rs:78` — `resolve_provider_auth()` 鉴权解析
- `model-provider/src/auth.rs:68` — `auth_manager_for_provider()`
- `model-provider/src/bearer_auth_provider.rs:7` — `BearerAuthProvider` 实现
- `model-provider/src/amazon_bedrock/mod.rs:30` — `AmazonBedrockModelProvider`
- `model-provider/src/amazon_bedrock/auth.rs:98` — Bedrock SigV4 鉴权
- `model-provider/src/models_endpoint.rs:36` — `OpenAiModelsEndpoint`
- `model-provider-info/src/lib.rs:46` — `WireApi` 枚举（仅 Responses）
- `model-provider-info/src/lib.rs:80` — `ModelProviderInfo` 完整字段
- `model-provider-info/src/lib.rs:145` — `validate()` 互斥规则
- `model-provider-info/src/lib.rs:232` — `to_api_provider()` 转换
- `model-provider-info/src/lib.rs:402` — `built_in_model_providers()` 4 个内置 provider
- `model-provider-info/src/lib.rs:435` — `merge_configured_model_providers()`

### codex-api 传输层
- `codex-api/src/provider.rs:42` — `Provider` 传输配置结构
- `codex-api/src/provider.rs:106` — `is_azure_responses_provider()` 检测
- `codex-api/src/auth.rs:29` — `AuthProvider` trait
- `codex-api/src/common.rs:166` — `ResponsesApiRequest` 统一请求
- `codex-api/src/common.rs:67` — `ResponseEvent` 流事件枚举
- `codex-api/src/common.rs:288` — `ResponseStream`
- `codex-api/src/endpoint/session.rs:18` — `EndpointSession` 传输执行
- `codex-api/src/endpoint/models.rs:14` — `ModelsClient`
- `codex-api/src/endpoint/responses.rs:26` — `ResponsesClient`
- `codex-api/src/api_bridge.rs:17` — `map_api_error()` 错误映射

### 模型路由与选择
- `models-manager/src/manager.rs:30` — `ModelsEndpointClient` trait
- `models-manager/src/manager.rs:46` — `RefreshStrategy` 枚举
- `models-manager/src/manager.rs:76` — `ModelsManager` trait
- `models-manager/src/manager.rs:180` — `OpenAiModelsManager`
- `models-manager/src/manager.rs:190` — `StaticModelsManager`
- `models-manager/src/manager.rs:268` — `refresh_available_models()`
- `models-manager/src/manager.rs:394` — `default_model_from_available()`
- `models-manager/src/manager.rs:439` — `construct_model_info_from_candidates()` 三级回退
- `models-manager/src/manager.rs:403` — `find_model_by_longest_prefix()`
- `models-manager/src/model_info.rs:66` — `model_info_from_slug()` fallback 构造
- `models-manager/src/model_info.rs:23` — `with_config_overrides()`
- `models-manager/src/cache.rs` — `ModelsCacheManager`（300s TTL）
- `models-manager/src/lib.rs:13` — `bundled_models_response()` 编译时嵌入
- `core/src/session/turn_context.rs:51` — `TurnContext` 模型绑定
- `core/src/session/turn_context.rs:141` — `TurnContext::with_model()`
- `core/src/session/turn.rs:747` — `maybe_run_previous_model_inline_compact()`
- `core/src/context/model_switch_instructions.rs:4` — `ModelSwitchInstructions`

### API 适配器
- `lmstudio/src/lib.rs:13` — `ensure_oss_ready()` 启动流程
- `lmstudio/src/client.rs:7` — `LMStudioClient`
- `ollama/src/lib.rs:22` — `ensure_oss_ready()` 启动流程
- `ollama/src/lib.rs:62` — `ensure_responses_supported()` 版本检测
- `ollama/src/client.rs:25` — `OllamaClient`（双模式检测）
- `chatgpt/src/chatgpt_client.rs:10` — `chatgpt_get_request()`
- `backend-client/src/client.rs:117` — `Client` + PathStyle
- `backend-client/src/client.rs:447` — `rate_limit_snapshots_from_payload()`
- `backend-client/src/types.rs:20` — `CodeTaskDetailsResponse`
- `codex-client/src/transport.rs` — `HttpTransport` trait + `ReqwestTransport`
- `codex-client/src/retry.rs` — `RetryPolicy`

### 图像路由
- `protocol/src/openai_models.rs:295` — `ModelInfo.input_modalities`
- `core/src/context_manager/history.rs:361` — `normalize_history()` 图像剥离
- `core/src/tools/handlers/view_image.rs:46` — `view_image` 门控
- `tools/src/tool_config.rs:158` — `include_image_gen_tool` 三重门控
- `core/src/session/turn_context.rs:13` — `image_generation_tool_auth_allowed()`

## 产出文档

1. `06-models-and-providers/01-model-provider-abstraction.md`
2. `06-models-and-providers/02-model-routing-and-selection.md`
3. `06-models-and-providers/03-api-adapters.md`

## 复核统计

- 源码文件：50+ paths across 10+ crates
- 涉及 crate：model-provider, model-provider-info, models-manager, codex-api, codex-client, backend-client, lmstudio, ollama, chatgpt, core, protocol, tools
