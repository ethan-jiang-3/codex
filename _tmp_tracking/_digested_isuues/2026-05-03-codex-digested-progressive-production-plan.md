---
title: "Codex _digested Progressive Production Plan"
date: "2026-05-03"
status: "active"
scope: "从零开始，分阶段生产 codex _digested/ 下全部 8 个主题目录的 owner 文档"
branch: "ethan"
---

# Codex _digested Progressive Production Plan

目标：把 codex `_digested/` 从"只有 README 设计文档的空壳"推进到"每一篇 owner 文档都有经过源码验证的事实正文"。

原则：
- 源码与 tests 是唯一真相源，不抄 release note，不凭记忆写。
- 按窄切片推进：每个切片只深挖一个主题的 1-2 篇文档。
- 主题依赖从底层到上层：先架构全景，再核心机制，再扩展面，最后入门文档。
- 每完成一个 Phase，追加记录到 sync log。

---

## 总体路线图

```
Phase 0: Git 基线锁定 + 依赖图扫描        (1 切片，不改正文)
Phase 1: 02-system-architecture           (2 篇文档)
Phase 2: 03-agent-core                    (4 篇文档)
Phase 3: 04-tools-execution-sandboxing    (5 篇文档)
Phase 4: 06-models-and-providers          (3 篇文档)
Phase 5: 05-skills-and-plugins            (3 篇文档)
Phase 6: 07-state-memory-and-sessions     (2 篇文档)
Phase 7: 08-tui-cli-and-integration       (3 篇文档)
Phase 8: 01-beginner                      (6 篇文档)
Phase 9: 交叉复核 + 一致性收尾             (不改正文为主)
```

共约 **28 篇 owner 文档**，分 **10 个 Phase**，预计长周期推进。

---

## Phase 0：Git 基线锁定 + Crate 依赖图扫描

**目标**：不写正文。先建立工作基础——
- 确认 git 基线锚点（记录到 `_meta/git-tracking/git-branch-and-upstream-tracking.md`）
- 扫描 `codex-rs/Cargo.toml` workspace 成员，理清 99 个 crate 的依赖关系
- 定位关键入口文件（core/src/lib.rs、tui/src/lib.rs 等）
- 产出一张 crate 依赖概览图（存入 `_tmp_tracking/_digested_isuues/` 作为后续各 Phase 的参考）

**产出**：
- 更新 `_meta/git-tracking/git-branch-and-upstream-tracking.md` 的当前快照
- `_tmp_tracking/_digested_isuues/2026-05-03-phase0-crate-dependency-scan.md`

**不产出**：任何 `_digested/` 主题正文。

---

## Phase 1：02-system-architecture（系统架构全景）

**目标**：建立 Codex 的系统全景心智模型，这是所有后续 Phase 的"地图"。

**关键 crate**：
- `codex-rs/Cargo.toml`（workspace 定义）
- `codex-rs/core/`、`codex-rs/core-api/`
- `codex-rs/config/`
- `codex-cli/`（Node.js 前端）

**产出文档**：

### 1.1 `02-system-architecture/01-system-landscape.md`
- 技术栈：Rust + Bazel + Cargo + Node.js CLI
- Crate 地图：~99 crate 按 9 个功能域分组
- 进程模型：CLI frontend ↔ Rust backend 的关系
- 主子系统地图（mermaid）
- 系统约束（prompt caching、sandboxing、cross-platform）
- 源码抓手（关键 crate 入口文件）

### 1.2 `02-system-architecture/02-information-flow-and-boundaries.md`
- 数据面 vs 控制面
- 从用户输入到 agent 响应的完整信息流
- 持久化写入点
- 模块边界与跨边界通信方式
- 源码抓手

**完成标准**：
- [ ] Cargo workspace 成员全部归类
- [ ] 至少 1 张 mermaid 全景图 + 1 张信息流图
- [ ] 所有 crate 分组有对应的源码路径引用

---

## Phase 2：03-agent-core（Agent 核心）

**目标**：把 Agent 的心脏——循环、生命周期、工具分发——讲清楚。

**关键 crate**：
- `codex-rs/core/`（主循环、对话生命周期）
- `codex-rs/core-api/`（核心 API 类型）
- `codex-rs/agent-identity/`（身份定义）
- `codex-rs/agent-graph-store/`（图存储）

**产出文档**：

### 2.1 `03-agent-core/01-agent-loop-and-lifecycle.md`
- Agent 实例的组成（宿主注入 vs agent-owned vs tool-owned）
- `run_conversation()` 主循环（抽象版 + 源码锚点）
- 工具调用的截获与分发（三类路由：agent-loop / registry / dynamic）
- Persistence：JSON + SQLite 双写
- Teardown：release vs shutdown vs close 三种语义
- 源码抓手 + tests 抓手

### 2.2 `03-agent-core/02-tool-dispatch-and-execution.md`
- Tool schema 的三条来源路径
- `_AGENT_LOOP_TOOLS` 概念（哪些工具必须由 agent loop 截获）
- Dispatch 链：registry → dispatch → handler
- 可用性过滤（三层 gate）
- 源码抓手 + tests 抓手

### 2.3 `03-agent-core/03-agent-identity-and-system-prompt.md`
- Identity 来源（SOUL.md / AGENTS.md / DEFAULT_AGENT_IDENTITY）
- System prompt 装配层次（identity → tool guidance → memory → skills → context files）
- 冻结快照机制（prompt caching 约束）
- Invalidation 触发条件
- 源码抓手

### 2.4 `03-agent-core/04-context-and-compression.md`
- Context window 管理
- Compression 触发时机与策略
- 压缩失败的回退行为
- Prompt caching 与 cache busting
- 源码抓手 + tests 抓手

**完成标准**：
- [ ] core crate 的关键源文件全部通读
- [ ] 至少有 3 个关键函数的源码路径引用
- [ ] Agent 生命周期从 init 到 teardown 的完整链路可追溯

---

## Phase 3：04-tools-execution-and-sandboxing（工具与执行）

**目标**：覆盖 Codex 的能力执行面——工具注册/分发、命令执行、沙箱隔离。

**关键 crate**：
- `codex-rs/tools/`（工具实现与注册）
- `codex-rs/exec/`、`codex-rs/exec-server/`（命令执行）
- `codex-rs/sandboxing/`、`codex-rs/linux-sandbox/`、`codex-rs/windows-sandbox-rs/`（沙箱）
- `codex-rs/process-hardening/`（进程加固）
- `codex-rs/shell-command/`、`codex-rs/shell-escalation/`（shell）

**产出文档**：

### 3.1 `04-tools-execution-and-sandboxing/01-tool-registry-and-dispatch.md`
- 工具注册机制（类似 hermes 的 `registry.register()`）
- Tool discovery（built-in + plugin + MCP）
- Toolset 概念与 platform toolsets
- Schema 生成与 check_fn 运行时过滤
- Dispatch 链路

### 3.2 `04-tools-execution-and-sandboxing/02-exec-and-shell.md`
- Exec 工具：命令执行流程
- Shell 集成与权限提升
- Terminal 环境管理（cwd、env、task_id）
- 后台进程管理

### 3.3 `04-tools-execution-and-sandboxing/03-sandboxing.md`
- Sandbox 抽象层
- Linux Seatbelt 实现
- Windows Sandbox 实现
- 进程加固机制
- Sandbox 边界与逃逸防护

### 3.4 `04-tools-execution-and-sandboxing/04-file-operations.md`
- 文件读写工具的并发控制
- 原子写入与符号链接处理
- 文件锁机制

### 3.5 `04-tools-execution-and-sandboxing/05-background-processes.md`
- 后台进程生命周期
- 通知与监控（notify_on_complete、watch_patterns）
- 进程注册表与清理

**完成标准**：
- [ ] tools/ 下所有工具模块至少识别并归类
- [ ] sandbox 三种后端（Linux/Windows/generic）的差异有明确说明
- [ ] exec 链路从 tool call 到 subprocess spawn 可追溯

---

## Phase 4：06-models-and-providers（模型与提供商）

**目标**：覆盖 Codex 的模型抽象层——如何统一接入多种 LLM 提供商。

**关键 crate**：
- `codex-rs/model-provider/`（核心抽象）
- `codex-rs/model-provider-info/`（模型元信息）
- `codex-rs/models-manager/`（模型管理）
- `codex-rs/lmstudio/`、`codex-rs/ollama/`、`codex-rs/chatgpt/`（具体适配）
- `codex-rs/backend-client/`、`codex-rs/codex-api/`、`codex-rs/codex-client/`（Codex 自有后端）

**产出文档**：

### 4.1 `06-models-and-providers/01-model-provider-abstraction.md`
- Provider trait 定义
- Transport 层（chat completions / codex responses / anthropic messages）
- Provider 适配器（OpenAI / Anthropic / Ollama / LM Studio / ChatGPT）
- Credential pool 与 key 管理

### 4.2 `06-models-and-providers/02-model-routing-and-selection.md`
- 主模型 vs auxiliary 模型
- Model catalog（静态 fallback + 远端 curated manifest）
- Model switch 逻辑
- Fallback chain（primary → openrouter → nous → custom → direct）

### 4.3 `06-models-and-providers/03-api-adapters.md`
- 各家 provider 的具体适配差异
- Azure Foundry / GMI / Tencent TokenHub 等特殊 provider
- Image routing（vision model 选择）
- Reasoning content 隔离

**完成标准**：
- [ ] model-provider trait 的完整接口定义可引用
- [ ] 至少 3 家 provider 的适配路径可追溯
- [ ] auxiliary model 解析优先级有明确文档

---

## Phase 5：05-skills-and-plugins（技能与插件）

**目标**：覆盖 Codex 的可编程扩展面——技能系统和插件机制。

**关键 crate**：
- `codex-rs/skills/`、`codex-rs/core-skills/`
- `codex-rs/plugin/`、`codex-rs/core-plugins/`

**产出文档**：

### 5.1 `05-skills-and-plugins/01-skill-system.md`
- Skill 定义格式（SKILL.md / front matter）
- Skill 加载与发现
- Skill 执行与上下文注入
- Skill 生命周期（load / reload / unload）
- Curator 后台维护链（如有）

### 5.2 `05-skills-and-plugins/02-plugin-mechanism.md`
- Plugin trait 定义
- Plugin 注册与生命周期
- Plugin context 与扩展点
- Plugin 发现（built-in + external）

### 5.3 `05-skills-and-plugins/03-skill-authoring.md`
- SKILL.md 编写规范
- Skill class / umbrella 概念
- Skill 测试与调试

**完成标准**：
- [ ] Skill 从定义到执行的完整链路可追溯
- [ ] Plugin 注册与发现机制有源码锚点
- [ ] 与 agent core 的 tool surface 扩展关系已说明

---

## Phase 6：07-state-memory-and-sessions（状态与记忆）

**目标**：覆盖 Codex 的数据持久化面。

**关键 crate**：
- `codex-rs/state/`（状态管理）
- `codex-rs/memories/`（记忆系统）
- `codex-rs/thread-store/`（会话/线程存储）
- `codex-rs/agent-graph-store/`（图存储）

**产出文档**：

### 6.1 `07-state-memory-and-sessions/01-state-and-thread-store.md`
- Thread 生命周期（创建 → 活跃 → 压缩 → 结束）
- State 持久化机制（SQLite / JSON / 其他）
- Session 检索与恢复（/resume 语义）
- Schema migration

### 6.2 `07-state-memory-and-sessions/02-memory-system.md`
- 内置 memory（MEMORY.md / USER.md / 冻结快照）
- 外部 memory provider 接口
- Memory 写入时机（session 边界 vs turn 边界）
- Memory 上下文注入（system prompt 中的位置）

**完成标准**：
- [ ] Thread 从创建到结束的完整生命周期有文档
- [ ] Memory 的冻结快照语义与刷新时机有明确说明
- [ ] 至少 1 个外部 memory provider 的接口有引用

---

## Phase 7：08-tui-cli-and-integration（交互面与集成）

**目标**：覆盖 Codex 的所有对外接口面。

**关键 crate**：
- `codex-rs/tui/`（TUI）
- `codex-rs/cli/`、`codex-cli/`（CLI）
- `codex-rs/app-server/`、`codex-rs/app-server-protocol/`、`codex-rs/app-server-transport/`（HTTP API）
- `codex-rs/codex-mcp/`、`codex-rs/mcp-server/`、`codex-rs/rmcp-client/`（MCP）
- `codex-rs/realtime-webrtc/`（实时通信）
- `codex-rs/responses-api-proxy/`（Responses API）
- `codex-rs/connectors/`（外部连接器）

**产出文档**：

### 7.1 `08-tui-cli-and-integration/01-tui-architecture.md`
- TUI 渲染管线
- 事件循环与输入处理
- 与 agent core 的桥接
- TUI session 生命周期（create / resume / close / delete）
- JSON-RPC transport 层

### 7.2 `08-tui-cli-and-integration/02-cli-and-app-server.md`
- CLI 入口与命令分发（slash commands）
- App-server HTTP API 路由
- 配置加载链
- 错误处理链

### 7.3 `08-tui-cli-and-integration/03-mcp-and-external-integration.md`
- MCP 协议交互（initialize、tools/list、tools/call）
- MCP server 启动与生命周期
- WebRTC 实时通信
- Responses API 代理
- 外部 connector 机制

**完成标准**：
- [ ] 至少 3 个入口面的启动链可追溯
- [ ] MCP 协议交互序列有文档
- [ ] TUI 的 JSON-RPC method/event catalog 至少列出关键方法

---

## Phase 8：01-beginner（入门文档）

**目标**：在所有机制文档完成后，写面向新用户的入门文档。这些文档依赖前面所有 Phase 的知识。

**产出文档**：

### 8.1 `01-beginner/01-quickstart.md`
- 5 分钟跑通 Codex
- 验证检查表（命令 + 成功信号）
- Mermaid 流程图
- 高频断点速查

### 8.2 `01-beginner/02-config-and-profiles.md`
- 安装、配置、Profile、CODEX_HOME
- 配置优先级（env > .env > config.yaml > defaults）

### 8.3 `01-beginner/03-models-and-providers.md`
- 模型选择、API key 配置
- Provider 基本概念
- 常见模型问题排查

### 8.4 `01-beginner/04-tools-and-availability.md`
- 工具是什么、怎么开关
- Toolset 概念（面向用户）
- "为什么工具不见了"的排查流程

### 8.5 `01-beginner/05-skills-basics.md`
- Skill 入门：是什么、怎么用、怎么创作第一个 skill

### 8.6 `01-beginner/06-logs-and-troubleshooting.md`
- 日志位置与分类
- 1 分钟诊断卡
- 高频问题 → 先查哪里 → 深层文档

**完成标准**：
- [ ] 每篇有至少 1 张验证检查表
- [ ] 每篇有术语中英文对照（严格模式：每次出现都写）
- [ ] 每篇有止步提示或下一步阅读指引
- [ ] 排错篇有完整的症状→抓手分流表

---

## Phase 9：交叉复核 + 一致性收尾

**目标**：全部正文写完后，回头检查一致性。

- [ ] 所有 `owns` / `update_when` / `out_of_scope` 是否仍然准确？
- [ ] 跨文档交叉引用是否断裂？
- [ ] 术语中英文对照是否在所有文档中一致？
- [ ] 源码路径是否仍然有效？（可能存在 Phase 0-8 期间的 upstream 变化）
- [ ] 是否需要做一次 upstream sync 复核？（如果 main 在 Phase 0-8 期间有显著变化）
- [ ] `_digested/README.md` 的文档清单是否与实际文件一致？

---

## 每个 Phase 的执行协议

每个 Phase 内的每个切片（1 篇文档）都遵循：

1. **读源码**：通读目标 crate 的关键源文件，记录关键函数/结构体/trait
2. **读 tests**：找到对应的测试文件，理解测试覆盖的行为边界
3. **写正文**：按 `WRITING-STYLE.md` 的标准骨架写 owner 文档
4. **自检**：通过 11 条信息密度自检清单（`WRITING-STYLE.md` 第十三节）
5. **记录**：在 `_meta/git-tracking/upstream-sync/` 下追加该切片的：
   - 已复核源码路径
   - 已复核 tests
   - 确认并写回的事实（bullet list）
   - 已更新文档
   - 未覆盖风险 / 下一步

### 一个切片的典型大小

- 源码阅读：1-3 个 crate，3-10 个关键源文件
- 文档产出：1 篇，约 150-400 行
- 不应跨 Phase 做切片（例如同时写 agent-core 和 tools 的文档）

---

## 当前状态

- [ ] Phase 0：Git 基线 + crate 扫描
- [ ] Phase 1：02-system-architecture（2 篇）
- [ ] Phase 2：03-agent-core（4 篇）
- [ ] Phase 3：04-tools-execution-sandboxing（5 篇）
- [ ] Phase 4：06-models-and-providers（3 篇）
- [ ] Phase 5：05-skills-and-plugins（3 篇）
- [ ] Phase 6：07-state-memory-and-sessions（2 篇）
- [ ] Phase 7：08-tui-cli-and-integration（3 篇）
- [ ] Phase 8：01-beginner（6 篇）
- [ ] Phase 9：交叉复核 + 收尾

**下一个切片**：Phase 0 — Git 基线锁定 + Crate 依赖图扫描

---

## 备注

- Phase 4（models）排在 Phase 5（skills）之前，因为 agent core 调用模型是更底层的依赖；skills 是建立在 agent + models 之上的扩展层。
- Phase 8（beginner）排在最后，因为入门文档需要引用所有其他主题的知识，最后写才能保证准确。
- 如果某个 Phase 涉及的 crate 量太大（如 Phase 3 涉及 8+ crate），可以在 Phase 内再拆子切片。
