---
title: "01 — 技能系统（Skill System）"
doc_type: "owner"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解 Codex 技能如何定义、加载、触发和注入到 prompt 的人"
purpose: "给出技能系统的完整架构：SKILL.md 格式、6 个 skill root 来源、BFS 发现与解析、4 阶段触发流程、2 级上下文注入、以及预算管理"
owns: "技能系统面：SkillMetadata 定义、SkillsManager 加载、技能触发（显式/隐式）、系统 prompt 注入、预算管理"
update_when:
  - "SKILL.md 格式或 frontmatter 字段变化时"
  - "skill root 来源或加载策略变化时"
  - "技能注入或预算管理策略变化时"
out_of_scope:
  - "插件技能加载（见 02-plugin-mechanism.md）"
  - "SKILL.md 编写规范（见 03-skill-authoring.md）"
---

# 01 — 技能系统（Skill System）

> [!IMPORTANT]
> 这一篇解释 Codex 的技能系统如何把"一段 Markdown 指令文件"变成"模型可以调用的专项能力"。

> 读完这篇你应该能回答：SKILL.md 怎么被发现和解析、技能怎么被触发（显式/隐式）、技能内容怎么注入到 prompt 的哪个位置、预算不够时怎么裁剪。

## 1) 先记住三件事

1. **技能是声明式的 Markdown 文件**：没有 trait、没有注册代码。`SKILL.md` + 可选 `agents/openai.yaml` 放在一个目录里就是一个技能。
2. **两级上下文注入**：可用技能列表注入到 system prompt（developer 角色），被触发的技能正文注入到对话历史（user 角色）。
3. **两种触发方式**：显式（用户用 `$skill-name` 或 UI 选择）和隐式（命令执行了技能 `scripts/` 下的脚本或读取了 SKILL.md）。

## 2) 架构总览

```
SkillsManager (core-skills/src/manager.rs)
  │
  ├── install_system_skills() (skills crate)
  │     └── 嵌入的 5 个 system skill 写入 $CODEX_HOME/skills/.system/
  │
  ├── skill_roots_from_layer_stack() (loader.rs:257)
  │     └── 6 个来源: Repo / User / System / Admin / Plugin / Cwd
  │
  ├── discover_skills_under_root() (loader.rs:440)
  │     └── BFS 扫描 (max depth 6) → parse_skill_file() → SkillMetadata
  │
  ├── build_available_skills() (render.rs:160)
  │     └── 渲染技能列表 → AvailableSkillsInstructions → system prompt
  │
  ├── collect_explicit_skill_mentions() (injection.rs:114)
  │     └── 检测 $skill-name / UserInput::Skill → SkillInjection
  │
  └── detect_implicit_skill_invocation_for_command() (invocation_utils.rs:29)
        └── 检测 scripts/ 执行 / SKILL.md 读取 → 隐式触发
```

## 3) 技能定义格式

### SKILL.md（必需）

```yaml
---
name: "skill-name"           # 必需，字母数字+连字符，最长 64 字符
description: "..."           # 必需，最长 1024 字符，模型用来判断是否触发
metadata:
  short-description: "..."   # 可选，最长 1024 字符
---
(Markdown 正文 — 仅在触发后注入给模型，建议 500 行以内)
```

- **name** 缺失时回退到父目录名
- 如果技能在插件路径下，自动添加 `namespace:name` 前缀

### agents/openai.yaml（推荐）

```yaml
interface:
  display_name: ""
  short_description: ""
  icon_small: ""           # assets/ 下相对路径
  icon_large: ""
  brand_color: "#RRGGBB"
  default_prompt: ""       # 最长 1024 字符
dependencies:
  tools:
    - type: "env_var"
      value: "VAR_NAME"
      description: "..."
policy:
  allow_implicit_invocation: true/false  # 是否允许隐式触发
  products: [Codex, Chatgpt]             # 限定产品面
```

### 目录结构

```
skill-name/
  SKILL.md              (必需 — 技能定义)
  agents/openai.yaml    (推荐 — UI 元数据 + 依赖 + 策略)
  scripts/              (可选 — 可执行代码)
  references/           (可选 — 额外参考文档)
  assets/               (可选 — 图标等)
```

## 4) SkillScope：四种作用域

`protocol/src/protocol.rs:3365`：

```rust
pub enum SkillScope { User, Repo, System, Admin }
```

优先级（高→低）：Repo > User > System > Admin。

| 作用域 | 来源目录 | 说明 |
|--------|---------|------|
| Repo | `$CONFIG_FOLDER/skills` + `.agents/skills` 从 cwd 向上 | 项目级别 |
| User | `$CODEX_HOME/skills` (旧) + `$HOME/.agents/skills` | 用户安装 |
| System | `$CODEX_HOME/skills/.system` | 嵌入的 5 个系统技能 |
| Admin | `/etc/codex/skills` (仅 Unix) | 系统管理员 |
| Plugin | 插件清单 `paths.skills` | 见 02-plugin-mechanism |

## 5) 技能加载

### 目录扫描

`discover_skills_under_root()` (`loader.rs:440`)：
- BFS 遍历，最大深度 6 层（`MAX_SCAN_DEPTH`）
- 每个 root 最多 2000 个目录（`MAX_SKILLS_DIRS_PER_ROOT`）
- Repo/User/Admin 作用域跟随符号链接，System 不跟随
- 忽略 `.` 开头的目录
- 发现 `SKILL.md` → 调用 `parse_skill_file()`

### 文件解析

`parse_skill_file()` (`loader.rs:582`)：
1. 读取文件内容
2. 提取 `---` 分隔的 YAML frontmatter
3. 解析 `name`、`description`、`metadata.short-description`
4. 如果 `name` 缺失 → 回退到父目录名
5. 如果技能在插件路径 → 添加 `namespace:name` 前缀（`plugin_namespace_for_skill_path`）
6. 验证长度限制（name 64、description 1024 等）
7. 加载 `agents/openai.yaml`（`load_skill_metadata`）

### 缓存

`SkillsManager` 有两个缓存：
- **by_cwd** — 按工作目录缓存（目录不同时重新加载）
- **by_config** — 按配置版本缓存（配置不变时复用）

文件监控器（`skills_watcher.rs`）在检测到变更时清除缓存（10s 节流）。

## 6) 技能触发：四阶段流程

### Stage A：可用技能列表（每次 turn 都执行）

`build_available_skills()` (`render.rs:160`)：
1. 从 `SkillLoadOutcome` 过滤允许的技能
2. 按 prompt 优先级排序：System > Admin > Repo > User
3. 渲染为 markdown 列表
4. 包装为 `AvailableSkillsInstructions`（developer 角色）
5. 注入到 base prompt 的 developer_sections

### Stage B：显式触发检测

`collect_explicit_skill_mentions()` (`injection.rs:114`)：
- **UI 选择**：`UserInput::Skill { name, path }` 结构化选择
- **文本触发**：`$skill-name` 语法（`$` = `TOOL_MENTION_SIGIL`）
- **链接触发**：`[$tool-name](resource path)` markdown 链接
- 去重：只有无歧义的纯名称匹配才被选中（`skill_name_counts == 1`）
- 排除 disabled paths 中的技能

### Stage C：技能正文注入

`build_skill_injections()` (`injection.rs:31`)：
1. 读取每个被触发技能的 `SKILL.md` 内容
2. 包装为 `SkillInjection { name, path, contents }`
3. 通过 `SkillInstructions`（user 角色）注入到对话历史
4. 渲染格式：`<skill>\n<name>...</name>\n<path>...</path>\n<contents>\n</skill>`
5. 记录 analytics 和 telemetry

### Stage D：隐式触发

`detect_implicit_skill_invocation_for_command()` (`invocation_utils.rs:29`)：
- **脚本执行**：`python/bash/node/...` 命令执行了技能 `scripts/` 下的文件 → 隐式触发
- **文档读取**：`cat/sed/head/tail/...` 命令读取了技能 `SKILL.md` → 隐式触发
- 受 `policy.allow_implicit_invocation` 控制
- 每 turn 去重

## 7) 预算管理

`SkillMetadataBudget` (`render.rs:86`)：
- 默认：上下文窗口的 2% 或 8000 字符
- 三级裁剪策略：
  1. **完整模式**：全部技能完整描述放得下
  2. **截断模式**：等比例截断每个技能的描述
  3. **省略模式**：跳过整个技能，只保留最小行数
- 截断或省略时产生 warning

两种路径渲染策略：
- **绝对路径**：直接使用技能路径（短路径时效率高）
- **别名路径**：当路径长（如插件缓存路径），用 `r0`、`r1` 等短别名

## 8) 内嵌系统技能

`skills/src/lib.rs` — `install_system_skills()` 在启动时将 `include_dir!` 嵌入的 5 个技能写入 `$CODEX_HOME/skills/.system/`：

| 技能 | 用途 |
|------|------|
| `skill-creator` | 创建新技能的指南 |
| `plugin-creator` | 创建插件的指南 |
| `skill-installer` | 从 GitHub 安装技能 |
| `openai-docs` | OpenAI 文档访问 |
| `imagegen` | 图像生成 |

使用指纹 marker 文件（`.codex-system-skills.marker`）跳过冗余的重复安装。

## 9) 配置规则

`core-skills/src/config_rules.rs` — 用户可通过 `skills.config` 选择性启用/禁用技能：
- 规则类型：`Name(String)` 或 `Path(AbsolutePathBuf)` + `enabled: bool`
- 仅 User 和 SessionFlags 配置层可用
- 后续规则覆盖前一个同名选择器

## 10) 源码抓手

| 想看什么 | 从哪里开始 |
|----------|-----------|
| SkillsManager | `core-skills/src/manager.rs:50` |
| SkillMetadata 定义 | `core-skills/src/model.rs:12` |
| SkillScope 枚举 | `protocol/src/protocol.rs:3365` |
| SkillLoadOutcome | `core-skills/src/model.rs:87` |
| skill_roots_from_layer_stack | `core-skills/src/loader.rs:257` |
| discover_skills_under_root | `core-skills/src/loader.rs:440` |
| parse_skill_file | `core-skills/src/loader.rs:582` |
| load_skill_metadata (openai.yaml) | `core-skills/src/loader.rs:668` |
| collect_explicit_skill_mentions | `core-skills/src/injection.rs:114` |
| build_skill_injections | `core-skills/src/injection.rs:31` |
| detect_implicit_skill_invocation_for_command | `core-skills/src/invocation_utils.rs:29` |
| build_available_skills | `core-skills/src/render.rs:160` |
| SkillMetadataBudget | `core-skills/src/render.rs:86` |
| SkillInstructions (context) | `core/src/context/skill_instructions.rs:7` |
| AvailableSkillsInstructions | `core/src/context/available_skills_instructions.rs:9` |
| SkillsWatcher (文件监控) | `core/src/skills_watcher.rs:32` |
| install_system_skills | `skills/src/lib.rs:32` |
| 技能配置规则 | `core-skills/src/config_rules.rs:25` |
| 环境变量依赖 | `core-skills/src/env_var_dependencies.rs` |
| Remote skills API | `core-skills/src/remote.rs` |

## 11) 相关文档

- 插件机制：`02-plugin-mechanism.md`
- 技能创作：`03-skill-authoring.md`
- Agent 身份与系统 prompt：`../03-agent-core/03-agent-identity-and-system-prompt.md`

---

> 读完这篇，你应该能追踪一个 `SKILL.md` 文件从 BFS 发现 → frontmatter 解析 → `SkillMetadata` → 可用列表注入 system prompt → 用户 `$skill-name` 触发 → 正文注入对话历史的完整链路。
