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

## 计划文档（尚未编写，待源码复核后逐步填充）

1. `01-skill-system.md` — 技能系统：定义、加载、执行、发现
2. `02-plugin-mechanism.md` — 插件机制：注册、生命周期、扩展点
3. `03-skill-authoring.md` — 技能创作指南：SKILL.md 格式、最佳实践

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
