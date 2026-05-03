---
title: "06 — Models & Providers 索引"
doc_type: "index"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解 Codex 模型提供商抽象、路由和 API 适配的人"
purpose: "提供模型与提供商层的阅读路径和文档边界"
owns: "模型提供商抽象、模型管理、API 路由与适配相关文档"
update_when:
  - "模型提供商架构或路由机制变化时"
  - "新增或移除 provider 支持时"
out_of_scope:
  - "单个 provider 的内部实现细节"
  - "历史归档记录"
---

# 06 — Models & Providers 索引

> 本层覆盖 Codex 的模型抽象面：如何统一接入 OpenAI、Anthropic、Ollama、LM Studio 等不同提供商，模型如何发现、选择和路由。

## 已产出文档

1. `01-model-provider-abstraction.md` — 模型提供商抽象：`ModelProvider` trait、`ModelProviderInfo` 配置、三层鉴权链、codex-api 统一 Responses API 传输
2. `02-model-routing-and-selection.md` — 模型路由与选择：`ModelsManager` trait、三种刷新策略、模型 Slug 三级回退解析、模型切换时的压缩与指令注入
3. `03-api-adapters.md` — API 适配器：LM Studio/Ollama 本地模型管理、ChatGPT 后端、backend-client API 端点、codex-client HTTP 传输、图像模态三重门控、Bedrock/Azure 特殊处理

## 关键 crate

- `codex-rs/model-provider/` — 模型提供商抽象核心
- `codex-rs/model-provider-info/` — 模型元信息
- `codex-rs/models-manager/` — 模型管理
- `codex-rs/lmstudio/` — LM Studio 适配
- `codex-rs/ollama/` — Ollama 适配
- `codex-rs/chatgpt/` — ChatGPT 适配
- `codex-rs/backend-client/` — 后端 API client（Codex 自有后端）
- `codex-rs/codex-api/` — Codex API 定义
- `codex-rs/codex-client/` — Codex client
- `codex-rs/codex-backend-openapi-models/` — OpenAPI 模型定义
