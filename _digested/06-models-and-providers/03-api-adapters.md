---
title: "03 — API 适配器（API Adapters）"
doc_type: "owner"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解 Codex 如何适配各家具体 LLM 提供商的人"
purpose: "给出三个 provider 适配器（LM Studio、Ollama、ChatGPT）的具体机制、backend-client 的 API 层、codex-client 的 HTTP 传输层、以及图像路由和特殊 provider 的处理"
owns: "API 适配面：LM Studio/Ollama/ChatGPT 适配器、backend-client、codex-client HTTP 传输、图像模态路由、Amazon Bedrock/Azure 特殊处理"
update_when:
  - "新增/移除 provider 适配器时"
  - "image routing 或 input_modalities 过滤逻辑变化时"
  - "backend-client 新增 API 端点时"
out_of_scope:
  - "ModelProvider trait 定义（见 01-model-provider-abstraction.md）"
  - "模型路由与选择（见 02-model-routing-and-selection.md）"
---

# 03 — API 适配器（API Adapters）

> [!IMPORTANT]
> 这一篇解释 Codex 如何适配各家具体的 LLM 提供商——从本地 OSS 模型（LM Studio、Ollama）到云端后端（ChatGPT、Amazon Bedrock、Azure），以及底层的 HTTP 传输和图像模态路由。

> 读完这篇你应该能回答：LM Studio 和 Ollama 适配器怎么发现服务器和下载模型、backend-client 提供哪些 API 端点、图像怎么路由到 vision model、以及 Bedrock/Azure 有哪些特殊处理。

## 1) 先记住三件事

1. **本地模型适配器不是推理时的 adapter**：LM Studio 和 Ollama 适配器负责的是服务器发现、健康检查、模型拉取/预热——而不是在推理时做 API 格式转换。所有 provider 都统一走 Responses API。
2. **图像由 `input_modalities` 控制**：`ModelInfo.input_modalities` 决定模型能否接收图像。非 vision 模型的对话历史会自动剥离图像输入。`view_image` 工具也会在非 vision 模型上直接报错。
3. **图像生成有三个门控**：provider 能力（`ProviderCapabilities.image_generation`）+ 鉴权模式（必须是 ChatGPT 登录，API key 不行）+ `Feature::ImageGeneration` feature flag。

## 2) LM Studio 适配器

`lmstudio/src/` — 管理 LM Studio 本地服务器的生命周期。

### 启动流程

`ensure_oss_ready()` (`lib.rs:13`)：
1. 解析模型名（用户配置或默认 `"openai/gpt-oss-20b"`）
2. 创建 `LMStudioClient`，通过 `GET /models` 检查服务器
3. 如果目标模型缺失 → `lms get --yes <model>` 下载
4. 后台 spawn 预热任务（发送空请求 `{"model": "...", "input": "", "max_output_tokens": 1}` 到 `/responses`）

### LMStudioClient

`lmstudio/src/client.rs:7`：

| 方法 | HTTP 端点 | 用途 |
|------|-----------|------|
| `try_from_provider()` | `GET /models` | 查找 lmstudio provider 并检查服务器 |
| `check_server()` | `GET /models` | 返回含安装说明的错误（如不可达） |
| `load_model()` | `POST /responses` | 预热模型（`max_output_tokens: 1`） |
| `fetch_models()` | `GET /models` | 解析 `data[].id` |
| `download_model()` | subprocess | 运行 `lms get --yes <model>` |
| `find_lms()` | PATH 搜索 | 找 `lms` 二进制（先 PATH 后 `~/.lmstudio/bin/lms`） |

## 3) Ollama 适配器

`ollama/src/` — 管理 Ollama 本地服务器的生命周期。

### 启动流程

`ensure_oss_ready()` (`lib.rs:22`)：
1. 创建 `OllamaClient`，通过 `GET /api/tags` 检查服务器
2. 如果目标模型缺失 → `POST /api/pull` 下载（带进度条）
3. 检查 Ollama server 版本是否 >= 0.13.4（Responses API 兼容性）

`ensure_responses_supported()` (line 62)：只有 Ollama >= 0.13.4 才支持 Responses API。版本 `0.0.0`（未发布/开发版）被视为支持。

### OllamaClient

`ollama/src/client.rs:25`：

| 方法 | HTTP 端点 | 用途 |
|------|-----------|------|
| `try_from_oss_provider()` | `GET /api/tags` | 查找 ollama provider 并检查服务器 |
| `try_from_provider()` | `GET /v1/models` 或 `GET /api/tags` | 检测 OpenAI 兼容 vs 原生模式 |
| `fetch_models()` | `GET /api/tags` | 解析 `models[].name` |
| `fetch_version()` | `GET /api/version` | 解析 semver |
| `pull_model_stream()` | `POST /api/pull` | 流式 JSON lines → `PullEvent` |
| `pull_with_reporter()` | — | 驱动 `PullProgressReporter` 的便利包装 |

**双模式检测**：`is_openai_compatible_base_url()` 判断 base URL 是否为 OpenAI 兼容格式，以选择用 `/v1/models` 还是 `/api/tags` 做健康检查。

## 4) ChatGPT 适配器

`chatgpt/src/` — 不用于模型推理，而是用于 ChatGPT 特定的后端操作。

`chatgpt_client.rs:10` — `chatgpt_get_request<T>()`：
- 要求 Codex 后端鉴权（`auth.uses_codex_backend()`）
- URL 由 `config.chatgpt_base_url` 构造
- 从 model provider 的 auth 系统附加鉴权 headers
- 用于工作区设置、apply commands 等非推理操作

## 5) Backend Client

`backend-client/` — Codex 自有后端的 API 客户端。

### Client 结构

`backend-client/src/client.rs:117`：

```rust
pub struct Client {
    base_url: String,
    http: reqwest::Client,
    auth_provider: SharedAuthProvider,
    user_agent: String,
    chatgpt_account_id: Option<String>,
    chatgpt_account_is_fedramp: bool,
    path_style: PathStyle,
}
```

### PathStyle

`client.rs:98` — 两种路径风格：
- **`CodexApi`** — `/api/codex/...` 前缀
- **`ChatGptApi`** — `/wham/...` 前缀

根据 base URL 是否包含 `/backend-api` 自动选择（line 108）。

### API 端点

| 端点 | HTTP | 用途 |
|------|------|------|
| `get_rate_limits()` | `GET /api/codex/usage` | 查询速率限制和用量 |
| `list_tasks()` | `GET /api/codex/tasks/list` | 列出 task 列表（分页） |
| `get_task_details()` | `GET /api/codex/tasks/{id}` | 获取 task 详情 |
| `create_task()` | `POST /api/codex/tasks` | 创建新 task |
| `get_config_requirements_file()` | `GET /api/codex/config/requirements` | 获取配置需求 |
| `send_add_credits_nudge_email()` | `POST /api/codex/accounts/send_add_credits_nudge_email` | 发送充值提醒邮件 |

### 速率限制映射

`rate_limit_snapshots_from_payload()` (line 447) — 将后端响应转换为 `RateLimitSnapshot`：
- `limit_id` — 限额标识
- 主/次时间窗口（如 RPM + RPD）
- `credits` — 当前可用积分
- `plan_type` — 计划类型

### Task 类型

`backend-client/src/types.rs` — `CodeTaskDetailsResponse` 包含 `Turn` 结构：
- `id`、`attempt_placement`、`turn_status`
- `sibling_turn_ids` — 同级 turn（分支）
- `input_items`、`output_items`、`worklog`、`error`

`CodeTaskDetailsResponseExt` trait (line 260)：提供 `unified_diff()`、`assistant_text_messages()`、`user_text_prompt()`、`assistant_error_message()` 方法。

## 6) codex-client：HTTP 传输层

`codex-client/` — 不负责 API 语义，只负责 HTTP 传输。

| 模块 | 用途 |
|------|------|
| `default_client` | `CodexHttpClient` / `CodexRequestBuilder` — 高层 HTTP 客户端 |
| `request` | `Request` / `RequestBody` / `PreparedRequestBody` / `Response` — 请求/响应模型 |
| `transport` | `HttpTransport` trait + `ReqwestTransport` — 传输抽象 |
| `retry` | `RetryPolicy` / `RetryOn` / `run_with_retry` — 可配置重试（429、5xx、传输错误） |
| `sse` | SSE 流解析 |
| `telemetry` | `RequestTelemetry` — 请求遥测钩子 |

特殊 TLS 处理：
- `build_reqwest_client_with_custom_ca()` — 自定义 CA 证书
- `with_chatgpt_cloudflare_cookie_store()` — ChatGPT 后端的 Cloudflare cookie 处理

## 7) 图像路由

### 输入模态过滤

`protocol/src/openai_models.rs`：

- `ModelInfo.input_modalities: Vec<InputModality>` (line 295) — `Text` 和/或 `Image`
- `default_input_modalities()` (line 90) — 返回 `[Text, Image]`（对旧 payload 的保守默认）
- `supports_image_detail_original` (line 277) — 是否支持原始分辨率图像

### 对话历史中的图像剥离

`core/src/context_manager/history.rs:361` — `normalize_history()`：

当 `input_modalities` 不包含 `InputModality::Image` 时，从历史中剥离所有图像输入和图像生成结果（line 294-300）。这防止向非 vision 模型发送图像数据。

### view_image 工具门控

`core/src/tools/handlers/view_image.rs:46`：

如果 `model_info.input_modalities` 不包含 `InputModality::Image` → 返回错误 `"view_image is not allowed because you do not support image inputs"` (line 53)。

### 图像生成的三重门控

`tools/src/tool_config.rs:158` — `include_image_gen_tool` 需要三项同时满足：

1. `image_generation_tool_auth_allowed` = true → 必须是 ChatGPT 登录鉴权，API key 鉴权不行（因为 API key 绕过了 Codex 后端 entitlement 检查）
2. `Feature::ImageGeneration` 已启用
3. `supports_image_generation(model_info)` = true → `model_info.input_modalities` 包含 `InputModality::Image`

此外，`ProviderCapabilities.image_generation` 是 provider 级的上限开关，可通过 `with_image_generation_capability()` 完全禁用。

## 8) 特殊 Provider

### Amazon Bedrock

`model-provider/src/amazon_bedrock/`：

- **鉴权**：AWS SigV4 签名（不是 bearer token）
- **模型目录**：静态（`catalog.rs` 中的 `gpt_5_4_cmb_bedrock_model()` 和 `bedrock_oss_model()`）
- **默认 base URL**：`https://bedrock-mantle.us-east-1.api.aws/openai/v1`
- **Header 剥离**：`remove_headers_not_preserved_by_bedrock_mantle()` 剥离 Bedrock mantle proxy 不能转发的 headers
- **能力限制**：`capabilities()` 全部返回 `false`
- **不支持 WebSocket**：`supports_websockets: false`

### Azure

`codex-api/src/provider.rs:106` — `is_azure_responses_provider()`：

检测方式：
- 名称匹配（`"azure"` 不区分大小写）
- base URL 包含 Azure 标记（`openai.azure.`、`cognitiveservices.azure.`、`aoai.azure.`、`azure-api.`、`azurefd.`、`windows.net/openai`）

Azure provider 被视为标准 Responses API provider（使用 Azure base URL）。无特殊鉴权处理——鉴权由 `ModelProvider` 层的 `BearerAuthProvider` 处理。

## 9) 源码抓手

| 想看什么 | 从哪里开始 |
|----------|-----------|
| LM Studio ensure_oss_ready | `lmstudio/src/lib.rs:13` |
| LMStudioClient | `lmstudio/src/client.rs:7` |
| Ollama ensure_oss_ready | `ollama/src/lib.rs:22` |
| OllamaClient | `ollama/src/client.rs:25` |
| ensure_responses_supported | `ollama/src/lib.rs:62` |
| ChatGPT chatgpt_get_request | `chatgpt/src/chatgpt_client.rs:10` |
| Backend Client | `backend-client/src/client.rs:117` |
| PathStyle 枚举 | `backend-client/src/client.rs:98` |
| rate_limit_snapshots_from_payload | `backend-client/src/client.rs:447` |
| CodeTaskDetailsResponse | `backend-client/src/types.rs:20` |
| codex-client HttpTransport | `codex-client/src/transport.rs` |
| ReqwestTransport | `codex-client/src/transport.rs` |
| RetryPolicy | `codex-client/src/retry.rs` |
| input_modalities 过滤 | `core/src/context_manager/history.rs:361` |
| view_image 门控 | `core/src/tools/handlers/view_image.rs:46` |
| image_gen_tool 三重门控 | `tools/src/tool_config.rs:158` |
| image_generation_tool_auth_allowed | `core/src/session/turn_context.rs:13` |
| Bedrock 静态目录 | `model-provider/src/amazon_bedrock/catalog.rs` |
| Bedrock SigV4 鉴权 | `model-provider/src/amazon_bedrock/auth.rs` |
| Azure 检测 | `codex-api/src/provider.rs:106` |
| with_chatgpt_cloudflare_cookie_store | `codex-client/src/` |

## 10) 相关文档

- Provider 抽象：`01-model-provider-abstraction.md`
- 模型路由与选择：`02-model-routing-and-selection.md`
- Agent 身份与鉴权：`../03-agent-core/03-agent-identity-and-system-prompt.md`

---

> 读完这篇，你应该能区分本地模型的生命周期管理（LM Studio/Ollama）、后端 API 的两种路径风格（CodexApi/ChatGptApi）、图像的三重门控、以及 Bedrock 和 Azure 的特殊处理。
