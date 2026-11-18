# 第 11 章：二次开发实战（二）—— 编写 Rust 原生 Capability 与 Feature Crate

> **本章目标**：掌握如何用 Rust 2024 编写高性能原生 Capability、共享服务和完整的 Feature Crate——覆盖从 `extension-a/b` 的最小双插件示范、到仿照 `dim-goal` 搭建跨作用域官方级功能模块、再到使用 `MinimalHost` 编写确定性集成测试。

---

## 11.1 两种 Rust 扩展形态：`Feature` vs `DynPlugin (install)`

在 Rust 层扩展 Dim 有两种常见粒度：
1. **轻量级 Feature（`Feature::new` + `require_feature`）**：适合直接往 `AgentBuilder` 里塞入一两个 Session/Branch 部件（不涉及 Process 级配置、自定义 Settings 源或复杂的共享跨作用域调度）。参考测试夹具 [`tests/extension-a`](../tests/extension-a/src/lib.rs) 与 [`tests/extension-b`](../tests/extension-b/src/lib.rs)。
2. **完整 Feature Crate（`install(builder)` + `PartRegistry`）**：如 [`crates/dim-goal`](../crates/dim-goal/src/lib.rs)、[`crates/dim-event-hub`](../crates/dim-event-hub/src/lib.rs)、[`crates/dim-agent-scheduler`](../crates/dim-agent-scheduler/src/lib.rs) 所用模式——作为 1 行 Process 插件安装，同时注册 Settings、RPC 管理方法，并通过 `PartRegistry` 自动挂载 Session 与 Branch 部件。

---

## 11.2 实战一：跨插件服务提供、订阅与 `StateCell`（拆解 `extension-a` & `extension-b`）

仓库的 [`tests/extension-a`](../tests/extension-a/src/lib.rs) 和 [`tests/extension-b`](../tests/extension-b/src/lib.rs) 是学习 Rust 扩展最纯粹的教科书例子。

### 第一步：声明服务并注册命名目录（Provider Crate）
假设我们提供一个分支级计数器 `Counter`：
```rust
use dim_agent_runtime::{Names, ServiceCatalog, BRANCH_GROUP, names};
use rutis::host::{BoxFuture, CtxHost, DynPlugin, Effect, Key, TypeKey};
use rutis::Ctx;
use std::sync::{Arc, atomic::{AtomicUsize, Ordering}};

#[derive(Default)]
pub struct Counter(AtomicUsize);

impl Counter {
    pub fn increment(&self) -> usize {
        self.0.fetch_add(1, Ordering::Relaxed) + 1
    }
}

// 1. 注册到服务目录（指定作用域组为 BRANCH_GROUP）
pub fn register_names(catalog: &mut ServiceCatalog) {
    names::register::<Counter>(catalog, Some(BRANCH_GROUP));
}

pub struct Provider {
    counter: Key<Counter>,
}

impl Provider {
    pub fn new(names: &Names<'_>) -> rutis::Result<Self> {
        Ok(Self { counter: names.key()? })
    }
}

impl DynPlugin for Provider {
    fn name(&self) -> &str { "ExternalProvider" }
    fn provides(&self) -> Vec<TypeKey> { vec![self.counter.erase()] }
    fn apply<'a>(&'a self, ctx: &'a Ctx) -> BoxFuture<'a, rutis::Result<Effect>> {
        Box::pin(async move {
            ctx.provide_at(self.counter.clone(), Arc::new(Counter::default()))?;
            Ok(Effect::Done)
        })
    }
}
```

### 第二步：消费服务、监听总线并用 `StateCell` 暴露可观测状态（Consumer Crate）
```rust
use dim_agent_runtime::{Dispatcher, Names, StateCell};
use rutis::host::{BoxFuture, CtxHost, DynPlugin, Effect, Key, TypeKey};
use rutis::Ctx;
use std::sync::Arc;

#[derive(Clone, Default, Debug, PartialEq, Eq)]
pub struct Snapshot {
    pub inputs: usize,
}

pub struct Consumer {
    dispatcher: Key<Dispatcher>,
    counter: Key<Counter>,
    state: Key<StateCell<Snapshot>>, // 通过 StateCell 暴露给宿主/测试观测
}

impl Consumer {
    pub fn part(names: &Names<'_>) -> rutis::Result<Box<dyn DynPlugin>> {
        Ok(Box::new(Self {
            dispatcher: names.key()?,
            counter: names.key()?,
            state: names.state_cell()?,
        }))
    }
}

impl DynPlugin for Consumer {
    fn name(&self) -> &str { "ExternalConsumer" }
    fn injects(&self) -> Vec<TypeKey> {
        // 凡是必需的依赖都必须声明在 injects() 中！否则未声明读取会被 rutis 严格拒绝
        vec![self.dispatcher.erase(), self.counter.erase()]
    }
    fn apply<'a>(&'a self, ctx: &'a Ctx) -> BoxFuture<'a, rutis::Result<Effect>> {
        Box::pin(async move {
            let dispatcher = ctx.require_at(&self.dispatcher)?;
            let counter = ctx.require_at(&self.counter)?;
            let state = StateCell::new(Snapshot::default());
            ctx.provide_at(self.state.clone(), Arc::new(state.clone()))?;

            // 若维护累计历史状态，生产环境建议采用 dispatcher.consume；无状态响应可用 subscribe
            dispatcher.subscribe(ctx, "ExternalConsumer", &["input/message"], move |_event| {
                let counter = counter.clone();
                let state = state.clone();
                Box::pin(async move {
                    state.set(Snapshot { inputs: counter.increment() });
                    Ok(())
                })
            })?;
            Ok(Effect::Done)
        })
    }
}
```

看看上面 `StateCell<Snapshot>` 的妙处：写好之后，在 Session 门面上直接就能用 `session.get::<Snapshot>("main")` 同步读取快照，或者用 `session.on::<Snapshot>("main")` 拿一个 `watch::Receiver` 实时订阅变化！

---

## 11.3 实战二：仿照 `dim-goal` 设计完整 Feature Crate

如果你想做一个像 `Goal` 或 `Scheduler` 一样完整的新功能（拥有自己的配置文件、Session 工具、Branch 独立状态机、并且需要写入原子事件批次），[`crates/dim-goal/src/lib.rs`](../crates/dim-goal/src/lib.rs) 是最佳模板：

### 架构骨架（1 个 Process Row + N 个 Session/Branch Parts）

```rust
pub fn install(builder: &mut AgentBuilder) {
    builder.plugin(|names| Ok(Box::new(MyFeaturePlugin::new(names)?)));
}

struct MyFeaturePlugin {
    settings: Key<Settings>,
    parts: Key<PartRegistry>,
}

impl DynPlugin for MyFeaturePlugin {
    fn name(&self) -> &str { "my-feature" }
    fn injects(&self) -> Vec<TypeKey> {
        vec![self.settings.erase(), self.parts.erase()]
    }
    fn apply<'a>(&'a self, ctx: &'a Ctx) -> BoxFuture<'a, rutis::Result<Effect>> {
        Box::pin(async move {
            // 1. 注册 Settings 配置 Schema（会自动出现在配置体系与校验中）
            ctx.require_at(&self.settings)?.register(
                ctx,
                SourceDeclaration {
                    source: "my-feature".into(),
                    user_file: "my-feature.json".into(),
                    project_file: None,
                    schema: serde_json::json!({
                        "type": "object",
                        "properties": { "enabled": { "type": "boolean" } }
                    }),
                    user_only: vec![],
                },
            )?;

            // 2. 用一张线程安全的 Weak 索引表串联同一插件下的 Session 与 Branch Part
            //    （无需专门为跨两层暴露一个公共 Service，见 ADR 002 D3）
            let shared_handles = Arc::new(Mutex::new(BTreeMap::new()));
            let parts = ctx.require_at(&self.parts)?;

            let handles_s = shared_handles.clone();
            parts.register(ctx, PartScope::Session, "MyFeatureSession", move |mount| {
                Ok(Some(Box::new(MySessionPart::new(mount.names, handles_s.clone())?) as Box<dyn DynPlugin>))
            })?;

            let handles_b = shared_handles;
            parts.register(ctx, PartScope::Branch, "MyFeatureBranch", move |mount| {
                Ok(Some(Box::new(MyBranchPart::new(mount.names, handles_b.clone())?) as Box<dyn DynPlugin>))
            })?;

            Ok(Effect::Done)
        })
    }
}
```

### 如何把新 Crate 装配到 `dimcode`？
1. 在根 [`Cargo.toml`](../Cargo.toml) 的 `[workspace.members]` 加入 `"crates/dim-my-feature"`；
2. 在 [`crates/dimcode/Cargo.toml`](../crates/dimcode/Cargo.toml) 添加依赖；
3. 在 [`crates/dimcode/src/main.rs`](../crates/dimcode/src/main.rs) 的 `install(builder)` 函数中加一行 `dim_my_feature::install(builder);`。

---

## 11.4 编写确定性 Rust 集成测试（`MinimalHost`）

在 Dim 里测试一个新 Capability 完全不需要真的连接大模型 API 或拉起整套 CLI！[`dim_agent_runtime::testing::MinimalHost`](../crates/dim-agent-runtime/src/testing.rs) 和 `dim_agent_capabilities::defaults::mock_hosts` 让你能在内存里秒级跑通完整的多分支并发测试：

```rust
use dim_agent_runtime::testing::MinimalHost;
use dim_agent_runtime::{AgentBuilder, AgentConfig, Feature, SessionConfig};
use dim_agent_types::{BranchSource, InputMode};

#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
async fn my_capability_isolates_branches() {
    let mut b = AgentBuilder::new(AgentConfig::default());
    // 1. 使用内置 Mock 宿主（含内存 TimelineStore 和可 scripted 的 Mock ModelHost）
    dim_agent_capabilities::defaults::mock_hosts(&mut b);
    dim_agent_capabilities::defaults::install(&mut b);

    // 2. 注册你的服务名并安装待测 Feature
    my_crate::register_names(b.catalog_mut());
    b.require_feature(
        Feature::new("MyFeature")
            .branch("Provider", |n| Ok(Box::new(Provider::new(n)?)))
            .branch("Consumer", |n| Consumer::part(n))
    );

    // 3. 启动轻量宿主并创建两个完全隔离的 Session
    let host = MinimalHost::start(&b).await.unwrap();
    let s1 = host.create_session(SessionConfig::default()).await.unwrap();
    let s2 = host.create_session(SessionConfig::default()).await.unwrap();

    // 4. 在 s1 的 main 分支提交输入并等待运行空闲
    s1.submit("hello".into(), InputMode::Queue, "main").await.unwrap();
    s1.idle("main").await.unwrap();

    // 5. 断言 s1 收到 1 次，s2 保持 0 次，证明分支与会话彻底隔离！
    assert_eq!(s1.get::<Snapshot>("main").unwrap().unwrap().inputs, 1);
    assert_eq!(s2.get::<Snapshot>("main").unwrap().unwrap().inputs, 0);

    host.close().await.unwrap();
}
```

---

## 本章小结

掌握了 `Names::key`、`DynPlugin`、`StateCell`、`PartRegistry` 以及 `MinimalHost`，你就真正具备了用 Rust 扩充 Dim 核心引擎的能力。

下一章，我们看另外三个同样高频的二次开发方向：**纯前端无构建 Web UI 扩展面板、自定义 JSON-RPC 接口、以及把 `dim-agent-runtime` 作为嵌入式引擎塞进你自己的 Rust 应用**！
