---
title: "04 — 文件操作（File Operations）"
doc_type: "owner"
status: "current"
branch: "ethan"
updated: "2026-05-03"
audience: "需要理解 Codex 如何读写文件、并发控制、以及 patch 应用机制的人"
purpose: "给出文件系统的抽象接口、并发控制方式、apply_patch 的解析与应用流程、以及 list_dir 的安全策略"
owns: "文件操作面：ExecutorFileSystem trait、文件并发模型、apply_patch 解析/验证/应用、目录遍历安全策略"
update_when:
  - "ExecutorFileSystem trait 新增方法或语义变化时"
  - "apply_patch 解析策略变化时"
  - "文件并发控制方式变化时"
out_of_scope:
  - "沙箱文件系统限制（见 03-sandboxing.md）"
  - "exec-server 的文件系统协议（见 02-exec-and-shell.md）"
---

# 04 — 文件操作（File Operations）

> [!IMPORTANT]
> 这一篇解释 Codex 如何安全地读写用户的文件系统——从抽象接口到 patch 应用到目录遍历策略。

> 读完这篇你应该能回答：文件操作经过哪些抽象层、apply_patch 怎么解析和验证、list_dir 怎么限制遍历深度和拒绝访问路径。

## 1) 先记住三件事

1. **文件系统通过 trait 抽象**：`ExecutorFileSystem` 定义 8 个操作（read/write/create_dir/get_metadata/read_dir/remove/copy），本地和远程各有实现。
2. **并发控制不在 trait 层**：没有内置锁机制；`is_mutating()` 决定是否需要等待文件监控静默窗口（`tool_call_gate`），作为跨工具协调的方式。
3. **Apply patch 有自己的一套安全验证链**：Tree-sitter 解析 → 严格/宽松模式 → 4 遍模糊搜索 → unicode 规范化匹配。

## 2) ExecutorFileSystem Trait

定义在 `file-system/src/lib.rs:134`：

| 方法 | 签名 | 说明 |
|------|------|------|
| `read_file` | `async fn(path) -> Vec<u8>` | 二进制读 |
| `read_file_text` | `async fn(path) -> String` | UTF-8 读（默认委托 read_file） |
| `write_file` | `async fn(path, Vec<u8>)` | 二进制写 |
| `create_directory` | `async fn(path, CreateDirectoryOptions)` | `recursive: bool` |
| `get_metadata` | `async fn(path) -> FileMetadata` | `is_directory, is_file, is_symlink`, timestamps |
| `read_directory` | `async fn(path) -> Vec<ReadDirectoryEntry>` | `file_name, is_directory, is_file` |
| `remove` | `async fn(path, RemoveOptions)` | `recursive, force` |
| `copy` | `async fn(path, CopyOptions)` | `recursive` |

**没有** `rename`、`symlink`、`chmod` 操作。

**原子写入**：无。没有 write-to-temp-then-rename 模式。`write_file` 直接写入目标路径。

## 3) 并发控制

Codex 不使用文件锁。并发安全通过以下方式实现：

| 机制 | 说明 |
|------|------|
| `ToolHandler::is_mutating()` | Shell/apply_patch 返回 true，list_dir/view_image 返回 false |
| `tool_call_gate` | `ReadinessFlag`，在文件监控检测到变化后的"静默窗口"期间阻止修改类工具 |
| `UnifiedExecProcessManager` | `Mutex<ProcessStore>` 保护进程注册表 |
| `Arc<dyn ExecutorFileSystem>` | trait object 无状态，由实现者负责线程安全 |

## 4) Apply Patch：解析、验证、应用

`apply-patch/src/` 实现了一套自定义 patch 格式。

### Patch 格式

```
*** Begin Patch
*** Add File: /path/to/new_file.txt
+content line 1
+content line 2
*** End Patch

*** Begin Patch
*** Update File: /path/to/existing.rs
*** Move to: /path/to/renamed.rs (可选)
@@ context line
-old line
+new line
*** End Patch
```

### 解析流程（`parser.rs`）

1. 状态机解析：`NotStarted → Started → AddFile/DeleteFile/UpdateFile → Ended`
2. 两种模式：
   - **Strict**：严格匹配 patch 标记
   - **Lenient**：剥离 heredoc 包装（`<<EOF\n...\nEOF`），GPT-4.1 常见输出格式
3. Hunk 类型：
   - `AddFile { path, contents }`
   - `DeleteFile { path }`
   - `UpdateFile { path, move_path, chunks: Vec<UpdateFileChunk> }`

### 调用检测（`invocation.rs`）

`maybe_parse_apply_patch()` 检测两种调用形式：
1. **直接调用**：`["apply_patch", "<patch string>"]`
2. **Shell heredoc**：`["bash", "-lc", "apply_patch <<'EOF'\n...\nEOF"]`

Shell heredoc 检测使用 Tree-sitter 的 `tree_sitter_bash` 语法做**严格查询**——只匹配单语句脚本（精确的 `apply_patch <<EOF ... EOF` 或 `cd <path> && apply_patch <<EOF ... EOF`），拒绝前置/附加了额外命令的脚本。

### 模糊搜索定位（`seek_sequence.rs`）

在文件中定位要修改的行时，使用 4 遍搜索：
1. 精确匹配
2. 右去空格匹配
3. 全 trim 匹配
4. Unicode 规范化匹配（全角破折号→ASCII、弯引号→直引号、全角空格→半角等）

### 流式进度

`StreamingPatchParser` 逐字符增量解析，每完成一个 hunk 就通过 `ApplyPatchArgumentDiffConsumer` 发送 `PatchApplyUpdatedEvent`，速率限制 500ms。

## 5) List Dir 安全策略

`ListDirHandler`（`core/src/tools/handlers/list_dir.rs`）：

**参数**：
- `dir_path`：**必须是绝对路径**（拒绝相对路径）
- `offset`：1-indexed，默认 1
- `limit`：默认 25
- `depth`：默认 2

**遍历**：
- BFS 逐层遍历
- 每个目录条目检查 `ReadDenyMatcher`（基于沙箱策略的可读路径判断）
- 被拒绝的路径 → 跳过（不报错以保护隐私）
- 符号链接标记 `is_symlink()` → 显示 `@` 后缀
- 目录显示 `/` 后缀

## 6) 源码抓手

| 想看什么 | 从哪里开始 |
|----------|-----------|
| FileSystem trait | `file-system/src/lib.rs:134` |
| Apply patch 解析 | `apply-patch/src/parser.rs:265` — `parse_one_hunk()` |
| Apply patch 调用检测 | `apply-patch/src/invocation.rs` — `maybe_parse_apply_patch()` |
| Tree-sitter bash 查询 | `apply-patch/src/invocation.rs:269-313` |
| 模糊搜索 | `apply-patch/src/seek_sequence.rs` |
| 流式 patch 解析 | `apply-patch/src/streaming_parser.rs` |
| Patch 应用 | `apply-patch/src/lib.rs:261` — `apply_hunks_to_files()` |
| Patch 拦截（shell 中检测） | `core/src/tools/handlers/apply_patch.rs:467` — `intercept_apply_patch()` |
| List dir handler | `core/src/tools/handlers/list_dir.rs:57` — `ListDirHandler::handle()` |
| is_mutating 实现 | `core/src/tools/handlers/shell.rs:196` |

## 7) 相关文档

- 工具注册：`01-tool-registry-and-discovery.md`
- 沙箱机制：`03-sandboxing.md`
- 后台进程：`05-background-processes.md`

---

> 读完这篇，你应该能追踪文件从 model 的 tool call → apply_patch 解析/验证/模糊匹配 → ExecutorFileSystem 写入的完整链路。
