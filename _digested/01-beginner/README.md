---
title: "01 — Beginner 索引"
doc_type: "index"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "第一次接触 Codex 的使用者，或需要先跑起来再逐步理解实现的人"
purpose: "提供 Beginner 层的推荐阅读路径和文档边界"
owns: "Beginner 层的阅读顺序、文档边界和导航方式"
update_when:
  - "Beginner 文档重组、增删或推荐阅读路径变化时"
  - "首跑路径与高频问题入口变化时"
out_of_scope:
  - "agent core、工具分发、模型路由等实现细节"
  - "历史归档记录"
---

# 01 — Beginner 索引

> 这一层不是为了把 Codex 所有实现都讲完，而是为了让第一次接触的人先拿到 **70-80% 的可用理解**：先跑起来，知道关键文件，遇到问题知道先查哪里。

> Beginner 的默认目标有四件事：
> 1. 先让 Codex 跑起来
> 2. 先知道哪些配置文件会真正影响行为
> 3. 先建立"agent / tools / skills / models"这四根主线的粗心智模型
> 4. `README` 负责导航；单篇正文默认要自己站得住

## 计划文档（尚未编写，待源码复核后逐步填充）

1. `01-quickstart.md` — 5 分钟跑通 Codex
2. `02-config-and-profiles.md` — 安装、配置、Profile、CODEX_HOME
3. `03-models-and-providers.md` — 模型选择、提供商配置、API key
4. `04-tools-and-availability.md` — 工具是什么、怎么开关、为什么看不见
5. `05-skills-basics.md` — 技能入门：是什么、怎么用、怎么创作
6. `06-logs-and-troubleshooting.md` — 默认排错入口：日志、诊断路线

## 这一层最适合承载什么

- 安装、配置、profile、日志、首跑检查
- 模型与 provider 的基本选择
- 工具开关与可用性
- 技能入门与基本创作
- 最常见问题的诊断流程

## 这一层故意不展开什么

- Agent 核心循环、工具分发细节
- 沙箱/exec 深层实现
- 模型路由与 provider adapter 内部机制
- TUI 渲染与事件循环
