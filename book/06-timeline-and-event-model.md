# 第 6 章：核心基石 —— `dim-agent-types` 与 Timeline 时间总线

> **本章目标**：彻底理解 Dim 如何把复杂的 AI Agent 运行过程抽象成一条**不可变、单调递增的事实账本（Timeline）**，并掌握事件类型、原子批提交、`before` 钩子、扩展事件与持久化查询模型。

---

## 6.1 为什么说 Timeline 是“唯一的事实源”？

在传统程序里，如果我们想知道“当前对话进行到哪一步、有哪些工具在跑、待审批请求是什么”，通常会去读几个内存变量（`is_running`、`pending_approvals`、`messages`）。但一旦遇到**进程崩溃重启、插件热重载、多个客户端同时连接、或者分支回溯**，内存变量就会变成一场灾难。

Dim 的解法来自经典事件溯源（Event Sourcing）思想，并将其打磨到极度克制（见 [`docs/architecture/rust-agent/time-bus.md`](../docs/architecture/rust-agent/time-bus.md)）：

> **公理 1：Timeline 记录“已经发生过的事实（Facts）”，而不是“打算做什么”。**  
> **公理 2：序列号（`sequence`）是 Timeline 上唯一有效的顺序标尺，永远禁止使用时间戳或外部时钟排序。**

你可以把 Timeline 想象成银行的核心总账本：
- 你不会去修改第 10 行的数字，只会追加第 11 行冲正或新交易；
- 账本每一页都盖有严格连续、单调递增的流水号（`1, 2, 3, ...`）；
- 不管你的手机 App 还是柜台电脑什么时候重启，只要从第 1 行到当前页扫一遍，就能算出分毫不差的余额！

---

## 6.2 解剖一条事件：`AgentEvent` 与 `EventDraft`

打开 [`crates/dim-agent-types/src/events.rs`](../crates/dim-agent-types/src/events.rs)，核心结构体非常干净：

### 写入前的草稿：`EventDraft`
生产者想往 Timeline 写事实时，构造的是 `EventDraft`：
```rust
pub struct EventDraft {
    pub id: Option<String>,             // 可选预分配 ID（如流式文本首个 delta 复用同一 ID）
    pub tags: Option<Vec<String>>,      // 附加标签
    pub parent_call_id: Option<String>, // 如果是子工具（如 CodeMode 嵌套调用）产生的事件，标记父调用 ID
    pub kind: EventKind,                // 具体事件类型与强类型 payload
}
```
> 注意：`EventDraft` **没有 `sequence` 和 `create_at`**——顺序和时间只能由 Timeline 提交那一刻权威赋予！另外，如果事件带了 `parent_call_id`，`AgentEvent::context_flags()` 会明确返回 `(false, false)`，保证子执行事实绝不会意外污染主模型对话上下文。

### 提交后的铁证：`AgentEvent`
一旦写入 Timeline 成功，它就变成了不可篡改的 `AgentEvent`：
```rust
pub struct AgentEvent {
    pub id: String,
    pub sequence: u64,                  // 从 1 开始单调递增，且不超过 JS 安全整数上限 (2^53 - 1)
    pub create_at: u64,                 // 毫秒时间戳（仅供展示，绝不用于逻辑排序）
    pub tags: Option<Vec<String>>,
    pub parent_call_id: Option<String>,
    pub kind: EventKind,                // 展开为 { "type": "domain", "name": "event", "data": ... }
}
```
> **跨语言整数细节**：注意到 `sequence` 和 `create_at` 都受 `JS_MAX_SAFE_INTEGER`（`9_007_199_254_740_991`）校验，因为每一个核心数据面都必须能够安全穿过 JSON/Bridge 抵达 TypeScript 和 Python 插件而不丢失精度！

---

## 6.3 13 个核心保留域（`RESERVED_DOMAINS`）与自定义扩展事件

每条事件的 `kind.key()` 格式统一为 **`domain/name`**（即序列化后的 `type` 和 `name` 字段）。Dim 将核心运行时占用的 13 个顶级域定义为 `RESERVED_DOMAINS`：

| 保留域 | 核心职责与典型事件示例 |
| --- | --- |
| `input` | 用户与插件输入生命周期：`input/message`、`input/applied`、`input/consumed`、`input/withdraw`、`input/deleted` |
| `stop` | 中断控制：`stop/interrupt`（作用域可为 `turn` 或 `branch`） |
| `approval` | 工具人工审批：`approval/request`、`approval/respond`、`approval/cancelled` |
| `permission` | 权限切换记录：`permission/changed` |
| `exec` | 后台 Shell 任务：`exec/started`、`exec/ended` |
| `turn` | 对话回合起止：`turn/start`、`turn/end` |
| `prompt` | Prompt 变更与配置握手：`prompt/changed`、`prompt/configure-request` |
| `context` | 上下文准备与压缩：`context/prepare`、`context/compact`、`context/compaction`、`context/budget` |
| `model` | 模型调用与流式输出：`model/call`、`model/start`、`model/thinking`、`model/text`、`model/tool-use`、`model/usage`、`model/output-end` |
| `step` | 工具调用执行：`step/tool-call`、`step/tool-start`、`step/tool-completed`、`step/tool-result` |
| `branch` | 时间线分支生命周期：`branch/create`、`branch/start`、`branch/resume`、`branch/end`、`branch/ended` |
| `collaboration` | 跨 Session 协作子智能体：`collaboration/request`、`collaboration/message`、`collaboration/result` |
| `session` | 会话级分叉记录：`session/forked` |

### 扩展事件（`ExtensionEvent`）的规矩
插件或二次开发新增的特性，**绝对禁止**往上述 13 个保留域里硬塞自定义事件（`ExtensionEvent::new` 会在构造期直接拦截报错）。
正确做法是使用**属于你自己的域名**（比如你的插件 ID：`goal/set`、`workflow/stage-done`、`my-plugin/audit`），要求：
- `domain` 和 `name` 非空，且不包含 `/` 或 `*`；
- 可以通过 `timeline.append(...)` 写入，然后由所有语言的 `dispatcher.consume` / `subscribe` 订阅读取；
- **最关键的一条上下文铁律（见 `AGENTS.md` & `principles.md`）**：
  > **任何事实想进入大模型的对话上下文，只有两条合法通道：通过 `input`（`input/applied` -> `input/consumed`）或者通过工具返回值（`ToolResult.content`）。**  
  > 扩展事件本身只用来记录“某件事发生了”并重建状态，绝对不要企图另造一套绕过 `input` 和 `ToolResult` 的上下文拼装后门！

---

## 6.4 写入机制：原子批提交与 `before` 同步拦截

在 [`crates/dim-agent-runtime/src/core/timeline.rs`](../crates/dim-agent-runtime/src/core/timeline.rs) 中，写入 Timeline 有两种非常强的原子保障：

### 1. 原子批提交（Atomic Batch）
如果一个操作必须同时产生两条不可分割的事实（比如 `dim-goal` 里设置目标时，既要写入 `goal/changed` 事实记录，又要紧接着塞一条唤醒分支的 `input/message`），可以用 **批提交（`append_batch`）**：
- 整个批次在同一个提交锁内拿到连续的 `sequence`；
- 要么全部落盘成功并依次派发，要么全部失败回滚，中间**绝对不会插入任何其他并发协程的事件**！

### 2. `before` 变换钩子（Rust 进程内同步中间件）
在提交锁内部、真正持久化落盘之前，Timeline 会通过私有的 `BeforeCommit` 通道运行 `before` 钩子：
- 这允许 Rust 原生部件（比如安全审查或敏感字段脱敏器）**在事实定型之前同步改写或校验 `EventDraft`**，并把每一步变换轨迹记录在 `TimelineTrace` 中；
- 之所以限定 **仅限 Rust 进程内同步执行**，是为了防止跨进程 IPC 阻塞住整个分支的写入锁。

---

## 6.5 多分支共用账本：祖先切分（Ancestors）与持久化查询

如果一条子分支从 `main` 的 `sequence = 50` 处分叉出去，难道要把主分支前 50 条事件全复制一遍写进数据库吗？
当然不用！Dim 的设计极其优雅：

```text
main 分支账本:       #1  #2  ...  #50  #51  #52 ...
                                  ▲
                                  │ 分叉切点 (ancestors: [("main", 50)])
sub-1 分支自身账本:                └─► #1  #2  #3 ...
```

1. **零拷贝分叉**：创建分支时只记录一张 **`ancestors` 切点链**（父分支 ID + 截至序号）。
2. **统一投影视图（`ContextHistory` & `TimelineQuery`）**：当你以 `scope: "branch"` 查询时，只能看到本分支事件；当你以 `scope: "lineage"`（或恢复上下文）读取时，`TimelineStore` 会自动沿祖先链拼装“截至 50 号的主线历史 + 子分支的新事件”，外加最近一次 **`ProjectionCheckpoint`（上下文压缩检查点）**，做到既省存储又秒级加载！
3. **内置安全查询 DSL（`TimelineQuery`）**：不需要把整个分支几万条 JSON 全拉到内存里自己 `filter`。Timeline 直接支持声明式过滤与聚合：
   - 按 `kind` 集合过滤（如 `op: "in", values: ["input/message", "turn/start"]`）
   - 指定 `through` 截止序号，配合 `include_publishing: true`（下一章揭秘为什么这个参数至关重要！）
   - 选择返回完整事件列表（`select: { kind: "events" }`）还是直接返回计数（`select: { kind: "count" }`），支持游标分页（`page: { limit, after }`）。

---

## 本章小结

这一章我们看透了 Dim 最稳固的地基：
- **Timeline = 不可变流水账本**，一切状态都必须能从上面重新算出来；
- **`sequence` 是唯一的秩序之王**；
- **13 个保留域不可擅闯**，扩展写自己的 domain；
- **入模上下文只有两条正门**（`input` 与 `ToolResult.content`）；
- **分支分叉靠 `ancestors` 切点指针**，既零拷贝又可独立演进压缩。

光有账本还不够，多个部件怎么在并发环境下消费这本账本而不乱套？下一章，我们解剖 Dim 最精妙的并发设计：**Dispatcher 与 `consume` 状态重建协议**！
