# 第 2 章：快速上手与日常实战指南

> **本章目标**：把 Dim 真正跑起来——覆盖安装/源码开发启动、模型接入、双模式 TUI 体验、高效率输入技巧（引用文件、贴图、队列与打断），以及完整命令速查。

---

## 2.1 两种启动方式：正式安装 vs 源码开发

### 方式一：通过 npm 快速体验安装（用户视角）

如果你只想直接把最新 Rust alpha 版本用起来，只需要 Node.js 环境并执行：

```bash
npm install -g dimcode-alpha@alpha
# 安装后可用命令为 dima
dima --version
dima
```

> **提示**：仓库里存在两个 npm 通道包名：`dimcode-alpha` 提供 `dima` 命令（对应本仓库的主力 Rust 实现），而正式稳定版通道对应 `dimcode` / `dim`。

### 方式二：从本仓库源码编译与 `./dev` 隔离运行（开发者强烈推荐）

既然你在学习这个项目并打算二次开发，最安全、最顺手的方式是使用仓库根目录自带的 [`./dev`](../dev) 脚本。

**前置依赖**：
- Rust 工具链（仓库根目录 [`rust-toolchain.toml`](../rust-toolchain.toml) 固定为 `1.100.0`，`rustup` 会自动拉取）
- Node.js（用于可选 TypeScript 插件或 Web 前端构建）

**启动命令**：

```bash
# 1. 启动交互式 TUI（自动编译 debug 版 dimcode，数据完全隔离在 ~/.dimcode-dev）
./dev

# 2. 携带参数运行，例如查看帮助或执行单次无头任务
./dev --help
./dev exec -p "用一句话解释这个仓库是干嘛的"

# 3. 也可以先编译再测试
cargo build --bin dimcode
```

**为什么 `./dev` 非常贴心？**
- 正式使用的 `dimcode` 默认将数据保存在 `~/.dimcode`（可通过 `--home PATH` 覆盖）。
- `./dev` 会把 `--home` 自动指向 `~/.dimcode-dev`，让你在折腾源码、实验插件甚至测试数据库迁移时，**绝对不会弄坏自己日常工作的正式会话记录**。

---

## 2.2 接入大模型：三种方式任选

第一次打开 Dim 前，需要让它知道如何连接你的大模型。Dim 支持三种非常灵活的配置模式：

### 方式 A：环境变量直连模式（最快、无需写配置文件）

只需设置 `DIM_BASE_URL`、`DIM_API_FORMAT`、`DIM_MODEL` 和密钥，即可临时接管任何主流接口（注意这两个环境变量必须成对出现，且不与 `--provider` / `--model-config` 混用）：

```bash
# 示例 1：连接 Anthropic 官方或兼容网关
export DIM_BASE_URL="https://api.anthropic.com"
export DIM_API_FORMAT="anthropic"
export DIM_API_KEY="sk-ant-..."
export DIM_MODEL="claude-sonnet-4-5"
./dev

# 示例 2：连接 OpenAI Chat Completions 兼容接口（如本地 vLLM / DeepSeek / OpenRouter 等）
export DIM_BASE_URL="https://api.openai.com/v1"
export DIM_API_FORMAT="openai-chat"
export DIM_API_KEY="sk-..."
export DIM_MODEL="gpt-4o"
./dev
```

支持的 `DIM_API_FORMAT` 包括：
- `anthropic`（使用 `x-api-key`）
- `openai-chat`（使用 `Bearer`）
- `openai-responses`（OpenAI Responses API）
- `openai-compatible`
- `gemini`（使用 `x-goog-api-key`）
- `xai` / `xai-responses`

还可以通过 `DIM_CUSTOM_HEADERS`（换行分隔的 `Name: Value`）、`DIM_EFFORT_LEVEL`（推理强度）、`DIM_AUTO_COMPACT_WINDOW`（如 `128k`）、`DIM_MAX_CONTEXT_TOKENS` / `DIM_MAX_OUTPUT_TOKENS` 进行精细覆盖，**且不会改写磁盘上已经保存的默认 Provider 配置**。

### 方式 B：交互式配置 Provider（日常使用推荐）

进入 TUI 后直接输入斜杠命令：
- **`/connect`**（别名 `/provider`）：打开 Provider 管理面板，添加、编辑、测试、启用或切换基于 API Key 的自定义服务商与模型列表。
- **`/login`**：如果你的构建中集成了受管账号或 OAuth Provider，可通过浏览器、企业邮箱或设备码登录（对应 `/logout` 退出）。
- **`/models`**：查看当前所有可用模型并切换，同时支持调整推理强度（也可直接用 **`/effort`** 或别名 `/reasoning`）与快速模式（**`/fast`**，仅当模型支持服务分级时出现）。

### 方式 C：显式 `--model-config` JSON 文件（工程部署 / 测试推荐）

当你启动独立的 App Server 或在 CI / 测试环境运行时，可以用 `--model-config /path/to/models.json` 传入完整的模型与 Provider 描述文件，配合 `--model` 与 `--provider` 启动。

---

## 2.3 认识双模式 TUI：Scrollback vs Fullscreen

启动 `dimcode`（或 `dimcode tui`）后，你会立即感受到它与众不同的终端体验。Dim 拥有**两套一等公民级别的终端渲染引擎**：

| 维度 | Scrollback（滚动历史模式，默认） | Fullscreen（全屏 TUI 模式） |
| --- | --- | --- |
| **核心理念** | 像 `git log` / 标准 CLI 一样自然打印进终端原生历史缓冲 | 类 IDE / Ratatui 全屏布局（Alternate Screen），独立管理视口 |
| **文字选中复制** | 100% 原生终端鼠标拖选，无任何干扰 | 内置智能鼠标选区（释放即自动复制） |
| **工具卡片折叠** | 由 `/tools` 全局控制（`collapsed` / `summary` / `expanded`），无单行箭头 | 每条工具调用、每批修改都带独立 `▸`/`▾` 可鼠标点击或 `Enter/Space` 展开 |
| **历史回溯** | 终端滚轮直接上滑看全部历史；`/timeline` 可重绘跳转 | 内置虚拟列表与可拖拽右侧滚动条，`Home`/`End`/`Tab` 跳转 |
| **适用场景** | 喜欢原生终端质感、长流式输出直接留存滚动区 | 需要密集点开查看 Diff、同时盯多个分支或面板 |

**热切换与设为默认**：
- 在对话中随时输入 **`/tui`**，瞬间从 Scrollback 切换到 Fullscreen（或反之），**正在生成的回复、后台运行的任务、队列中的消息全部原封不动保留**。
- 启动时指定：`dimcode tui --fullscreen` 或 `dimcode tui --scrollback`。
- 永久保存默认模式：`dimcode tui --fullscreen default` 或 `dimcode tui --scrollback default`。

### 细节体验彩蛋
- **工具摘要行**：哪怕在默认折叠状态下，Shell 组也会聪明地把执行过的命令程序名拼在标题上，例如：`✓ Ran 3 commands · rg ×2, cargo`，一眼知道 Agent 干了啥。
- **单列清晰 Diff**：文件写入和编辑采用统一单列对比，带旧/新行号和颜色高亮，且仅根据**实际记录的变更快照**计算 `+N −M`，绝不伪造统计。
- **终端状态与进度条**：在支持的现代终端（Ghostty、WezTerm、iTerm2 3.6.6+、Windows Terminal 等）中，Dim 通过 `OSC 7501` 与 `OSC 9;4` 在终端标签页直接展示"工作中 / 等待审批 / 已完成 / 报错"进度指示（可通过环境变量 `DIM_TERMINAL_STATUS=off`、`DIM_TERMINAL_PROGRESS=off` 关闭）。

---

## 2.4 输入区黑魔法：打字、贴图、引用与插队

Dim 的输入编辑器（Composer）在表面简洁之下藏着一套非常高效的操作逻辑：

### 1. 多行文本与大段粘贴
- **`Enter`**：发送消息。
- **`Ctrl+J`**（或在支持的终端中 **`Shift+Enter`** / `Alt+Enter`）：插入换行。
- **智能粘贴折叠**：当你一次性粘贴多行内容或超过 `16 KiB` 的长日志时，输入框不会炸屏，而是优雅地折叠成一个原子标记 `[Paste 1: 120 lines]`，用 `Backspace` 可以一键整块删掉。超过 `16 KiB` 的大文本（最大 `10 MiB`）在按 `Enter` 提交时会自动存为本地文本附件传入。

### 2. 图片、剪贴板与 `@` 文件引用
- **`@` 路径补全**：输入 `@` 立即弹出当前工作目录（`--cwd`）下的文件列表，`↑`/`↓` 选择，`Tab` 进入目录或选中文件（显示为 `[File: guide.md]`）。
- **`Ctrl+V` 粘贴图片或文件**：直接把系统剪贴板里的截图、或者文件管理器里复制的图片/文件贴进输入框！**甚至在 SSH 远程连接时**，只要本地终端支持 kitty 剪贴板协议（OSC 5522，如 kitty、Ghostty 1.4+），`Ctrl+V` 也能把你本地电脑的截图隔空传给远端 Dim！
- 拖拽文件进终端（依赖终端传递路径）同样会自动识别：图片变成图片附件（支持 PNG/JPEG/WebP/GIF，最多 20 张，提交前自动按最长边 2048px、5MB base64 上限缩放校验），普通文件变成路径引用。

### 3. `/` 菜单与 `/skill:<name>` 技能点名
输入 `/` 会按精心编排的固定顺序弹出命令面板。如果列表里有 Skills，选中某个 Skill（按 `Enter` 或 `Tab`）会在草稿里插入一个高亮的 `/skill:<name> ` 药丸（Pill）——**注意：插入 Skill 引用不会立刻发送**，你可以继续在后面补上你的具体指令再一起提交！

### 4. 运行中的“队列（Queue）”与“纠偏（Steer）”
当 Agent 正在吭哧吭哧改代码时，你突然想到一个补充要求，Dim 给你四种精确控制：

| 快捷键 / 操作 | 行为 | 适用场景 |
| --- | --- | --- |
| **`Enter`**（默认 `steer` 模式） | **插队纠偏（Steer）**：在当前正在跑的工具/步结束后，立刻把你的话插进本回合给模型看 | “别改那个文件了，先看下 `tests/`！” |
| **`Ctrl+S`**（切到另一种模式） | 在默认 `steer` 下按 `Ctrl+S` 会变为 **Queue（排队）**；在 `/settings` 把默认改为 `queue` 后则相反 | “等手头这个任务彻底做完，下一轮再帮我写文档” |
| **`Ctrl+Enter`** | **立刻打断并发送**：马上中止当前模型/工具，把刚才排队的草稿收回来，直接发出新消息 | 发现模型彻底跑偏了，一秒都不想多等 |
| **`Esc`** | **中止当前运行**：优雅停止本轮，并将所有还没消费的排队消息退回输入框 | 喊停并自己重新组织语言 |
| **`Shift+Left`**（空输入框时） | **撤回最近一条排队消息**回输入框继续修改 | 刚才排进去的那句写错字了 |
| **`/queue`** | 打开队列管理面板，查看正在等待或正在融入的指令，支持单独删除或提前 Steer | 队列里攒了好几条指令时精细调度 |

### 5. `!` 与 `!!` 本地 Shell 直通车
不想为了看一眼 `git status` 就浪费一次大模型 Token？直接在输入框里：
- **`!git status`**：在当前 Session 环境下执行命令，**输出结果会进入后续大模型上下文**（相当于你帮模型看了一眼并告诉它）。
- **`!!ls -la`**：在当前 Session 环境下执行命令，**只给你自己看终端输出，不进入模型上下文**。

---

## 2.5 常用命令与 CLI 子命令速查

### 对话内 Slash 命令精选

- **会话管理**：`/new`（新开）、`/resume`（恢复历史会话）、`/rename`（改标题）、`/timeline`（按消息历史跳转/编辑/分叉）、`/export`（导出 Markdown/JSON）、`/compact`（手动压缩上下文）、`/usage`（查看 Token 与套餐用量）
- **模型与权限**：`/models`、`/effort`、`/fast`、`/connect`、`/login`、`/permissions`（切换 `read-only` / `workspace-write` / `full-access` 及命令规则）
- **高级与分支**：`/branch`（管理分支）、`/fork`（分叉新会话）、`/lines`（切换当前查看的分支线）、`/goal`（开启/编辑/暂停目标模式）、`/tasks`（查看后台任务、子智能体与工作流）
- **扩展与工作流**：`/skills`、`/reload-skills`、`/mcp`、`/plugins`、`/reload-plugins`、`/hooks`、`/scheduled`、`/workflow`、`/workflow:save`
- **界面与联动**：`/tui`、`/tools`、`/theme`、`/language`、`/keys`、`/settings`、`/web`、`/report`、`/exit`

### 终端 CLI 子命令速查

```bash
# 1. 交互式 TUI（带首条消息/图片/恢复指定会话）
dimcode tui -p "帮我检查 src/lib.rs 的错误处理" --image error.png
dimcode tui --resume <session-id>

# 2. 在独立 Git Worktree 中启动新会话（不脏当前工作区）
dimcode --worktree --worktree-branch feat/my-fix

# 3. 无头执行（CI / 脚本神器）
dimcode exec -p "修复单元测试" --permission workspace-write --json
dimcode exec --verify "cargo test" --repair-attempts 3 -p "修复构建报错"

# 4. 会话管理与全量时间线导出（调试排障利器）
dimcode session export <session-id> --format markdown
dimcode export <session-id> --output trace.jsonl

# 5. 打开浏览器 Web 工作区
dimcode web

# 6. 管理 MCP、Skills、Hooks、插件与定时任务
dimcode mcp list
dimcode plugin install <path-or-git-url>
dimcode scheduled list
```

---

## 本章小结

现在你已经能熟练启动 `dimcode`（或用 `./dev` 启动隔离开发环境），配好大模型，并在双模式 TUI、队列纠偏、`@` 文件与截图粘贴间自由穿梭了。

下一章，我们将带你解锁 Dim **最硬核、最能拉开生产力差距的特色功能**：Timeline 分支与 Subagent 的实战区别、Goal/Pursuit 自治模式、Workflow、定时任务、CodeMode 与安全沙盒体系。
