---
title: "04 — Tools, Execution & Sandboxing 索引"
doc_type: "index"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解 Codex 工具系统、命令执行和沙箱隔离的人"
purpose: "提供工具、执行和沙箱层的阅读路径和文档边界"
owns: "工具注册与分发、exec/shell、沙箱隔离、进程加固相关文档"
update_when:
  - "工具注册或分发机制变化时"
  - "沙箱或执行环境发生重大变化时"
out_of_scope:
  - "单工具业务逻辑细节"
  - "历史归档记录"
---

# 04 — Tools, Execution & Sandboxing 索引

> 本层覆盖 Codex 的能力执行面：工具怎么注册/发现/过滤/分发，命令怎么在沙箱中安全执行，以及相关的进程加固机制。

## 已产出文档

1. `01-tool-registry-and-discovery.md` — 工具注册与发现：`ToolRegistryPlan` 20 步构建、`ToolsConfig` 门控、平台差异化、延迟加载
2. `02-exec-and-shell.md` — 命令执行与 Shell：三种执行模式对比、shell snapshot 加速、Zsh fork 提权、exec-server JSON-RPC 协议
3. `03-sandboxing.md` — 沙箱隔离：macOS Seatbelt / Linux bwrap+seccomp / Windows RestrictedToken 三种后端、escalation 重试
4. `04-file-operations.md` — 文件操作：`ExecutorFileSystem` trait、apply_patch 解析与 4 遍模糊搜索、list_dir BFS 安全策略
5. `05-background-processes.md` — 后台进程管理：`UnifiedExecProcess` 生命周期、HeadTailBuffer、流式输出、进程修剪、网络审批联动

## 关键 crate

- `codex-rs/tools/` — 工具实现与注册
- `codex-rs/exec/` — 命令执行
- `codex-rs/exec-server/` — 执行服务端
- `codex-rs/sandboxing/` — 沙箱抽象层
- `codex-rs/linux-sandbox/` — Linux 沙箱（Seatbelt）
- `codex-rs/windows-sandbox-rs/` — Windows 沙箱
- `codex-rs/process-hardening/` — 进程加固
- `codex-rs/shell-command/` — Shell 命令
- `codex-rs/shell-escalation/` — Shell 权限提升
