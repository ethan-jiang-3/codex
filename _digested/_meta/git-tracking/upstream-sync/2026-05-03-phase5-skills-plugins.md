---
title: "Phase 5 Upstream Sync — 05-skills-and-plugins"
date: "2026-05-03"
status: "completed"
scope: "Phase 5 of progressive production plan: skill system, plugin mechanism, skill authoring"
---

# Phase 5 Sync Log — 05-skills-and-plugins

## 对齐锚点

- **baseline_ref**: `origin/main`
- **baseline_commit**: `35aaa5d9f` — Bound websocket request sends with idle timeout (#20751)
- **ethan_commit**: 本 Phase 开始前 `e5bce69c9`（Phase 4）

## 已复核源码路径

### 技能系统
- `core-skills/src/lib.rs` — crate 结构与公共导出
- `core-skills/src/model.rs:12` — `SkillMetadata` 完整字段
- `core-skills/src/model.rs:48` — `SkillPolicy`（隐式触发 + 产品限定）
- `core-skills/src/model.rs:56` — `SkillInterface` UI 元数据
- `core-skills/src/model.rs:66` — `SkillDependencies`（env_var 等）
- `core-skills/src/model.rs:87` — `SkillLoadOutcome` 加载结果
- `core-skills/src/model.rs:166` — `filter_skill_load_outcome_for_product()`
- `core-skills/src/manager.rs:50` — `SkillsManager` 结构体
- `core-skills/src/manager.rs:62` — `SkillsManager::new()` 启动流程
- `core-skills/src/loader.rs:105-122` — 关键常量（SKILLS_FILENAME、MAX_SCAN_DEPTH、MAX_SKILLS_DIRS_PER_ROOT）
- `core-skills/src/loader.rs:221-325` — `skill_roots_from_layer_stack_inner()` 6 来源
- `core-skills/src/loader.rs:440-580` — `discover_skills_under_root()` BFS 扫描
- `core-skills/src/loader.rs:582-643` — `parse_skill_file()` frontmatter 解析
- `core-skills/src/loader.rs:668-731` — `load_skill_metadata()` openai.yaml
- `core-skills/src/injection.rs:31-85` — `build_skill_injections()` 正文注入
- `core-skills/src/injection.rs:114-171` — `collect_explicit_skill_mentions()` 触发检测
- `core-skills/src/render.rs:160-200` — `build_available_skills()` prompt 渲染
- `core-skills/src/render.rs:86` — `SkillMetadataBudget` 预算管理
- `core-skills/src/invocation_utils.rs:29-145` — `detect_implicit_skill_invocation_for_command()`
- `core-skills/src/config_rules.rs:25` — `SkillConfigRules`
- `core-skills/src/env_var_dependencies.rs` — env var 依赖收集
- `core-skills/src/remote.rs` — remote skills API（标注为暂未使用）
- `skills/src/lib.rs:32` — `install_system_skills()` 嵌入系统技能
- `skills/src/assets/samples/` — 5 个嵌入技能（skill-creator, plugin-creator, skill-installer, openai-docs, imagegen）
- `core/src/context/skill_instructions.rs:7` — `SkillInstructions` ContextualUserFragment
- `core/src/context/available_skills_instructions.rs:9` — `AvailableSkillsInstructions`
- `core/src/skills_watcher.rs:32` — `SkillsWatcher` 文件监控（10s 节流）
- `core/src/session/mod.rs:2641-2662` — session 级技能注入 wiring
- `protocol/src/protocol.rs:3365` — `SkillScope` 枚举

### 插件系统
- `plugin/src/lib.rs` — PluginId、PluginCapabilitySummary、PluginHookSource、PluginTelemetryMetadata
- `plugin/src/plugin_id.rs:8` — `PluginId { plugin_name, marketplace_name }`
- `plugin/src/plugin_id.rs:26` — `PluginId::parse("name@marketplace")`
- `plugin/src/plugin_namespace.rs:35` — `plugin_namespace_for_skill_path()`
- `plugin/src/load_outcome.rs:14` — `LoadedPlugin<M>` 结构体
- `plugin/src/load_outcome.rs:83` — `PluginLoadOutcome` + effective_* 方法
- `core-plugins/src/lib.rs:22-38` — `TOOL_SUGGEST_DISCOVERABLE_PLUGIN_ALLOWLIST`（15 个插件）
- `core-plugins/src/manager.rs:83-107` — `PluginsConfigInput`
- `core-plugins/src/manager.rs:383-395` — `PluginsManager` 结构体
- `core-plugins/src/manager.rs:457` — `plugins_for_config()` 加载入口
- `core-plugins/src/manager.rs:789` — `install_plugin()`
- `core-plugins/src/manager.rs:884` — `uninstall_plugin()`
- `core-plugins/src/manager.rs:1374` — `maybe_start_plugin_startup_tasks_for_config()`
- `core-plugins/src/loader.rs:110-151` — `load_plugins_from_layer_stack()`
- `core-plugins/src/loader.rs:496-610` — `load_plugin()` 核心加载逻辑
- `core-plugins/src/loader.rs:648-676` — `load_plugin_skills()`
- `core-plugins/src/loader.rs:760-813` — `load_plugin_hooks()`
- `core-plugins/src/manifest.rs:36-74` — `PluginManifest` + `PluginManifestPaths`
- `core-plugins/src/manifest.rs:137` — `load_plugin_manifest()`
- `core-plugins/src/marketplace.rs:67-107` — `MarketplacePluginSource` + `MarketplacePluginPolicy`
- `core-plugins/src/marketplace.rs:334` — marketplace 发现路径
- `core-plugins/src/store.rs:25-29` — `PluginStore` 磁盘缓存布局
- `core-plugins/src/startup_sync.rs` — 策展仓库 sync（git clone → HTTP API → export archive）
- `core/src/plugins/injection.rs:15` — `build_plugin_injections()` 显式提及
- `core/src/plugins/discoverable.rs:20` — `list_tool_suggest_discoverable_plugins()`
- `tools/src/tool_discovery.rs:274` — `request_plugin_install` 工具
- `tools/src/tool_discovery.rs:114-126` — TUI 模式下过滤 `request_plugin_install`

## 产出文档

1. `05-skills-and-plugins/01-skill-system.md`
2. `05-skills-and-plugins/02-plugin-mechanism.md`
3. `05-skills-and-plugins/03-skill-authoring.md`

## 复核统计

- 源码文件：50+ paths across 8 crates
- 涉及 crate：skills, core-skills, plugin, core-plugins, core, protocol, tools, tui
