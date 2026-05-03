---
title: "05 — Skills & Plugins 索引"
doc_type: "index"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解 Codex 技能系统和插件机制的人"
purpose: "提供技能与插件层的阅读路径和文档边界"
owns: "技能系统、插件机制、skill 创作与发现相关文档"
update_when:
  - "技能系统或插件机制发生重大变化时"
  - "新增扩展面时"
out_of_scope:
  - "单个 skill 或 plugin 的业务实现"
  - "历史归档记录"
---

# 05 — Skills & Plugins 索引

> 本层覆盖 Codex 的可编程扩展面：技能（skills）如何定义、加载和执行，插件（plugins）如何扩展系统能力。

## 已产出文档

1. `01-skill-system.md` — 技能系统：SKILL.md 格式、6 个 skill root 来源、BFS 发现与解析、4 阶段触发流程、2 级上下文注入、预算管理
2. `02-plugin-mechanism.md` — 插件机制：声明式模型、`PluginId` 格式、`plugin.json` 清单、4 种扩展点、marketplace 发现与商店、完整生命周期
3. `03-skill-authoring.md` — 技能创作指南：SKILL.md + openai.yaml 字段规范、验证约束、目录结构约定、skill-creator 最佳实践

## 关键 crate

- `codex-rs/skills/` — 技能系统
- `codex-rs/core-skills/` — 核心内置技能
- `codex-rs/plugin/` — 插件系统
- `codex-rs/core-plugins/` — 核心内置插件

## 关键概念（待源码复核确认）

- Skill 定义格式（SKILL.md / front matter）
- Skill 加载与发现机制
- Skill 执行与上下文注入
- Plugin trait 与注册
- Plugin 生命周期管理
