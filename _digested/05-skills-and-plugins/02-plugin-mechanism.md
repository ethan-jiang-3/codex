---
title: "02 — 插件机制（Plugin Mechanism）"
doc_type: "owner"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解 Codex 插件如何定义、安装、加载和扩展系统能力的人"
purpose: "给出插件的声明式模型、PluginId 格式、plugin.json 清单、4 种扩展点、marketplace 发现与商店、以及完整的安装/加载/卸载生命周期"
owns: "插件系统面：PluginId、PluginManifest、PluginsManager 加载、marketplace 发现、PluginStore、扩展点（skills/MCP/apps/hooks）"
update_when:
  - "插件清单格式或扩展点变化时"
  - "marketplace 机制或商店策略变化时"
  - "插件安装/卸载流程变化时"
out_of_scope:
  - "技能系统内部机制（见 01-skill-system.md）"
  - "具体插件的业务实现"
---

# 02 — 插件机制（Plugin Mechanism）

> [!IMPORTANT]
> 这一篇解释 Codex 的插件系统——插件不是 Rust trait，而是声明式的文件集合（清单 + skills + MCP + apps + hooks），通过 marketplace 分发，由 `PluginsManager` 统一加载。

> 读完这篇你应该能回答：插件怎么定义和分发、`PluginId` 的 `name@marketplace` 格式、4 种扩展点分别怎么加载、插件从安装到激活的完整生命周期。

## 1) 先记住三件事

1. **插件是声明式的，不是 trait**：没有 `Plugin` trait。一个插件 = `plugin.json` 清单 + 可选 `skills/`、`.mcp.json`、`.app.json`、`hooks/` 这些文件放在一起。
2. **插件 ID 是 `name@marketplace`**：`PluginId { plugin_name, marketplace_name }`，解析自 `"slack@openai-curated"` 格式。
3. **三种 marketplace 来源**：OpenAI 策展仓库（`openai-curated`，从 GitHub sync）、内置打包（`openai-bundled`，随 binary 发布）、用户自定义 marketplace（`config.toml` 中的 `[marketplaces]`）。

## 2) 核心类型

### PluginId

`plugin/src/plugin_id.rs:8`：

```rust
pub struct PluginId {
    pub plugin_name: String,       // 字母数字 + 连字符
    pub marketplace_name: String,  // 同上
}
```

- `PluginId::parse("name@marketplace")` — 从 `@` 分隔的字符串解析
- `PluginId::as_key()` — 格式化回 `"name@marketplace"`
- `validate_plugin_segment()` — 只允许 ASCII 字母数字 + `-` + `_`

### PluginManifest

`core-plugins/src/manifest.rs:36` — 从 `.codex-plugin/plugin.json`（或 `.claude-plugin/plugin.json`）加载：

```rust
pub struct PluginManifest {
    pub name: String,
    pub version: Option<String>,
    pub description: Option<String>,
    pub paths: PluginManifestPaths,
    pub interface: Option<PluginManifestInterface>,
}
```

**PluginManifestPaths** (line 44)：

| 字段 | 默认路径 | 用途 |
|------|---------|------|
| `skills` | `./skills/` | 技能目录 |
| `mcp_servers` | `./.mcp.json` | MCP 服务器配置 |
| `apps` | `./.app.json` | App connector 配置 |
| `hooks` | `./hooks/hooks.json` | 钩子配置 |

**PluginManifestInterface** (line 58)：UI 展示元数据 — `display_name`、`short_description`、`long_description`、`developer_name`、`category`、`capabilities`、`website_url`、隐私/条款 URL、`default_prompt`、`brand_color`、图标、截图。

### LoadedPlugin

`plugin/src/load_outcome.rs:14`：

```rust
pub struct LoadedPlugin<M> {
    pub config_name: String,
    pub manifest_name: Option<String>,
    pub manifest_description: Option<String>,
    pub root: AbsolutePathBuf,
    pub enabled: bool,
    pub skill_roots: Vec<AbsolutePathBuf>,
    pub disabled_skill_paths: HashSet<AbsolutePathBuf>,
    pub has_enabled_skills: bool,
    pub mcp_servers: HashMap<String, M>,
    pub apps: Vec<AppConnectorId>,
    pub hook_sources: Vec<PluginHookSource>,
    pub hook_load_warnings: Vec<String>,
    pub error: Option<String>,
}
```

`is_active()` 当 `enabled && error.is_none()` 时返回 true。

## 3) 插件注册

插件通过 `config.toml` 注册（无动态注册 API）：

```toml
[plugins]
"plugin-name@marketplace-name" = { enabled = true }
```

这是唯一的配置机制。`PluginsConfigInput`（`manager.rs:83`）收集配置层栈 + feature flags（`Feature::Plugins`、`Feature::RemotePlugin`、`Feature::PluginHooks`）。

## 4) 插件加载流程

`PluginsManager::plugins_for_config()` (`manager.rs:457`) 是整个加载的入口：

```
1. 检查缓存（按 config version + hooks 开关）
2. load_plugins_from_layer_stack() (loader.rs:110)
   │
   ├── 读取 [plugins] 下所有条目
   │
   └── 对每个条目调用 load_plugin() (loader.rs:496)：
         │
         ├── a. 解析 PluginId
         ├── b. 从 PluginStore 解析已安装的 root
         ├── c. 加载 plugin.json → PluginManifest
         ├── d. 解析 skill roots（默认 skills/ 或 manifest paths.skills）
         ├── e. load_plugin_skills() — 加载技能（含产品限制）
         ├── f. load_plugin_mcp_servers() — 加载 .mcp.json
         │      └── 合并 per-plugin MCP server 策略（启用/禁用工具、审批模式）
         ├── g. load_plugin_apps() — 加载 .app.json
         └── h. load_plugin_hooks() — 加载 hooks（需 Feature::PluginHooks）
```

返回 `PluginLoadOutcome`，通过以下方法提取有效部分：
- `effective_skill_roots()` — 去重技能根路径
- `effective_mcp_servers()` — 合并 MCP server 映射
- `effective_apps()` — 去重 app connector ID
- `effective_plugin_hook_sources()` — 去重 hook 源

### 产物过滤

`PluginCapabilitySummary` (`plugin/src/lib.rs:22`) 提供遥测摘要：
```rust
pub struct PluginCapabilitySummary {
    pub config_name: String,
    pub display_name: String,
    pub description: String,
    pub has_skills: bool,
    pub mcp_server_names: Vec<String>,
    pub app_connector_ids: Vec<String>,
}
```

## 5) Marketplace 系统

### Marketplace 定义

Marketplace 是 `marketplace.json` 文件，包含插件目录和策略。

**MarketplacePluginSource** (`marketplace.rs:67`)：
```rust
pub enum MarketplacePluginSource {
    Local { path: AbsolutePathBuf },
    Git { url: String, path: Option<String>, ref_name: Option<String>, sha: Option<String> },
}
```

**MarketplacePluginPolicy** (line 80)：
```rust
pub struct MarketplacePluginPolicy {
    pub installation: MarketplacePluginInstallPolicy,  // NotAvailable | Available | InstalledByDefault
    pub authentication: MarketplacePluginAuthPolicy,    // OnInstall | OnUse
    pub products: Option<Vec<Product>>,
}
```

### 发现路径

Marketplace 从以下位置发现（`marketplace.rs:334`）：
1. `$HOME` 目录（直接检查或通过 git repo root）
2. 额外的配置根目录
3. `$CODEX_HOME/.tmp/plugins/`（策展仓库 sync 目标）

### 三种 marketplace

| Marketplace | 来源 | 存储位置 |
|-------------|------|---------|
| `openai-curated` | GitHub `openai/plugins.git` sync | `$CODEX_HOME/.tmp/plugins/` |
| `openai-bundled` | 随 binary 发布 | 嵌入 |
| 用户自定义 | `config.toml` 的 `[marketplaces]` | `$CODEX_HOME/.tmp/marketplaces/` |

## 6) PluginStore：磁盘缓存

`core-plugins/src/store.rs:25`：

目录布局：`$CODEX_HOME/plugins/cache/{marketplace_name}/{plugin_name}/{version}/`

- `active_plugin_version()` (line 68)：读取版本目录，优先 `"local"` 版本，否则选最高排序版本
- `install_with_version()` (line 109)：复制源码到 `{root}/{marketplace}/{plugin}/{version}`，验证 `plugin.json` name 匹配
- `uninstall()` (line 144)：删除 `{root}/{marketplace}/{plugin}/`
- 原子安装：复制到临时目录 → 备份已有安装 → 重命名就位

## 7) 启动同步

`core-plugins/src/startup_sync.rs` — 启动时同步 OpenAI 策展仓库：

1. `git clone --depth 1`（优先方式）
2. GitHub HTTP API（git 失败时的 fallback）
3. `chatgpt.com/backend-api/plugins/export/curated` 备份导出（最后手段）

同步到 `$CODEX_HOME/.tmp/plugins/`，SHA 保存到 `.tmp/plugins.sha`。

`maybe_start_plugin_startup_tasks_for_config()` (`manager.rs:1374`) 是启动入口：同步策展仓库 + 升级 Git marketplace + 启动远程插件 sync。

## 8) 插件安装与卸载

### 安装

`PluginsManager::install_plugin()` (`manager.rs:789`)：
1. 从 marketplace 解析 `MarketplacePluginSource`
2. 对于 `Git` 源：`git clone`（可能 sparse checkout）到临时目录
3. 对于 `Local` 源：直接使用路径
4. `PluginStore::install_with_version()` 复制到缓存
5. 写入用户 config：`[plugins]` 条目
6. 清除缓存以触发重新加载
7. 遥测：`track_plugin_installed`

### 卸载

`PluginsManager::uninstall_plugin()` (`manager.rs:884`)：
1. 删除 `$CODEX_HOME/plugins/cache/{marketplace}/{plugin}/`
2. `clear_user_plugin()` 清除 config 条目
3. 遥测：`track_plugin_uninstalled`

## 9) 四种扩展点

| 扩展点 | 配置文件 | 加载函数 | 贡献到 |
|--------|---------|---------|--------|
| Skills | `skills/*.md` | `load_plugin_skills` | SkillsManager → system prompt |
| MCP Servers | `.mcp.json` | `load_plugin_mcp_servers` | MCP 连接管理器 |
| Apps | `.app.json` | `load_plugin_apps` | Connector 系统、工具发现 |
| Hooks | `hooks/hooks.json` | `load_plugin_hooks` | Hooks 引擎 |

### 显式提及触发

当用户通过 `@plugin-name` 显式提及时，`build_plugin_injections()` (`core/src/plugins/injection.rs:15`) 构造 developer 消息，告诉模型该插件提供了哪些 MCP server 和 app。

## 10) 工具发现中的插件

`tool_search` 系统（`tools/src/tool_discovery.rs`）支持：
- MCP 工具延迟加载 — 通过 BM25 搜索发现
- `request_plugin_install` 工具 — 模型可建议安装未安装的插件

**可发现插件白名单** (`core-plugins/src/lib.rs:22-38`) — `TOOL_SUGGEST_DISCOVERABLE_PLUGIN_ALLOWLIST` 硬编码了 15 个插件 key（如 `"slack@openai-curated"`、`"gitlab@openai-curated"` 等）。只有在此白名单中的插件才会出现在 `tool_suggest` 推荐中。

在 Codex TUI 模式下，`request_plugin_install` 的 discoverable tools 会被过滤（`filter_request_plugin_install_discoverable_tools_for_client`）。

## 11) 生命周期总结

```
[安装]
  marketplace 发现 → 用户选择安装 → PluginStore.install_with_version()
  → 写入 config.toml → 清除缓存

[启动]
  maybe_start_plugin_startup_tasks_for_config()
  → sync 策展仓库 → 升级 Git marketplace → 启动远程 sync

[加载]
  plugins_for_config() → load_plugins_from_layer_stack()
  → load_plugin() × N → PluginLoadOutcome

[激活]
  LoadedPlugin.is_active() → effective_skill_roots/mcp_servers/apps/hook_sources
  → 注入到 SkillsManager / MCP 管理器 / Connector 系统 / Hooks 引擎

[卸载]
  uninstall_plugin() → 删除缓存目录 → 清除 config 条目 → 遥测
```

## 12) 源码抓手

| 想看什么 | 从哪里开始 |
|----------|-----------|
| PluginId 定义 | `plugin/src/plugin_id.rs:8` |
| PluginManifest | `core-plugins/src/manifest.rs:36` |
| LoadedPlugin | `plugin/src/load_outcome.rs:14` |
| PluginLoadOutcome | `plugin/src/load_outcome.rs:83` |
| PluginsManager | `core-plugins/src/manager.rs:383` |
| plugins_for_config | `core-plugins/src/manager.rs:457` |
| load_plugins_from_layer_stack | `core-plugins/src/loader.rs:110` |
| load_plugin | `core-plugins/src/loader.rs:496` |
| load_plugin_skills | `core-plugins/src/loader.rs:648` |
| load_plugin_hooks | `core-plugins/src/loader.rs:760` |
| PluginStore | `core-plugins/src/store.rs:25` |
| Marketplace 发现 | `core-plugins/src/marketplace.rs:334` |
| MarketplacePluginSource | `core-plugins/src/marketplace.rs:67` |
| 启动同步 | `core-plugins/src/startup_sync.rs` |
| 策展仓库 sync | `core-plugins/src/startup_sync.rs` |
| TOOL_SUGGEST 白名单 | `core-plugins/src/lib.rs:22` |
| build_plugin_injections | `core/src/plugins/injection.rs:15` |
| 可发现插件列表 | `core/src/plugins/discoverable.rs:20` |
| request_plugin_install | `tools/src/tool_discovery.rs:274` |
| plugin_namespace_for_skill_path | `plugin/src/plugin_namespace.rs:35` |

## 13) 相关文档

- 技能系统：`01-skill-system.md`
- 技能创作：`03-skill-authoring.md`
- 工具注册与发现：`../04-tools-execution-and-sandboxing/01-tool-registry-and-discovery.md`

---

> 读完这篇，你应该能追踪一个插件从 `plugin.json` 清单 → marketplace 安装 → PluginStore 缓存 → `load_plugin()` 四扩展点加载 → `effective_*` 方法提取 → 各系统注入的完整链路。
