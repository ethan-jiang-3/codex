---
title: "_tmp_tracking"
doc_type: "working-notes-index"
status: "active"
updated: "2026-05-03"
purpose: "为临时但需要持续追踪的问题建立分目录边界，避免把不同类型的 issue、计划和中间材料混在仓库根下。"
owns: "`_tmp_tracking/` 下各子目录的用途、命名口径和使用边界"
out_of_scope:
  - "把这些临时材料提升为 `_digested/` canonical 文档"
  - "替代 `_digested/_meta/` 的正式历史留痕"
---

# _tmp_tracking

`_tmp_tracking/` 用来放"暂时性的、但不是一次性 scratch"的追踪材料。

定位：

- 它比散落在仓库根下的临时文件更有组织。
- 它仍然不是 `_digested/` 或 `_digested/_meta/` 那种正式 owner / archive 层。
- 这里的内容可以服务于某一轮工作，但不应被当成源码事实或长期 canonical 文档。

## 当前子目录

- `_digested_isuues/`
  - 跟踪 `_digested` 相关的问题、复核计划、coverage map、triage、临时审计材料。
  - 典型内容：progressive sync 拆题、owner map、review report、gap fill plan。
- `_install_issues/`
  - 跟踪安装、环境准备、依赖、启动失败、平台兼容性这类安装过程问题。
  - 适合放安装排障记录、复现笔记、修复方案草稿。
- `TEMPLATES/`
  - 存放五类工作文件的模板，供新一轮追踪时 `cp` 使用。

## 五类工作文件

`_digested_isuues/` 中的工作文件按用途分为五类，构成一条完整的"评估 → 规划 → 执行 → 验证"链路：

| 类别 | 用途 | 关键特征 |
|------|------|---------|
| **审核报告** (Review Report) | 6 维评分 + 源码声明验证 | 问题按严重/中等/轻微分级，含合并建议 |
| **差距填补计划** (Gap-fill Plan) | 每个差距精确到文档+章节+内容要点 | 按必须/应该/可选排序执行 |
| **最终评估** (Final Evaluation) | 修复前后对比 + 修改清单 + 剩余问题 | 含数据摘要（行数、图表数、覆盖率） |
| **质量改进计划** (Quality Plan) | 多周路线图 + 定量验收标准 | 含风险与缓解措施 |
| **上游同步工件** (Upstream Sync) | Triage / Owner Map / Coverage Map / Sync Plan | 管理上游源码变化到 `_digested` 的同步 |

这五类的数据流是：
```
源代码（git range） → Triage → Owner Map → Coverage Map → Sync Plan → 阶段切片执行
                                ↓
审核报告 → Gap-fill Plan → 最终评估
```

## 使用规则

- 新建内容时，先判断它属于哪一类 issue，再放到对应子目录。
- 文件名尽量保留日期前缀，例如 `YYYY-MM-DD-*.md`，方便按时间追踪。
- 这些文件可以引用 `_digested/`、release notes、tests、源码路径，但它们自己不是 owner truth。
- 一旦某条结论已经稳定，并且属于正式文档体系，应写回 `_digested/` 或 `_digested/_meta/`，不要长期只停留在这里。
- 如果以后出现新的追踪面，优先在 `_tmp_tracking/` 下新增兄弟子目录，而不是重新在仓库根下散放 `_tmp_*` 目录。

## 与正式文档的关系

- 源码与 tests 仍然是事实源。
- `_digested/` 是 owner 文档层。
- `_digested/_meta/` 是正式历史与 tracking 归档层。
- `_tmp_tracking/` 只是工作中的问题追踪层。
