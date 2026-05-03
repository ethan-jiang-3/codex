---
title: "03 — 沙箱隔离（Sandboxing）"
doc_type: "owner"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解 Codex 如何在三种操作系统上隔离命令执行的人"
purpose: "给出沙箱抽象层、Linux（bwrap+seccomp）、macOS（Seatbelt）、Windows（RestrictedToken）三种后端的实现细节、沙箱选择逻辑和 escalation 重试流程"
owns: "沙箱系统的三层实现：抽象层（SandboxManager）、Linux 后端（bwrap+seccomp）、macOS 后端（sandbox-exec）、Windows 后端（RestrictedToken+ACL+WFP）"
update_when:
  - "新增沙箱后端或沙箱策略变化时"
  - "SandboxType / SandboxPolicy 变化时"
  - "沙箱 escalation 逻辑变化时"
out_of_scope:
  - "审批流程（见 03-agent-core/02-tool-dispatch-and-execution.md）"
  - "exec 工具实现细节（见 02-exec-and-shell.md）"
---

# 03 — 沙箱隔离（Sandboxing）

> [!IMPORTANT]
> 这一篇解释 Codex 如何用三种完全不同的 OS 机制实现同一套沙箱语义。

> 读完这篇你应该能回答：三种沙箱后端分别用什么机制、它们在文件系统和网络隔离上的差异、沙箱被拒绝后怎么 escalation 重试。

## 1) 先记住三件事

1. **沙箱抽象层只有一个 struct**：`SandboxManager` 暴露两个方法 — `select_initial()` 选择沙箱类型，`transform()` 将可移植命令转为平台特定包装命令。
2. **三种后端，三种机制**：macOS 用 Seatbelt（sandbox-exec + sbpl 策略），Linux 用 bwrap + seccomp（双层隔离），Windows 用 RestrictedToken + ACL + WFP + 私有桌面（四层隔离）。
3. **Escalation 是统一的**：被沙箱拒绝 → 检查 `escalate_on_failure()` → 请求用户/Guardian 审批 → 重试 `SandboxType::None`。

## 2) 沙箱抽象层

### 核心类型（`sandboxing/src/manager.rs`）

```rust
pub enum SandboxType {
    None,
    MacosSeatbelt,
    LinuxSeccomp,
    WindowsRestrictedToken,
}

pub enum SandboxablePreference {
    Auto,      // 策略要求时才用
    Require,   // 始终使用
    Forbid,    // 永不使用
}
```

### 选择逻辑（`select_initial()`）

```
Forbid → SandboxType::None
Require → get_platform_sandbox() (平台默认)
Auto → should_require_platform_sandbox() ?
         Yes → 平台默认
         No → None
```

`should_require_platform_sandbox()` 在以下情况返回 true：
- Managed network requirements 活跃
- 网络策略未启用（非 ExternalSandbox）
- 文件系统策略为 Restricted 且无全盘写

### 转换流程（`transform()`）

```
SandboxCommand (portable) → SandboxManager::transform()
  → SandboxExecRequest (platform-specific)
  → command: Vec<String> (包装后的 argv)
```

## 3) macOS：Apple Seatbelt

**Wrapper**：`/usr/bin/sandbox-exec`（写死路径）

**策略格式**：Scheme-like `sbpl` 语言

**策略构造**（`sandboxing/src/seatbelt.rs:603` — `create_seatbelt_command_args()`）：

```
(deny default)                              ← 基本策略
  (allow file-read* (subpath "/ readable_root"))
  (allow file-write* (subpath "/writable_root"))
    (require-not (subpath "/writable_root/.git"))  ← 保护子路径
  (allow network-outbound)                  ← 如网络启用
  (allow system-socket (socket-domain AF_UNIX))   ← Unix socket
  restricted_read_only_platform_defaults    ← 系统框架/lib 可读
```

**参数传递**：通过 `-DKEY=VALUE` 标志传给 `sandbox-exec`。

**网络策略**（`dynamic_network_policy_for_network()`）：
- 无网络 → 空（fail closed）
- 全网络 → `(allow network-outbound) (allow network-inbound)`
- 仅代理 → 允许 loopback 出站到 `localhost:<proxy_port>` + DNS (port 53)
- Unix domain socket → `(allow system-socket (socket-domain AF_UNIX))` + 按路径规则

**保护子路径**：`.git`、`.codex`、`.agents` 等在 writable root 下仍然只读（`require-not` regex）。

## 4) Linux：bwrap + Seccomp

**Wrapper**：`codex-linux-sandbox`（自重复调用，两阶段）

### 第一阶段：Bubblewrap 文件系统隔离

`bwrap.rs` 构造 bwrap argv：
- `--ro-bind / /` — 根文件系统只读
- `--bind <writable_root> <writable_root>` — 可写目录
- `--ro-bind <protected_subpath>` — 保护子路径（`.git` 等）
- `--tmpfs /tmp` — 隔离临时目录
- `--proc /proc` — 新 proc 挂载（可回退到 `--no-proc`）
- `--unshare-user` — 新用户 namespace
- `--unshare-net` — 网络隔离（按需）

平台默认可读路径：`/bin`、`/sbin`、`/usr`、`/etc`、`/lib`、`/lib64`、`/nix/store`

**预探测**：先用 `/bin/true` 测试 bwrap 是否支持 `--proc /proc`。

### 第二阶段：Seccomp BPF

`landlock.rs`（Linux sandbox 内的 seccomp）安装线程级 BPF 过滤器：

**`Restricted` 模式**：
- 拒绝：`connect`, `accept`, `bind`, `listen`, `sendto`, `ptrace`, `io_uring_*`
- 只允许 `AF_UNIX` socket

**`ProxyRouted` 模式**：
- 只允许 `AF_INET` / `AF_INET6`（到代理桥）
- 拒绝 `AF_UNIX`（防止绕过）
- 同样的 ptrace/io_uring 拒绝

**Landlock 旧版回退**：使用 Landlock ABI V5，授予 `/` 读权限 + 指定 writable root 写权限。

## 5) Windows：RestrictedToken + ACL + WFP + 私有桌面

### 四层隔离

| 层 | 机制 | 作用 |
|----|------|------|
| 身份 | 专用 sandbox 用户账户（`CodexSandboxOffline` / `CodexSandboxOnline`） | 进程以受限用户运行 |
| 文件系统 | Capability SID + ACL | 可写 root 加 allow ACE，保护路径加 deny ACE |
| 网络 | WFP kernel filter + Windows Firewall | 按用户 SID 过滤出站流量 |
| 桌面 | 私有 Windows Desktop | 防止屏幕截图和输入注入 |

### 两种后端

| | Legacy (Unelevated) | Elevated |
|---|---|---|
| **Token** | `CreateRestrictedToken` + capability SIDs | Elevated setup 程序 |
| **桌面** | 可选的私有桌面 | — |
| **网络** | Firewall + WFP | WFP kernel filter |
| **FS 覆盖** | ACL deny ACEs | 仅 elevated 后端支持额外 deny write paths |

### Token 类型

- `create_readonly_token_with_cap` — 只读沙箱
- `create_workspace_write_token_with_caps_from` — 工作区写沙箱

### 私有桌面

`Winsta0\codex_sandbox_<random>` — 通过 `CreateDesktopW` 创建，防止：
- 与用户桌面交互
- 屏幕截图
- 输入注入（`SendInput` 无法跨桌面）

## 6) 沙箱 Escalation 重试

`ToolOrchestrator::run()`（`core/src/tools/orchestrator.rs:126`）：

```
Phase 1: 首次尝试（沙箱模式）
  sandbox = select_initial()
  result = tool.run(req, &sandbox_attempt)

Phase 2: 被拒后重试（无沙箱模式）
  if result == SandboxErr::Denied
     && escalate_on_failure() == true
     && wants_no_sandbox_approval() → 请求审批
  if approved:
    sandbox = SandboxType::None
    result = tool.run(req, &no_sandbox_attempt)  // 重试
```

**Escalation 条件**：
- `escalate_on_failure()` — 默认 true
- `wants_no_sandbox_approval(approval_policy)` — `OnFailure` / `UnlessTrusted` / `Granular(sandbox_approval: true)` → 允许
- `Never` / `OnRequest` → 不重试

**沙箱拒绝检测**（`exec.rs:783` — `is_likely_sandbox_denied()`）：
1. 沙箱类型为 `None` 或 exit code 为 0 → 非拒绝
2. 扫描 stdout/stderr：`"operation not permitted"`, `"permission denied"`, `"read-only file system"`, `"seccomp"`, `"sandbox"`, `"landlock"`, `"failed to write file"`
3. 快速排除 exit code 2, 126, 127
4. Linux：exit code == 128 + SIGSYS（seccomp kill）

## 7) 平台对比速查

| | macOS Seatbelt | Linux (bwrap+seccomp) | Windows (RestrictedToken) |
|---|---|---|---|
| **FS 强制** | Seatbelt policy (deny default) | bwrap mount namespace | ACL via capability SIDs |
| **网络强制** | Seatbelt network rules | Seccomp BPF + netns | WFP + Firewall |
| **进程隔离** | 继承策略 | PID ns + PR_SET_NO_NEW_PRIVS | Restricted token |
| **保护路径** | `require-not` regex | bind-mount 只读 | Deny ACEs |
| **代理模式** | Loopback-only | Proxy-routed netns | Firewall + WFP |
| **用户身份** | 当前用户 | 当前用户 | 专用 sandbox 账户 |
| **桌面隔离** | N/A | N/A | 私有 Windows Desktop |
| **Setup 要求** | 无 | 无（系统 bwrap 或 vendored） | Elevated setup 创建用户 |
| **旧版回退** | N/A | Landlock ABI V5 | N/A |

## 8) 源码抓手

| 想看什么 | 从哪里开始 |
|----------|-----------|
| 沙箱类型定义 | `sandboxing/src/manager.rs:22` — `SandboxType` |
| 沙箱选择 | `sandboxing/src/manager.rs:139` — `select_initial()` |
| 沙箱转换 | `sandboxing/src/manager.rs:168` — `transform()` |
| macOS Seatbelt 策略 | `sandboxing/src/seatbelt.rs:603` — `create_seatbelt_command_args()` |
| Linux bwrap | `linux-sandbox/src/bwrap.rs` — `create_bwrap_command_args()` |
| Linux seccomp | `linux-sandbox/src/landlock.rs:169-267` |
| Windows token | `windows-sandbox-rs/src/token.rs` |
| Windows ACL | `windows-sandbox-rs/src/allow.rs` |
| Windows 桌面隔离 | `windows-sandbox-rs/src/desktop.rs` |
| Windows 网络过滤 | `windows-sandbox-rs/src/wfp.rs`, `firewall.rs` |
| Core sandbox 集成 | `core/src/sandboxing/mod.rs:164` — `execute_env()` |
| Escalation 重试 | `core/src/tools/orchestrator.rs:126` — `ToolOrchestrator::run()` |
| 沙箱拒绝检测 | `core/src/exec.rs:783` — `is_likely_sandbox_denied()` |

## 9) 相关文档

- 工具注册：`01-tool-registry-and-discovery.md`
- Exec 与 Shell：`02-exec-and-shell.md`
- Agent 工具分发：`../03-agent-core/02-tool-dispatch-and-execution.md`

---

> 读完这篇，你应该能对着 `SandboxType` 判断每种平台用什么机制、以及沙箱拒绝后 escalation 的两阶段重试怎么走。
