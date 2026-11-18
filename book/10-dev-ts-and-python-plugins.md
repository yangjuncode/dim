# 第 10 章：二次开发实战（一）—— 编写 TypeScript 与 Python 插件

> **本章目标**：零 Rust 修改，直接通过外部插件扩展 Dim 的能力！手把手写出一个完整的 TypeScript 插件（带自定义工具、Settings 持久化、Branch 事件消费者、扩展事件与 Guide 条目）和一个 Python 3.12+ 插件，并掌握秒级热重载调试技巧。

---

## 10.1 环境准备与插件目录结构

外部脚本插件由 App Server 托管的 Node.js / Python 运行时负责执行（见 [`docs/typescript-plugin-authoring.md`](../docs/typescript-plugin-authoring.md) 与 [`runtime/plugins/plugin-authoring/skills/plugin-authoring/SKILL.md`](../runtime/plugins/plugin-authoring/skills/plugin-authoring/SKILL.md)）。

### 1. 检查或启用脚本运行时
```bash
# 查看当前 Node / Python / uv 运行时状态
./dev runtime list

# 按需安装/启用对应运行时
./dev runtime install node
./dev runtime install python
```

### 2. 一个标准插件目录长什么样？
```text
my-ts-plugin/
├── dim-plugin.json        # 插件清单（名称、版本、入口、运行时要求）
├── package.json           # 依赖 "@arcships/rutis": "0.19.3" 等（可选 TypeScript 类型 "dimcode"）
└── index.ts               # 插件主入口（Node 原生 strip-types 或 TS 直接运行）
```

看看官方内置插件 [`runtime/plugins/workflow/dim-plugin.json`](../runtime/plugins/workflow/dim-plugin.json) 的标准结构：
```json
{
  "name": "my-audit-plugin",
  "version": "0.1.0",
  "description": "记录分支事件统计并提供审计工具的演示插件",
  "main": "index.ts",
  "runtime": "node"
}
```
> **重要开发技巧（热重载陷阱！）**：  
> 当你调用 `plugin reload` 时，Node 运行时只会**重新加载入口 `index.ts`**，而不会自动清除它 `import` 的其他子模块的模块缓存！因此在**敏捷开发和频繁 `reload` 调试阶段**，建议把正在频繁改动的代码都放在 `index.ts` 里，等定型后再拆分子文件并重启守护进程测试。

---

## 10.2 实战一：编写全能型 TypeScript 插件

我们来做一个麻雀虽小五脏俱全的 **`audit-counter` 插件**，涵盖企业级扩展最常用的 **5 项能力**：
1. **Process 层注册配置源（`settings`）**：支持持久化和热更新配置；
2. **Branch 层消费事件流（`dispatcher.consume`）**：统计本分支一共发过几条用户消息；
3. **Branch 层写入自定义扩展事件（`timeline.append`）**：在合法的扩展域 `audit/checked` 记一笔；
4. **Session 层向大模型注册工具（`toolRegistry`）**：提供一个 `audit_summary` 工具让模型读取统计结果；
5. **Session 层注册输入提示说明（`guideEntries`）**：教会模型看懂我们注入的 `<audit-note>` 标记。

新建目录 `/tmp/my-audit-plugin`（或放到你自己的工作项目外独立文件夹），创建 `dim-plugin.json` 后，编写 `index.ts`：

```ts
import { definePlugin } from '@arcships/rutis'
import type { BranchEvent, Services } from 'dimcode/plugin'

// 每个分支私有的计数器服务名（供同一分支上的工具或其它部件读取，或直接保存在按 branchId 索引的 Map 中）
const MY_DOMAIN = 'audit'

// 1. 定义 Branch 级部件：每个分支打开时自动挂载一份实例
const auditBranch = definePlugin({
  name: 'AuditBranch',
  inject: ['dispatcher', 'timeline', 'timelineIdentity'],
  provides: { auditCounter: { snapshot: 'sync' } },
  async apply(ctx) {
    const dispatcher = ctx.use<Services['dispatcher']>('dispatcher')
    const timeline = ctx.use<Services['timeline']>('timeline')
    const identity = await ctx.use<Services['timelineIdentity']>('timelineIdentity').get()

    let messageCount = 0
    let lastCheckedSeq = 0

    const fold = (event: BranchEvent) => {
      if (event.type === 'input' && event.name === 'message') {
        messageCount++
      }
      if (event.type === MY_DOMAIN && event.name === 'checked') {
        lastCheckedSeq = Number(event.sequence)
      }
    }

    // 核心：用 dispatcher.consume 实现“历史重建 + 实时消费”无缝衔接
    await dispatcher.consume(
      'audit-branch-consumer',
      ['input/message', `${MY_DOMAIN}/checked`],
      async () => {
        const page = await timeline.query({
          scope: 'branch',
          include_publishing: true, // 必须为 true，防止发布窗口漏事件
          query: {
            filter: {
              op: 'in',
              value: { kind: 'field', path: 'kind' },
              values: ['input/message', `${MY_DOMAIN}/checked`],
            },
            select: { kind: 'events' },
          },
          page: { limit: 1000 },
        })
        if (page.result.kind !== 'events') throw new Error('expected events')
        for (const row of page.result.events) {
          fold(row.event)
        }
        return BigInt(page.through) // 返回本次折叠的截止 sequence
      },
      async event => {
        fold(event)
      },
    )

    // 暴露给同分支或诊断读取
    ctx.provide('auditCounter', {
      snapshot: () => ({
        branchId: identity.branchId,
        messageCount,
        lastCheckedSeq,
      }),
    })
  },
})

// 2. 定义 Session 级部件：每个会话挂载一份，注册工具和 Guide 条目
const auditSession = definePlugin({
  name: 'AuditSession',
  inject: ['toolRegistry', 'guideEntries', 'treeCommands'],
  async apply(ctx) {
    const tools = ctx.use<Services['toolRegistry']>('toolRegistry')
    const guides = ctx.use<Services['guideEntries']>('guideEntries')

    // 可选：教大模型理解你的运行时标记（若你通过 treeCommands.input 写了 <audit-note>）
    await guides.register('audit-note', {
      tag: 'audit-note',
      description: 'Represents an automated audit checkpoint injected by the audit plugin.',
    })

    // 注册大模型可直接调用的工具
    await tools.register(
      {
        name: 'audit_record_checkpoint',
        description: 'Record a custom audit checkpoint event on the current branch timeline.',
        inputSchema: {
          type: 'object',
          properties: {
            note: { type: 'string', description: 'Short summary of what was audited' },
          },
          required: ['note'],
        },
        source: 'extension',
        risk: 'session', // 或 'read' / 'write'，决定是否需要用户授权审批
      },
      async call => {
        const note = String((call.arguments as { note?: string }).note ?? '')
        // 注意：只有 content（纯字符串）会被大模型读到！
        return {
          content: `Audit checkpoint acknowledged for branch ${call.branchId}: ${note}`,
          structuredContent: { ok: true, branchId: call.branchId, note },
        }
      },
    )
  },
})

// 3. Process 级入口（一插件一行）：注册 Settings 并将部件交由 partRegistry 管理
export default definePlugin({
  name: 'my-audit-plugin',
  inject: ['partRegistry', 'settings'],
  async apply(ctx, _config) {
    const parts = ctx.use<Services['partRegistry']>('partRegistry')
    const settings = ctx.use<Services['settings']>('settings')

    // 注册一条持久化业务配置文档（存入 SQLite settings_documents）
    await settings.register({
      source: 'user/my-audit',
      scope: 'user',
      defaultDocument: { enabled: true, threshold: 10 },
    })

    // 注册 Session 与 Branch 部件蓝图
    await parts.registerDefinition('session', 'AuditSession', auditSession)
    await parts.registerDefinition('branch', 'AuditBranch', auditBranch)
  },
})
```

### 如果你想在工具或分支里真正写入一条自定义扩展事件？
只需要在 `Branch` 部件或通过拥有该分支 `timeline` 句柄的位置调用：
```ts
await timeline.append({
  type: 'audit',    // 绝对不能使用 13 个 RESERVED_DOMAINS！
  name: 'checked',
  data: { note: 'passed' },
})
```
只要写进去，它就会像原生事件一样被持久化到 SQLite、出现在 `dimcode export` 导出的全量时间线中，并触发所有订阅了 `audit/checked` 的 `dispatcher.consume` 消费者！

---

## 10.3 安装、启用与热重载调试工作流

写好插件后，千万别每次都重启整个终端，用下面这套极速迭代流：

```bash
# 1. 安装本地路径插件并启用（在隔离的 ./dev 环境中测试）
./dev plugin install /path/to/my-audit-plugin

# 2. 查看插件加载状态（确认不是 errored / pending）
./dev plugin list

# 3. 修改 index.ts 后，一行命令热重载！
./dev plugin reload my-audit-plugin

# 4. 打开交互终端，直接问模型："请调用 audit_record_checkpoint 工具记录一次检查"
./dev
```

在 TUI 界面里还可以输入 **`/plugins`** 随时查看插件状态，或者用 **`/reload-plugins`** 一键热重载所有外部插件。

---

## 10.4 实战二：编写 Python 3.12+ 插件

Dim 对 Python 插件给予了一等公民同等待遇！仓库里的 [`runtime/fixtures/python/`](../runtime/fixtures/python/python_plugin.py) 展示了极其简洁的官方写法（要求 Python 3.12+）：

### 1. 插件目录结构
```text
my-py-plugin/
├── dim-plugin.json        # "main": "plugin.py", "runtime": "python"
├── plugin.py              # Process 入口
├── session_part.py        # Session 级部件（注册工具）
└── branch_part.py         # Branch 级部件（consume 消费事件并回写输入）
```

### 2. Process 入口：`plugin.py`
```python
import session_part
import branch_part

inject = ["partRegistry"]
provides = {}

async def apply(ctx, config):
    parts = ctx.use("partRegistry")
    await parts.registerDefinition("session", "PySessionPart", session_part.part)
    await parts.registerDefinition("branch", "PyBranchPart", branch_part.part)
```

### 3. Session 工具部件：`session_part.py`
```python
import json
from rutis import define_plugin

async def apply(ctx, config):
    session = await ctx.use("sessionIdentity").get()

    async def execute(call, cancelled):
        text = call["arguments"].get("text", "")
        return {
            "content": json.dumps({
                "session": session["sessionId"],
                "branch": call["branchId"],
                "echo": text,
            })
        }

    await ctx.use("toolRegistry").register({
        "name": "py_echo",
        "description": "Echo text from Python plugin",
        "inputSchema": {
            "type": "object",
            "properties": {"text": {"type": "string"}},
        },
        "source": "extension",
        "risk": "session",
    }, execute)

part = define_plugin(apply, inject=["sessionIdentity", "toolRegistry"], name="PySessionPart")
```

### 4. Branch 事件消费与回写部件：`branch_part.py`
```python
from rutis import define_plugin

async def apply(ctx, config):
    dispatcher = ctx.use("dispatcher")
    timeline = ctx.use("timeline")
    tree = ctx.use("treeCommands")
    identity = await ctx.use("timelineIdentity").get()
    branch_id = identity["branchId"]
    state = {"notified": False}

    def step(event, live):
        key = f"{event['type']}/{event['name']}"
        # 当模型第一回合结束时，且此前尚未触发过，写一条提示消息进输入队列
        if live and key == "turn/end" and not state["notified"]:
            state["notified"] = True
            return tree.input(branch_id, {
                "blocks": [{"type": "raw", "text": "<py-note>First turn finished!</py-note>"}],
                "mode": "queue",
            })
        return None

    async def on_live(event):
        coro = step(event, True)
        if coro is not None:
            await coro

    async def rebuild():
        cutoff = int(await dispatcher.committed())
        # 分页查询并折叠截至 cutoff 的历史事件（只改 state，不执行 tree.input！）
        # ... 参考 runtime/fixtures/python/python_branch_part.py ...
        return cutoff

    await dispatcher.consume("py-branch-watcher", ["turn/end"], rebuild, on_live)

part = define_plugin(
    apply,
    inject=["dispatcher", "timeline", "timelineIdentity", "treeCommands"],
    name="PyBranchPart",
)
```

请注意 Python 版同样严格遵守我们在第 7 章学到的黄金法则：
- **`rebuild()` 阶段绝不发副作用**（只在 `live == True` 时才调用 `tree.input`）；
- **用 `dispatcher.consume`** 返回折叠截断点 `cutoff`，防止遗漏或重复。

---

## 本章小结

无需编译一秒钟 Rust 代码，你已经学会了用 **TypeScript** 和 **Python** 自由扩展 Dim 的工具集、设置存储、自定义扩展事件和分支自动化行为！

但如果你追求极致零开销、需要访问 Rust 进程内独占接口（如 `before` 同步拦截钩子）、或者想构建一个内置于发行版中的大型原生 Capability，下一章我们将直接进入 **Rust 原生 Capability 与 Feature Crate 开发实战**！
