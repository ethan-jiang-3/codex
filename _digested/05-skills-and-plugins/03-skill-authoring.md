---
title: "03 — 技能创作指南（Skill Authoring）"
doc_type: "owner"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要编写和调试 SKILL.md 的人"
purpose: "给出 SKILL.md 和 openai.yaml 的完整字段规范、验证约束、目录布局、以及从 skill-creator 系统技能中提取的最佳实践"
owns: "技能创作规范：SKILL.md frontmatter 字段、openai.yaml 结构、目录约定、验证规则"
update_when:
  - "SKILL.md frontmatter 字段或验证规则变化时"
  - "openai.yaml 新增字段或语义变化时"
  - "skill-creator 系统技能更新时"
out_of_scope:
  - "技能系统的内部加载机制（见 01-skill-system.md）"
  - "插件打包（见 02-plugin-mechanism.md）"
---

# 03 — 技能创作指南（Skill Authoring）

> [!IMPORTANT]
> 这一篇是技能编写规范的参考手册——定义了 `SKILL.md` 和 `agents/openai.yaml` 中每个字段的含义、验证约束、以及目录组织的最佳实践。

> 读完这篇你应该能回答：一个合法的技能至少需要什么、`openai.yaml` 里每个字段的校验上限是什么、技能目录怎么组织、以及 `skill-creator` 系统技能给出了哪些编写指南。

## 1) 先记住三件事

1. **最小的合法技能**：一个目录 + 目录里的 `SKILL.md` 文件 + frontmatter 里有 `name` 和 `description`。仅此而已。
2. **description 是触发机制**：模型根据 description 判断"什么时候该用这个技能"。description 写得好坏直接决定技能被触发的概率。
3. **正文 500 行以内**：`skill-creator` 推荐 SKILL.md 正文控制在 500 行以内。超出部分放到 `references/` 目录，按需加载。

## 2) SKILL.md 格式规范

### 完整结构

```markdown
---
name: "my-skill"
description: "当用户需要做 X 时使用此技能。它提供了 Y 和 Z。"
metadata:
  short-description: "简短描述，用于 UI 展示"
---

(正文：Markdown 格式的技能指令，仅在技能被触发时注入给模型)
```

### Frontmatter 字段

| 字段 | 必需 | 类型 | 最长 | 说明 |
|------|------|------|------|------|
| `name` | 是 | `String` | 64 字符 | 字母数字 + 连字符。缺失时回退到父目录名 |
| `description` | 是 | `String` | 1024 字符 | 模型用来判断是否触发该技能。这是最主要的触发机制 |
| `metadata.short-description` | 否 | `String` | 1024 字符 | UI 展示用的简短描述 |

### 正文规范

- 格式：Markdown
- 推荐上限：500 行
- 注入时机：仅当技能被显式或隐式触发时，正文才会作为对话内容注入给模型
- 超出 500 行的内容 → 放到 `references/` 目录，在正文中用引用方式指向

## 3) agents/openai.yaml 格式规范

```yaml
interface:
  display_name: "My Skill"       # UI 显示名称
  short_description: "..."       # UI 简短描述
  icon_small: "icon-small.png"   # assets/ 下相对路径
  icon_large: "icon-large.png"   # assets/ 下相对路径
  brand_color: "#FF5733"         # 品牌色
  default_prompt: "..."          # 默认提示词

dependencies:
  tools:
    - type: "env_var"            # 依赖类型
      value: "MY_API_KEY"        # 环境变量名
      description: "用于访问 My API"  # 说明

policy:
  allow_implicit_invocation: true   # 是否允许隐式触发
  products: [Codex, Chatgpt]        # 限定产品面
```

### interface 字段（全部可选）

| 字段 | 类型 | 最长 | 说明 |
|------|------|------|------|
| `display_name` | String | — | UI 中的显示名称 |
| `short_description` | String | — | UI 中的简短描述 |
| `icon_small` | Path | — | 小图标，必须是 `assets/` 下的相对路径 |
| `icon_large` | Path | — | 大图标，必须是 `assets/` 下的相对路径 |
| `brand_color` | String | — | 品牌色，`#RRGGBB` 格式 |
| `default_prompt` | String | 1024 字符 | 技能专用的默认提示词 |

### dependencies.tools 字段

| 字段 | 类型 | 最长 | 说明 |
|------|------|------|------|
| `type` | String | — | 依赖类型。目前主要是 `"env_var"` |
| `value` | String | 64 字符 | 环境变量名 |
| `description` | String | 1024 字符 | 说明为什么需要这个环境变量 |

当 `Feature::SkillEnvVarDependencyPrompt` 启用时，未解析的环境变量会触发交互式提示，询问用户输入值。

### policy 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `allow_implicit_invocation` | `bool` | 是否允许隐式触发（通过执行 scripts/ 下的脚本或读取 SKILL.md 自动触发） |
| `products` | `Vec<Product>` | 限定技能可见的产品面。如不指定则对所有产品可见 |

## 4) 目录结构约定

```
my-skill/
├── SKILL.md              ★ 必需 — 技能定义文件
├── agents/
│   └── openai.yaml       ☆ 推荐 — UI 元数据、依赖声明、安全策略
├── scripts/              (可选) — 可执行脚本，支持隐式触发
├── references/           (可选) — 额外参考文档，按需加载
└── assets/               (可选) — 图标、模板等静态资源
```

### 关于 references/

当 SKILL.md 正文超过 500 行时：
- 将详细文档放到 `references/` 目录
- 在 SKILL.md 正文中用 `[link](references/doc.md)` 引用
- 模型可以在需要时通过工具读取 reference 文件

### 关于 scripts/

- 脚本在模型执行到它们时被隐式检测
- 当模型运行的命令（`python`、`bash`、`node` 等）执行了 `scripts/` 下的文件 → 自动触发技能
- 需要 `policy.allow_implicit_invocation = true`

## 5) 验证约束汇总

所有约束都在 `core-skills/src/loader.rs` 中实施：

| 约束项 | 值 | 说明 |
|--------|-----|------|
| name 最大长度 | 64 字符 | 字母数字 + 连字符 |
| description 最大长度 | 1024 字符 | 触发匹配的主要来源 |
| short_description 最大长度 | 1024 字符 | UI 展示 |
| default_prompt 最大长度 | 1024 字符 | 接口元数据 |
| dependency value 最大长度 | 64 字符 | 环境变量名 |
| dependency description 最大长度 | 1024 字符 | 依赖说明 |
| 图标路径 | `assets/` 下相对路径 | 不允许 `../` 逃逸 |
| SKILL.md 文件存在性 | 必需 | 缺失时该目录不被识别为技能 |
| 扫描深度 | 最大 6 层 | BFS 从 root 向下 |
| 每 root 最大目录数 | 2000 | 防止扫描爆炸 |

## 6) skill-creator 系统技能指南

`skill-creator` 是嵌入在 `skills/src/assets/samples/skill-creator/` 中的系统技能。它给出了以下编写规范：

### 核心原则

1. **description 是关键**：模型完全靠 description 判断"什么时候该加载这个技能"。写清楚"当用户需要做什么时使用此技能"，而非"此技能是什么"。
2. **正文即 prompt**：SKILL.md 正文直接作为模型指令注入。写成像在和一个聪明的同事说话——简洁、指令性、有例子。
3. **references 用于长文档**：正文 <= 500 行。API 文档、完整 schema、长代码示例 → `references/`。

### 推荐的正文结构

1. 技能用途的一段概述
2. 何时使用（触发条件）
3. 核心工作流或步骤
4. 关键约束或警告
5. 示例（如有必要）

### 多技能组织

- 一个插件可以包含多个技能（每个技能是一个子目录）
- 用 `policy.products` 限定技能适用产品
- 相关技能可分到同一个插件里
- 技能命名应描述其功能：`deploy-to-vercel` 优于 `deploy`

## 7) 调试技能

### 检查技能是否被加载

1. 确认目录在 skill root 下（Repo/User/System/Admin/Plugin）
2. 检查 `SKILL.md` 的 frontmatter 是否有 `name` 和 `description`
3. 检查 config 中是否有禁用该技能的规则
4. 检查 `policy.products` 是否包含当前产品
5. 检查扫描深度（目录层级不超过 6）

### 检查技能是否被触发

- **显式触发**：在对话中用 `$skill-name` 提及技能
- **隐式触发**：确保 `policy.allow_implicit_invocation = true`，然后执行 scripts 下的脚本
- 技能注入会出现在对话中，以 `<skill>` 标签包裹

### 文件变更监控

`SkillsWatcher` 每 10 秒检测文件变更并自动清除缓存。修改 `SKILL.md` 后等待 10 秒即可在下一次 turn 加载新内容。

## 8) 源码抓手

| 想看什么 | 从哪里开始 |
|----------|-----------|
| SKILL.md 解析 | `core-skills/src/loader.rs:582` — `parse_skill_file()` |
| openai.yaml 加载 | `core-skills/src/loader.rs:668` — `load_skill_metadata()` |
| 验证常量 | `core-skills/src/loader.rs:105-122` |
| SkillMetadata 完整字段 | `core-skills/src/model.rs:12` |
| SkillInterface | `core-skills/src/model.rs:56` |
| SkillDependencies | `core-skills/src/model.rs:66` |
| SkillPolicy | `core-skills/src/model.rs:48` |
| skill-creator SKILL.md | `skills/src/assets/samples/skill-creator/SKILL.md` |
| 环境变量依赖 | `core-skills/src/env_var_dependencies.rs` |

## 9) 相关文档

- 技能系统：`01-skill-system.md`
- 插件机制：`02-plugin-mechanism.md`

---

> 读完这篇，你应该能写出一个合法的 `SKILL.md` + `agents/openai.yaml` 组合，理解每个字段的校验上限，并按照 skill-creator 的指南组织技能目录。
