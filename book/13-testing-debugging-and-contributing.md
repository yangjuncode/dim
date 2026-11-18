# 第 13 章：工程质量保障、排障调试与贡献指南

> **本章目标**：学会像核心维护者一样验证、调试和贡献代码——掌握 `xtask` 四大守护命令、跨语言/TUI 黑盒测试套件、生产排障三板斧（`export`、`recording`、`replay`），以及 Dim 独特的工程纪律。

---

## 13.1 改完代码先跑什么？验证工具链全览

在提交 PR 或测试你二次开发的模块前，按照由快到慢的顺序使用以下验证命令：

### 1. 常规 Rust 编译与静态检查
```bash
cargo fmt --check
cargo clippy --workspace --all-targets
cargo test -p dim-agent-runtime -p dim-agent-capabilities
```

### 2. Dim 专属架构门禁：`xtask check`
只要你**新增了 Crate、调整了依赖关系、新增或修改了 `#[rutis::service]` 跨语言服务、或改动了文档路径**，务必执行：
```bash
# 重新生成目录图、服务目录文档与 TS 类型声明（如有变更）
cargo run --locked -p xtask -- graph
cargo run --locked -p xtask -- types

# 一键运行全部 5 项架构与契约门禁检查！
cargo run --locked -p xtask -- check
```
[`xtask check`](../xtask/src/main.rs) 会自动拦截以下违规：
1. **依赖边界检查（`boundaries`）**：例如阻止 `dim-agent-types` 误引入运行时，或底层 crate 反向依赖上层 crate；
2. **服务目录一致性（`catalog::check`）**：确保 [`docs/architecture/rust-agent/generated/catalog.md`](../docs/architecture/rust-agent/generated/catalog.md) 与真实代码一致；
3. **跨语言类型一致性（`types::check`）**：确保 [`npm/branch-services.d.ts`](../npm/branch-services.d.ts) 与 `#[rutis::service]` 源码一字不差；
4. **npm 包校验（`npm::check`）**；
5. **文档与注释内链接有效性（`comment_links::check`）**。

### 3. 性能与内存基线测量：`xtask measure`
如果你修改了事件分发、历史投影或启动路径，用以下命令评估影响：
```bash
cargo run --locked -p xtask -- measure
```

---

## 13.2 全栈测试矩阵

Dim 的测试覆盖从内存单元测试一直延伸到真实终端交互：

| 测试层级 | 位置与典型命令 | 测什么 |
| --- | --- | --- |
| **Runtime & Capability 契约测试** | `cargo test -p dim-agent-integration-tests` ([`tests/integration/`](../tests/integration/tests/extensions.rs)) | 用 `MinimalHost` 验证跨 Crate 扩展、分支隔离、崩溃重建与权限边界 |
| **跨语言（TS / Python）夹具测试** | [`runtime/fixtures/`](../runtime/fixtures/) | 验证 Node `session-node` 与 Python `py` 桥接器下的工具调用、`dispatcher.consume` 与热重载 |
| **App Server & Web 鉴权测试** | `cargo test -p dim-app-server` | 测试 SQLite 存储恢复、`web_entry` Cookie 鉴权与爆破限流、v3 RPC |
| **TUI 终端黑盒 E2E** | `dim-cli` 相关脚本测试与 tmux E2E | 在真实伪终端（PTY）中验证按键、粘贴折叠、Scrollback/Fullscreen 切换不丢数据 |

---

## 13.3 生产与二开排障“三板斧”

遇到“模型怎么没看到我注入的提示？”、“会话恢复后状态不对了？”、“某次调用到底发了什么给 Anthropic/OpenAI？”这类疑难杂症时，别猜，直接用 Dim 内置的三件诊断重器：

### 第一斧：导出全量时间线（`dimcode export`）
```bash
dimcode export <session-id> --output debug-trace.jsonl
```
导出的内容不仅包含该 Session 下所有分支的完整 `AgentEvent` 序列，还能还原事件之间的父子分支切点，让你用显微镜检查到底哪条事件顺序或 payload 不对。

### 第二斧：开启模型原始请求录制（`dimcode recording`）
想确认经过 `PromptRegistry` + `ContextPreparation` + `Aimux` 转换后，**真正发往大模型 HTTP 接口的 Payload** 到底长什么样？
```bash
# 查看当前录制状态或开启录制
dimcode recording on
# 跑完复现步骤后，关闭并查看生成的 model-log/recordings.jsonl
dimcode recording off
```

### 第三斧：离线重放（`xtask context-replay` & `restore-replay`）
如果某个线上 SQLite 库里某次长会话的压缩或恢复逻辑行为怪异，你可以把数据库拷贝出来，用 `./dev` 或 `xtask` 离线单步重放它的上下文构建和恢复过程，**零 Token 消耗**完成根因定位：
```bash
cargo run --locked -p xtask -- context-replay --help
cargo run --locked -p xtask -- restore-replay --help
```

---

## 13.4 Dim 贡献准则与“减法审查”文化

认真读一遍根目录的 [`AGENTS.md`](../AGENTS.md)，你会发现 Dim 团队有几条非常值得学习的工程文化：

1. **所有数据边界默认先设限、默认隔离**：
   - 新增状态先问：**它属于 `process`、`session` 还是 `branch`？** 能放 `branch` 绝不放 `session`，能放 `session` 绝不污染 `process`。
   - 来自未知项目的插件、Hooks、MCP Server，默认必须关闭。
2. **禁止制造“第 N+1 套真相”**：
   - 不要绕开 `Timeline` 偷偷改内存状态；
   - 不要绕开 `input` 和 `ToolResult.content` 往模型上下文塞私货；
   - 不要写"为了省事以后再删"的兼容兜底分支（Fallback）。
3. **减法审查（Ablation Review）**：
   - 在提交架构级改动时，不仅要证明“加上这段代码能工作”，还要主动追问：**有没有哪层抽象、哪份状态副本、哪条参数其实是多余的、删掉后系统反而更干净？**
4. **保持中文/英文设计与文档同步**：
   - 当你设计或改动核心边界时，先读一读 `docs/adr/` 与 `docs/design/` 里的设计文档，用 `xtask check` 确保实现与契约始终同行。

---

## 结语

恭喜你走完了整本《Dim 完全学习指南》！

从第一眼打开 `dimcode` 的终端双模式、到看透 `Timeline + Dispatcher + rutis` 的微内核哲学、再到亲手写出 TypeScript / Python / Rust 插件与 Web 扩展面板——你现在不仅是一名能把 `dimcode` 玩出花的熟练用户，也已经具备了在其基础上自由演进与二次开发的全部底层视野。

接下来，建议你立刻打开终端运行 `./dev`，对照 [第 10 章](./10-dev-ts-and-python-plugins.md) 或 [第 11 章](./11-dev-rust-capabilities-and-features.md) 亲手写出你的第一个扩展吧！
