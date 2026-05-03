---
title: "01 — 模型提供商抽象层（Model Provider Abstraction）"
doc_type: "owner"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解 Codex 如何统一接入多种 LLM 提供商的人"
purpose: "给出 ModelProvider trait、ModelProviderInfo 配置、AuthProvider 鉴权链、以及 codex-api 的统一 Responses API 传输层"
owns: "模型提供商抽象：ModelProvider trait、ModelProviderInfo 定义、AuthProvider 鉴权、codex-api 统一传输"
update_when:
  - "ModelProvider trait 新增方法或语义变化时"
  - "ModelProviderInfo 新增字段或 WireApi 变化时"
  - "AuthProvider 鉴权机制变化时"
  - "新增/移除内置 provider 时"
out_of_scope:
  - "具体 provider 适配器实现（见 03-api-adapters.md）"
  - "模型路由与选择（见 02-model-routing-and-selection.md）"
---

# 01 — 模型提供商抽象层（Model Provider Abstraction）

> [!IMPORTANT]
> 这一篇解释 Codex 如何用一套 trait + 配置 + 传输层的三层抽象，统一接入 OpenAI、Anthropic、Ollama、LM Studio、Amazon Bedrock 等不同提供商。

> 读完这篇你应该能回答：`ModelProvider` trait 有哪些方法、`ModelProviderInfo` 包含哪些配置字段、鉴权链怎么从 CodexAuth → AuthProvider → HTTP Header、以及 codex-api 如何统一所有 provider 的 wire protocol。

## 1) 先记住三件事

1. **统一 wire protocol**：Codex 已经标准化为 OpenAI Responses API（`WireApi::Responses`）。旧的 Chat Completions 格式（`WireApi::Chat`）已被移除，反序列化时直接报错。
2. **三层鉴权链**：`CodexAuth`（用户身份）→ `ModelProvider::api_auth()`（解析为 AuthProvider）→ `add_auth_headers()` / `apply_auth()`（注入 HTTP headers）。
3. **Provider 能力是上限声明**：`ProviderCapabilities` 是 provider 拥有的能力上限，调用方可以在此基础上进一步禁用，但不应超出 provider 声明的能力。

## 2) 架构总览

```
ModelProviderInfo (配置层：序列化的 provider 定义)
  │
  ▼
ModelProvider trait (运行时层：鉴权解析、能力查询、模型管理)
  │
  ├── ConfiguredModelProvider (OpenAI 兼容 provider 的默认实现)
  └── AmazonBedrockModelProvider (AWS SigV4 鉴权的特殊实现)
  │
  ▼
codex-api (传输层：Provider、AuthProvider、EndpointSession、ResponsesClient)
```

## 3) ModelProviderInfo：Provider 配置定义

定义在 `model-provider-info/src/lib.rs:80`。

### 字段一览

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | `String` | 友好显示名称 |
| `base_url` | `Option<String>` | OpenAI 兼容 API 的 base URL |
| `env_key` | `Option<String>` | 存储用户 API key 的环境变量名 |
| `env_key_instructions` | `Option<String>` | 设置环境变量的帮助文本 |
| `experimental_bearer_token` | `Option<String>` | 内联 bearer token（安全考虑，不推荐） |
| `auth` | `Option<ModelProviderAuthInfo>` | 命令驱动的 bearer token 配置 |
| `aws` | `Option<ModelProviderAwsAuthInfo>` | AWS SigV4 鉴权配置（profile + region） |
| `wire_api` | `WireApi` | 始终为 `Responses` |
| `query_params` | `Option<HashMap>` | 附加到 base URL 的查询参数 |
| `http_headers` | `Option<HashMap>` | 额外的 HTTP 请求头 |
| `env_http_headers` | `Option<HashMap>` | 从环境变量读取的 HTTP 请求头 |
| `request_max_retries` | `Option<u64>` | 最大重试次数（默认 4，上限 100） |
| `stream_max_retries` | `Option<u64>` | 流重连最大次数（默认 5，上限 100） |
| `stream_idle_timeout_ms` | `Option<u64>` | 流空闲超时（默认 300,000ms） |
| `websocket_connect_timeout_ms` | `Option<u64>` | WebSocket 连接超时（默认 15,000ms） |
| `requires_openai_auth` | `bool` | 是否显示登录界面并存储 auth 到 auth.json |
| `supports_websockets` | `bool` | 是否支持 WebSocket 传输 |

### 关键方法

- **`validate()`** (line 145)：确保互斥规则 — `aws` 不能与 `env_key`、`experimental_bearer_token`、`auth` 或 `requires_openai_auth` 共存；`aws` 不能与 `supports_websockets` 组合；`auth.command` 必须非空。
- **`to_api_provider()`** (line 232)：将配置转换为 `codex_api::Provider`。根据鉴权模式确定默认 base URL：ChatGPT 鉴权用 `https://chatgpt.com/backend-api/codex`，API key 鉴权用 `https://api.openai.com/v1`。
- **`api_key()`** (line 268)：从 `env_key` 指定的环境变量读取 API key。如果 key 必须但缺失则返回 `EnvVarError`。
- **`is_amazon_bedrock()`** (line 382)：按 provider 名称判断是否为 Bedrock provider。
- **`supports_remote_compaction()`** (line 386)：OpenAI 和 Azure provider 返回 true，表示压缩可由远端执行。

### 内置 Provider 目录

`built_in_model_providers()` (line 402) 返回 4 个内置 provider：

| Provider ID | 说明 | 默认 base URL |
|-------------|------|---------------|
| `openai` | OpenAI 官方 | `https://api.openai.com/v1` |
| `amazon-bedrock` | Amazon Bedrock | `https://bedrock-mantle.us-east-1.api.aws/openai/v1` |
| `ollama` | Ollama 本地 | `http://localhost:11434/v1` |
| `lmstudio` | LM Studio 本地 | `http://localhost:1234/v1` |

用户可在 `config.toml` 中添加自定义 provider。内置 provider 通常不可覆盖，但 Bedrock 的 `aws.profile` 和 `aws.region` 例外。

## 4) ModelProvider Trait：运行时抽象

定义在 `model-provider/src/provider.rs:73`。

```rust
#[async_trait]
pub trait ModelProvider: Debug + Send + Sync {
    fn info(&self) -> &ModelProviderInfo;
    fn capabilities(&self) -> ProviderCapabilities;
    fn auth_manager(&self) -> Option<Arc<AuthManager>>;
    async fn auth(&self) -> Option<CodexAuth>;
    fn account_state(&self) -> ProviderAccountResult;
    async fn api_provider(&self) -> Result<Provider>;
    async fn runtime_base_url(&self) -> Result<Option<String>>;
    async fn api_auth(&self) -> Result<SharedAuthProvider>;
    fn models_manager(&self, codex_home: PathBuf, config_model_catalog: Option<ModelsResponse>) -> SharedModelsManager;
}
```

### 方法职责

| 方法 | 职责 |
|------|------|
| `info()` | 返回配置元数据（序列化的 provider 定义） |
| `capabilities()` | Provider 能力上限声明 |
| `auth_manager()` | Provider 级别的 auth manager（如命令驱动的 token） |
| `auth()` | 当前 provider 级别的 auth 快照 |
| `account_state()` | 面向 UI 的账户状态（API key、ChatGPT、Bedrock 等） |
| `api_provider()` | 将 provider 配置转为 API client 可用的 `Provider` |
| `runtime_base_url()` | 请求时实际使用的 base URL |
| `api_auth()` | 返回用于附加请求凭证的 `AuthProvider` |
| `models_manager()` | 创建该 provider 的模型管理器 |

### ProviderCapabilities

`model-provider/src/provider.rs:27`：

```rust
pub struct ProviderCapabilities {
    pub namespace_tools: bool,   // 是否支持 namespaced tools
    pub image_generation: bool,  // 是否支持图像生成
    pub web_search: bool,        // 是否支持 web 搜索
}
```

默认全部为 `true`。`AmazonBedrockModelProvider` 全部返回 `false`。

### 两种实现

**ConfiguredModelProvider** (line 145)：OpenAI 兼容 provider 的默认实现。
- `account_state()`：将 `CodexAuth` 变体映射为 `ProviderAccount`（提取 email 和 plan type）
- `models_manager()`：有 `config_model_catalog` 时使用 `StaticModelsManager`，否则创建 `OpenAiModelsEndpoint` 驱动 `OpenAiModelsManager`

**AmazonBedrockModelProvider** (`amazon_bedrock/mod.rs:30`)：覆盖多个方法以支持 AWS SigV4 鉴权和静态模型目录。

### 工厂函数

`create_model_provider()` (line 132)：根据 `provider_info.is_amazon_bedrock()` 分发到两个实现。

## 5) 鉴权链（Auth Chain）

### 三层结构

```
CodexAuth (用户身份层)
  ├── AgentIdentity  → AgentIdentityAuthProvider (签名 authorization header)
  ├── ApiKey         → BearerAuthProvider (Authorization: Bearer <token>)
  ├── Chatgpt        → BearerAuthProvider + ChatGPT-Account-ID header
  └── ChatgptAuthTokens → BearerAuthProvider
```

### 解析流程

`resolve_provider_auth()` (`model-provider/src/auth.rs:78`)：
1. 检查 provider 级别的 bearer token（`env_key` 或 `experimental_bearer_token`）
2. 回退到用户级 `CodexAuth`

`auth_manager_for_provider()` (line 68)：如果 provider 配置了 `auth.command`，创建命令驱动的 `AuthManager`；否则返回调用方的 base auth manager。

### BearerAuthProvider

`model-provider/src/bearer_auth_provider.rs:7`：持有 `token`、`account_id`、`is_fedramp_account`。实现 `AuthProvider::add_auth_headers()`，设置 `Authorization: Bearer <token>`、`ChatGPT-Account-ID`、`X-OpenAI-Fedramp` 等 headers。

### AWS SigV4 鉴权

`model-provider/src/amazon_bedrock/auth.rs`：Bedrock 有两种鉴权方式：
- **EnvBearerToken**：从 `AWS_BEARER_TOKEN_BEDROCK` 环境变量读取
- **AwsSdkAuth**：使用 AWS SDK 进行 SigV4 签名

`BedrockMantleSigV4AuthProvider` 不重写 `add_auth_headers()`，而是重写 `apply_auth()`，因为 SigV4 需要签名完整的请求（URL、headers、body）。它还会剥离 Bedrock Mantle 不保留的 `session_id` header。

## 6) codex-api：统一传输层

### Provider（传输配置）

`codex-api/src/provider.rs:42`：

```rust
pub struct Provider {
    pub name: String,
    pub base_url: String,
    pub query_params: Option<HashMap<String, String>>,
    pub headers: HeaderMap,
    pub retry: RetryConfig,
    pub stream_idle_timeout: Duration,
}
```

关键方法：
- `url_for_path()` — 构建完整 URL（含 query params）
- `build_request()` — 创建带 method、URL、headers、可选 body 的 HTTP Request
- `websocket_url_for_path()` — 将 `http` 转为 `ws`、`https` 转为 `wss`
- `is_azure_responses_endpoint()` — 检测 Azure base URL（匹配 `openai.azure.`、`cognitiveservices.azure.`、`windows.net/openai` 等模式）

### AuthProvider Trait

`codex-api/src/auth.rs:29`：

```rust
#[async_trait]
pub trait AuthProvider: Send + Sync {
    fn add_auth_headers(&self, headers: &mut HeaderMap);
    async fn apply_auth(&self, request: Request) -> Result<Request, AuthError>;
}
```

两级设计：
- **仅 header 的 provider** 实现 `add_auth_headers()`（轻量、非阻塞）
- **请求签名的 provider**（如 SigV4）重写 `apply_auth()` 以检查/修改完整请求

`AuthError` 有两种变体：`Build(String)`（永久性）和 `Transient(String)`（可重试），分别映射到 `TransportError::Build` 和 `TransportError::Network`。

### 统一 API 格式

`codex-api/src/common.rs` — 全部使用 OpenAI Responses API 作为规范 wire format：

- **`ResponsesApiRequest`** (line 166)：统一请求载荷，包含 `model`、`instructions`、`input: Vec<ResponseItem>`、`tools`、`tool_choice`、`parallel_tool_calls`、`reasoning`、`text: TextControls`（verbosity + format）、`prompt_cache_key`
- **`ResponseEvent`** 枚举 (line 67)：统一流事件 — `Created`、`OutputItemDone`、`ServerModel`、`Completed`（含 `token_usage` 和 `end_turn`）、`OutputTextDelta`、`ToolCallInputDelta`、`ReasoningSummaryDelta`、`ReasoningContentDelta`、`RateLimits`、`ModelsEtag`
- **`ResponseStream`** (line 288)：包装 `mpsc::Receiver`，实现 `Stream` trait

### EndpointSession：传输执行

`codex-api/src/endpoint/session.rs` 是共享的底层传输包装器。持有 `transport: T: HttpTransport`、`provider: Provider`、`auth: SharedAuthProvider`。

两个执行方法：
- `execute_with()` — 应用 auth → 调用 `transport.execute(req)`（用于一元请求如 `/models`）
- `stream_with()` — 同样流程但调用 `transport.stream(req)`（用于流请求如 `/responses`）

两者都通过 `run_with_request_telemetry()` 包装传输调用，实现基于 `RetryPolicy` 的重试循环。

### 错误映射

`codex-api/src/api_bridge.rs:17` — `map_api_error()` 将 `ApiError` 映射为 `CodexErr`：
- HTTP 503 → `ServerOverloaded`
- HTTP 400 → `InvalidRequest` 或 `CyberPolicy`
- HTTP 500 → `InternalServerError`
- HTTP 429 → `UsageLimitReached`（含 plan type、reset time、rate limits）
- 从 `x-request-id`、`x-oai-request-id`、`cf-ray` 提取请求追踪 ID
- 从 `x-openai-authorization-error` 和 `x-error-json` base64 headers 提取鉴权错误

### 端点客户端

| 客户端 | 位置 | 用途 |
|--------|------|------|
| `ModelsClient` | `endpoint/models.rs` | GET `/models` — 获取模型列表 |
| `ResponsesClient` | `endpoint/responses.rs` | POST `/responses` — 流式推理 |
| `RealtimeWebsocketClient` | `endpoint/realtime_websocket.rs` | 实时音频 API（WebSocket） |
| Compaction 端点 | `endpoint/compact.rs` | 对话压缩 |
| Memories 端点 | `endpoint/memories.rs` | 记忆摘要 |

## 7) 从配置到 API 调用的完整链路

```
config.toml / built_in_model_providers()
  │
  ▼
ModelProviderInfo { name, base_url, auth, wire_api, retry... }
  │  validate() → to_api_provider()
  ▼
create_model_provider(info, auth_manager)
  ├── AmazonBedrockModelProvider (AWS SigV4)
  └── ConfiguredModelProvider  (OpenAI 兼容)
  │
  ▼
ModelClient (core/src/client.rs)
  ├── provider.auth().await        → CodexAuth
  ├── provider.api_provider().await → codex_api::Provider (base_url, headers, retry)
  ├── provider.api_auth().await    → SharedAuthProvider (bearer / SigV4 / agent identity)
  │
  ▼
ResponsesClient.stream_request()
  → EndpointSession.stream_with()
    → auth.apply_auth(request)
    → transport.stream(request)
    → SSE parsing → ResponseStream
```

## 8) 源码抓手

| 想看什么 | 从哪里开始 |
|----------|-----------|
| ModelProvider trait | `model-provider/src/provider.rs:73` |
| ConfiguredModelProvider | `model-provider/src/provider.rs:145` |
| AmazonBedrockModelProvider | `model-provider/src/amazon_bedrock/mod.rs:30` |
| create_model_provider 工厂 | `model-provider/src/provider.rs:132` |
| ModelProviderInfo 定义 | `model-provider-info/src/lib.rs:80` |
| 内置 provider 目录 | `model-provider-info/src/lib.rs:402` |
| WireApi 枚举 | `model-provider-info/src/lib.rs:46` |
| BearerAuthProvider | `model-provider/src/bearer_auth_provider.rs:7` |
| resolve_provider_auth | `model-provider/src/auth.rs:78` |
| Bedrock SigV4 鉴权 | `model-provider/src/amazon_bedrock/auth.rs:98` |
| ProviderCapabilities | `model-provider/src/provider.rs:27` |
| codex_api::Provider | `codex-api/src/provider.rs:42` |
| AuthProvider trait | `codex-api/src/auth.rs:29` |
| ResponsesApiRequest | `codex-api/src/common.rs:166` |
| ResponseEvent 枚举 | `codex-api/src/common.rs:67` |
| EndpointSession | `codex-api/src/endpoint/session.rs:18` |
| map_api_error 错误映射 | `codex-api/src/api_bridge.rs:17` |

## 9) 相关文档

- 模型路由与选择：`02-model-routing-and-selection.md`
- API 适配器：`03-api-adapters.md`
- Agent 身份与鉴权：`../03-agent-core/03-agent-identity-and-system-prompt.md`

---

> 读完这篇，你应该能追踪 `config.toml` 中一个 provider 定义 → `ModelProviderInfo` 解析 → `ModelProvider` trait → `AuthProvider` 注入 headers → `ResponsesClient` 发出 HTTP 请求的完整链路。
