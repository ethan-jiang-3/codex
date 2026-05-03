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

## 计划文档（尚未编写，待源码复核后逐步填充）

1. `01-model-provider-abstraction.md` — 模型提供商抽象层：trait、adapter、transport
2. `02-model-routing-and-selection.md` — 模型路由与选择：主模型、auxiliary、fallback
3. `03-api-adapters.md` — API 适配器：各家 provider 的具体适配

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
