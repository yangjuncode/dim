# 第 12 章：二次开发实战（三）—— 扩展 UI、自定义 RPC 与嵌入 Runtime

> **本章目标**：掌握跨界二次开发的最后三块拼图——（1）零构建工具为 Web Workspace 增加定制侧边栏面板；（2）通过 `sessionMethods` / `managementMethods` 向外部暴露自定义 RPC 接口；（3）把 `dim-agent-runtime` 作为可嵌入引擎集成到你自己的 Rust 产品中。

---

## 12.1 实战一：零构建编写 Web 客户端 UI 扩展面板

在团队协作或特定垂类场景中（比如你想在浏览器工作台旁边实时显示数据库表结构、测试覆盖率看板、或自定义产物预览器），你**不需要修改和重新编译 `web/` 目录的 Vue 源码**！

按照 [`docs/design/client-ui-extensions.md`](../docs/design/client-ui-extensions.md) 与官方示例 [`examples/ui-extensions`](../examples/ui-extensions/README.md)，任何插件都可以直接声明一段浏览器端模块，由 Web 宿主自动放进一个隔离的 **`ShadowRoot`** 中挂载渲染。

### 1. 插件清单声明（`dim-plugin.json`）
哪怕是一个**没有任何后端代码（UI-only）**的纯展示插件，也只需要如下清单：
```json
{
  "name": "conversation-inspector",
  "ui": {
    "version": 1,
    "root": "ui",
    "panels": [{
      "id": "activity",
      "title": {
        "default": "Activity",
        "translations": { "zh": "会话状态" }
      },
      "entry": "panel.js",
      "placement": "session-panel",
      "requires": ["session.state", "session.history"],
      "methods": []
    }]
  }
}
```
> 注意：`ui.root`（即 `ui/` 目录）下的静态资源会被 App Server 的 `UiExtensions` 制作为不可变快照供浏览器加载，**切勿在 `ui/` 中存放后端源代码、私钥或配置文件**！

### 2. 面板模块（`ui/panel.js`）
你可以用原生 JS、Web Components 或自行打包任何前端框架，只需导出一个标准函数 **`mount(root, context)`**，并返回一个清理函数：

```js
/** @type {import('dimcode/ui').PanelModule['mount']} */
export function mount(root, context) {
  root.innerHTML = `<div class="box"><h3>加载中...</h3><pre></pre></div>`
  const title = root.querySelector('h3')
  const pre = root.querySelector('pre')

  // 1. 响应式感知宿主环境（语言、主题、当前 Session/Branch ID）
  context.observeEnvironment(env => {
    title.textContent = env.locale.startsWith('zh') ? '会话活动面板' : 'Session Activity'
  })

  // 2. 订阅当前分支运行状态（idle / running / stopping）
  context.services.session.state.observe(activity => {
    pre.textContent = `Branch: ${activity.branch.running ? 'running' : 'idle'}`
  })

  // 3. 也可以随时分页读取近期历史（或通过 context.services.session.call 调用插件自定义 Session 方法！）
  // const page = await context.services.session.history.read({ limit: 20 })

  // 4. 返回卸载清理函数；宿主销毁或切分支时同时会触发 context.signal.abort
  return () => root.replaceChildren()
}
```

### 3. 后端发图片/大文件给面板展示（`uiResources`）
如果你的后端插件生成了一张架构图或一段大型产物文件，千万不要把几十 MB 的 Base64 塞进 JSON-RPC！
- 后端 Session 部件注入 **`uiResources`**，调用 `publishFile(...)` 发布该文件句柄（`ResourceRef`）；
- 前端面板拿到引用后，直接调用 **`context.services.resources.url(ref)`** 拿到一个受 Cookie 鉴权的本地 HTTP 流式 URL，直接丢给 `<img>`、`<video>` 或 `fetch()` 即可！

---

## 12.2 实战二：对外暴露自定义 RPC 方法

无论是给你的 UI 扩展面板调用，还是给外部自动化系统控制，插件都可以通过两个标准注册表向 App Server 暴露 RPC 接口：

### 1. Session 级业务方法：`sessionMethods`
当插件处于 `Session` 作用域时，注入 `sessionMethods` 服务，注册针对**当前会话**的方法（见 [`runtime/plugins/workflow/index.ts`](../runtime/plugins/workflow/index.ts) 的真实用法）：
- **命名规范**：一般以 `<plugin-name>.<action>` 命名，避免与内置方法冲突；
- **Web 端按需放行**：在 `dim-plugin.json` 的 `ui.panels[].methods` 列表中显式声明面板需要调用的 Session 方法名，Web 面板就能通过 `context.services.session.call(method, params)` 安全调用，其他未声明的方法会被网关注销拦截；
- **流式通知**：对于耗时操作，还可以注入 **`callChannel`**，在单次 RPC 调用期间向发起方客户端推送实时进度更新！

### 2. 全局管理方法：`managementMethods`
当插件在 `Process` 作用域时，注入 `managementMethods`，可以暴露跨会话的全局管理接口（仅限本地受保护的 Unix Socket / Pipe 客户端调用，不直接暴露给 Web 页面）。

---

## 12.3 实战三：在自己的 Rust 程序中嵌入 `dim-agent-runtime`

如果你不想走 CLI 或 App Server 守护进程，而是想把 Dim 强大的 **Timeline + Dispatcher + 多协议模型路由 + 工具审批引擎**直接变成自己 Rust 服务（例如自研 IDE 后台、AI 运维机器人或桌面应用）的核心心脏，只需要引入两个核心 crate：`dim-agent-runtime` 与 `dim-agent-capabilities`！

参考 [`crates/dim-agent-runtime/src/testing.rs`](../crates/dim-agent-runtime/src/testing.rs) 中的 `MinimalHost`（也是官方推荐的嵌入范式）：

```rust
use dim_agent_runtime::testing::MinimalHost;
use dim_agent_runtime::{AgentBuilder, AgentConfig, SessionConfig};
use dim_agent_types::InputMode;

async fn embed_dim_in_my_app() -> anyhow::Result<()> {
    // 1. 创建装配器
    let mut builder = AgentBuilder::new(AgentConfig::default());

    // 2. 注入你自己的宿主端口实现（ModelHost / StorageHost / Platform）
    //    或使用 dim-agent-aimux 提供的真实多协议大模型驱动 + 自定义 Sqlite Store
    dim_agent_capabilities::defaults::install(&mut builder);
    // your_custom_host::install(&mut builder);

    // 3. 按需加装内置 Feature（如 Goal / CodeMode）或你自己的 Feature
    dim_codemode::install(&mut builder);

    // 4. 启动独立 rutis 微内核宿主并构建 Agent
    let host = MinimalHost::start(&builder).await?;

    // 5. 创建一个带专属工作目录的 Session，开始收发任务
    let session = host.create_session(SessionConfig {
        cwd: Some("/path/to/workspace".into()),
        ..Default::default()
    }).await?;

    session.submit("分析当前项目依赖".into(), InputMode::Queue, "main").await?;
    session.idle("main").await?;

    host.close().await?;
    Ok(())
}
```

### 如果你想深入定制 App Server 或 TUI 本体
- **定制终端界面**：核心代码全在 [`crates/dim-cli/src/ui/`](../crates/dim-cli/src/ui/)（`scrollback` 与 `terminal` 两大渲染目录），以及斜杠命令注册表。
- **定制协议与存储**：[`crates/dim-app-server`](../crates/dim-app-server/) 清晰划分了 `storage/`（SQLite 持久化）、`rpc/`（v3 方法路由）、`web_entry.rs`（HTTP/WS 网关）与 `plugins/`（插件生命周期管理器）。

---

## 本章小结

至此，三条主流二次开发路径已全线打通：
- **轻量化业务与工具**：TS / Python 热重载插件；
- **原生高性能或核心调度**：Rust `DynPlugin` / `Feature` Crate；
- **前端面板与内核嵌入**：`ShadowRoot` UI 扩展与 `AgentBuilder` 嵌入。

最后一章，我们来打磨工程师的基本功——学习 Dim 的**架构自动检查工具链（`xtask`）、E2E 测试策略、生产导出/录屏排障三板斧，以及“减法审查”工程文化**！
