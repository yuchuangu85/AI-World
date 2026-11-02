# DeepSeek Harness（dsh）框架本体 — 事实底稿

> 用途：供 Lead 整合「dsh 框架本体 × Super-agent 自治架构」笔记时直接引用。
> 采集时间：2026-11-01。采集方式：`raw.githubusercontent.com` 直取单文件 + `module-graph.zh.md` 反查包目录（GitHub REST API 匿名额度已耗尽，未使用 contents API）。
> 约定：标 **【事实】** 的来自我实际读到的文档/源码锚点；标 **【推论】** 的是我的判断，不是文档原话。读不到的一律写「未取到」。

---

## 0. 仓库元信息（已核实）

| 项 | 值 | 锚点 |
|---|---|---|
| 仓库 | `deepseek-ai/deepseek-harness`，默认分支 `master` | raw 拉取路径即 master |
| 定位 | "DeepSeek Harness: Everything is a Plugin." 开源 agent harness | `README.zh.md` |
| 许可证 | MIT | `README.zh.md` §许可证；shields.io `/github/license` 返回 `MIT` |
| Star | `267k` | shields.io `/github/stars` 返回 `value: "267k"`（REST API 已限流，未能取精确整数） |
| 版本 | 根 `package.json` `@deepseek-ai/dsh-root` = **0.2.2-alpha.2** | `package.json` |
| 包管理器 / 运行时 | `pnpm@11.28.5`，`engines.node >= 24.9.0` | `package.json` |
| 状态 | 开发者预览（Developer Preview），「未来将出现破坏兼容性的变更」 | `README.zh.md` §开发者预览 |
| 底层框架 | Cordis（以 vendor 方式引入，`vendor/`） | `docs/cordis-primer.zh.md` |
| 设计论文 | *A Programming Paradigm for Spatiotemporal Composability*（arXiv 2608.25512） | `README.zh.md` |
| 规模（包目录） | `packages/<group>/<pkg>` 形式被 module-graph 枚举出 **372** 个包目录 | `docs/module-graph.zh.md` |
| 规模（服务 key） | capability-seams 图里注册到 `ctx.*` 的服务 **108** 个 | `docs/capability-seams.zh.md` |
| 规模（工具） | tool-catalog 收录 **35** 个工具包、**76** 条工具 schema（含同名工具在不同包的两种实现） | `docs/tool-catalog.zh.md` |
| 规模（npm scripts） | 根 `scripts` 共 **204** 条 | `package.json` |

**顶层目录**（`AGENTS.md` §Repository layout）：`vendor/`（vendor 的 Cordis）、`packages/`（`@deepseek-ai/dsh-<pkg>`，按 group 分组）、`apps/`（`cli` / `web` / `desktop`）、`python/`、`native/`、`benchmarks/`、`.agents/`、`docs/`、`scripts/`、`website/`。

**apps/ 三个应用**（各自 `package.json` 均 0.2.2-alpha.2）：

| 包名 | 目录 | 职责（原文摘要） |
|---|---|---|
| `@deepseek-ai/dsh` | `apps/cli` | "dsh CLI: profile launch, plugin management, and configuration inspection"——**唯一受支持的 Node 应用启动器** |
| `@deepseek-ai/dsh-web-frontend` | `apps/web` | "Web application entry: vite build over the @deepseek-ai/dsh-client-web shell lib" |
| `@deepseek-ai/dsh-desktop` | `apps/desktop` | "Electron desktop shell for a bundled dsh runtime and external plugins" |

---

## A. Cordis 与「一切皆插件」的确切含义与粒度

### A1. 五个核心概念【事实】

`docs/cordis-primer.zh.md` §五个核心概念 原文五条：

1. **插件是实现 Service 的对象**。可以是带可选 `inject` 和 `apply(ctx)` 字段的**函数**，也可以是 `Service` 子类；生命周期由 Cordis 挂载到当前上下文。
2. **上下文是服务的容器**。一个服务占据稳定 `ctx.<key>`（如 `ctx.tools`、`ctx.llm`、`ctx.sessions`）；其他插件**按 key 查找服务，而非导入具体实现**。
3. **通过 `inject` 声明服务依赖**。声明后等待依赖就绪才启动；**加载顺序由服务依赖表达，而非手动编排启动序列**。
4. **类型化事件用于通信**。TypeScript 声明合并注册事件名，然后以 `emit` / `waterfall` / `parallel` / `serial` / `bail` 分发。
5. **注册是可逆的副作用**。提示词片段、工具 schema、适配器、提供方、监听器通过 `ctx.effect()` 或 `ctx.on()` 安装，**reload 与 teardown 时按预期撤销**。

五种分发模式（`docs/cordis-primer.zh.md#dispatch-modes`）：`emit`（不 await、无返回值，纯观察）、`waterfall`（不 await、有返回值，**环绕中间件**，监听器收 `(...args, next)`，不调 `next()` 即短路）、`parallel`（await、并行、无返回值）、`serial`（await、按注册顺序、有返回值）、`bail`（不 await、停在首个返回值）。**分发模式是事件公开约定的一部分**，新事件用 `@mode` 标签记录以便生成目录交叉校验。

### A2. 「一切皆插件」到底指什么粒度【事实】

`docs/architecture.zh.md` 开篇 §Cordis 的原文（这是全仓最精确的一句）：

> "产品的**每一部分**都是插件，包括**模型适配器、工具注册表、会话日志，以及 agent loop 本身**，因此**每个都可以从配置替换**。"
> "**不存在需要打补丁的特权内核**：扩展 dsh 的方式是把插件挂载到其他插件旁边，而各项注册都是副作用，会在其插件卸载时撤销。"

所以粒度不是「工具/模型/UI/渠道」这种产品功能切分，而是**「一个 `ctx.<key>` 服务 = 一个可替换 seam」**。可核实的替换面清单（架构文档 §新行为的归属位置 表格 + §事件 + §能力 seam）：

- 模型提供方 → 在 `ctx.llm` 注册适配器（`packages/llm/*`：`llm`、`llm-deepseek`、`llm-pi-ai`、`llm-retry`、`token-meter`）
- 工具 → 在 `ctx.tools` 注册，schema 自动进提示词组装
- 会话日志 → `ctx.sessions`（`packages/core/session`）
- **agent loop 本身** → `ctx.agentLoop`（`packages/core/agent-loop`）；文档明确：**扩展插件依赖 `agent`，绝不直接依赖 `agent-loop`，因此循环保持可替换**（`docs/subsystems/core.zh.md`）。`AgentFactory` 由循环通过 `ctx.agents.setFactory()` 注册，消费方只认 `ctx.agents`。
- 文件系统 / 进程 / 沙箱 / shell / 终端 → `ctx.fs`、`ctx.subprocess`、`ctx.sandbox`、`ctx.shell`、`ctx.terminals`
- 持久化 → 实现 `SessionPersistence`（`create`/`open`/`stat`/`list`/`export`）即可换后端（`packages/session/session-persistence-jsonl` 是内置实现）
- UI / 编辑器集成 → 驱动 `ctx.agents` 并从 `session/event` 渲染

**【事实】唯一「不是服务」的核心包**：`packages/core/scope` 是零依赖库（`createScope`/`scopeOf`/`scopeTarget`），在模块图中位于 `session/` 与 `system-prompt/` 之下，**正是为了让它们消费它而不形成环**（`docs/subsystems/core.zh.md` §主干逐包速览）。→ 说明「无特权内核」不等于「没有分层」，仍有刻意设计的底层依赖方向。

### A3. seam 的精确定义【事实】

`docs/glossary.zh.md#capability-seam`：seam 是**含三种角色的可替换能力**——
- **Service Definition**：拥有自身 `ctx.<key>` 与词汇类型的 Cordis `Service`（**可以是抽象类或具体注册表，绝不是 TypeScript `interface`**）
- 一个或多个 **Service Provider**
- 一个或多个注入该服务的 **Consumer**（通常是面向模型的工具）

规范范例 `packages/shell`：`dsh-shell`（Definition）+ `dsh-bash-local` / `dsh-bash-sandbox`（Provider）+ `dsh-tool-bash`（Consumer）。文档强调：**「seam 是完整能力，绝不是其中一个角色」**。

### A4. 热插拔的三种落地方式【事实】

1. **disposer 机制**：每个注册都要有 disposer，「要么从 `ctx.effect()` 返回一个，要么用 Cordis 辅助方法自动处理」；若 teardown 顺序有要求，相关工作要放进同一个 effect（`docs/cordis-primer.zh.md` §实践规则）。
2. **HMR 插件**：`packages/boot/hmr`（`ctx.hmr`，"Serialized module and configuration reloads"）。**YAML 控制是否启用**：`dsh-base` 启用「仅监视配置的」`dsh-hmr`；Headless / SDK / ACP **禁用它**；`sdk-minimal` **不包含它**（`docs/architecture.zh.md` §Profile 与组合包）。
3. **插件管理**：`packages/boot/plugin-manager`（`ctx.pluginManager`）+ `dsh plugin --profile <name> <pnpm args>`，或模型工具 `plugin_manager`；保存的 profile 变更返回 `restart-required` / `no-hmr`（`packages/bundle/sdk-app/README.zh.md`）。

**【推论】** 所谓「一切皆插件」在工程上是「一切能力都注册在共享 `ctx` 上、都可以按 key 被后挂载的插件遮蔽或替换」，而**不是**「插件能跨越进程边界或运行期任意热替换一切」——HMR 默认只覆盖配置文件与模块，且 headless/sdk/acp 明确关掉。

---

## B. profile + bundle + patch 的组装机制

### B1. 三层概念【事实】

- **profile**：存放在 `$DSH_HOME/profiles/<name>` 的具名组装。含 `package.json`（声明 `dsh.profile.bundles` 有序组合包列表 + 树外插件依赖）与 `cordis.patch.yml`（用户 patch 层）（`docs/architecture.zh.md` §Profile 与组合包；`apps/cli/README.zh.md` §Profile）。
- **组合包（bundle）**：Cordis 配置项及其挂载代码的分发格式，`package.json` 里用 `dsh.bundle` 指向一个 patch 文件或**有序文件列表**（`bundlePatchFiles` 校验、`bundlePatchPaths` 解析）（`packages/boot/app-boot/README.zh.md#profiles`）。
- **patch**：按 `id` 定位某个条目并**替换其整个 config**，或 `insert` 新条目。**不做深度合并**——「每个 patch 条目会替换目标的整个配置，因此请重述每个想保留的设置」（`packages/bundle/base/README.zh.md` §更改默认值）。`preset` patch 在目标声明的 `config.plugins` 内应用相同操作。

### B2. 层序（从空到运行）【事实】

`docs/architecture.zh.md` §Profile 与组合包 + `apps/cli/README.zh.md` §Profile 一致给出：

```
空条目列表
 → profile 中有序的组合包（dsh.profile.bundles）
 → profile 自身 cordis.patch.yml
 → home 级 $DSH_HOME/cordis.patch.yml
 → --patch overlay
```

后层按 id 覆盖行，**每行最后一次写入生效**。

### B3. 五个随附 profile 与六个组合包【事实】

profile → bundle 组合（`docs/architecture.zh.md`、`apps/cli/README.zh.md`、各 bundle README）：

| profile | 组合包栈 | 形态 |
|---|---|---|
| `web` | `dsh-base` + `dsh-web-app` | 浏览器 GUI，默认 `http://127.0.0.1:3080`，启用配置 HMR |
| `headless` | `dsh-base` + `dsh-headless` | 一次性命令行任务，打印最终答案后退出；**不打开端口、不留后台运行物**；HMR 禁用 |
| `sdk` | `dsh-base` + `dsh-sdk-app` | JSON-RPC stdio，供 TS/Python SDK；HMR 禁用 |
| `acp` | `dsh-base` + `dsh-acp-app` | 纯自动化 ACP stdio（Agent Client Protocol v1）；HMR 禁用 |
| `sdk-minimal` | 仅 `dsh-sdk-minimal` | **刻意例外**：一棵完整显式配置树，**不应用 `dsh-base`**；不含 HMR |

**`dsh-base` 的定位（关键事实）**：
- 它是 `web`/`headless`/`sdk`/`acp` 的共享第一层：模型适配器、工具、持久化、沙箱与审批策略、设置、凭据、遥测。
- 它**本身是一份静态 patch 文档**：「一个应用到空 profile 根之上的 `insert` 列表。它**不挂载任何服务、不发出任何事件、也不持有任何可变状态**；每条插入行所属的包负责该行的行为与不变式」。
- 它是**组合包不是库**（"本包不是供导入的库"）。
- 平台门控写在 patch 自身：`bash-sandbox`/`tool-bash` 带 `disabled: !!js process.platform === 'win32'`，孪生 `pwsh-sandbox`/`tool-pwsh` 仅 win32 挂载（`packages/bundle/base/README.zh.md` §平台门控）。
- **`sdk-minimal` 的取舍**（可作「最小内核包含什么」的直接证据）：默认公布按平台的持久 shell、`read_image`、`working_directory`，会话持久化为**未压缩 JSONL**，模型由 SDK 初始化请求选择；**排除** `dsh-base`、Web、settings、托管凭据、遥测、文本文件工具、workspace 指令、skill、jobs、subagent。其 `danger-full-access` 策略允许 shell 修改进程可访问的任何路径。

### B4. `--dump-config` 与启动审计【事实】

- `dsh --profile web --dump-config` 打印**生效配置树**（原始 `!!js` 表达式 + 按源文件与 patch 层分组的注释），输出是一份可加载的 YAML 文档；未匹配到任何行的 patch 会连同层标签一起报告。另有 `--dump-default-config`（模板）与 `--dump-config-schema`（导入每个组合模块读 schema，**只应对已信任插件的 profile 运行**）（`apps/cli/README.zh.md`、`packages/boot/app-boot/README.zh.md` §预览生效配置）。
- **required entry 全局列表**（含 `agent-loop`、`tool-executor-native`、`tool-executor-ptc`、`webserver`、`web-runtime`、`modules`、`connection`、`headless-runner`、`acp`、`sdk-jsonrpc-server`）：这些条目启动失败会**拆卸整个应用并以非零码退出**；optional 条目失败只警告并继续（`packages/boot/app-boot/README.zh.md#startup-and-reload-failures`，含 13 种失败模式的完整矩阵）。
- 版本兼容性：导入插件前比对 `engines.dsh` 或 `@deepseek-ai/dsh*` 的 peer 范围与运行时版本；不兼容插件需用户显式精确版本豁免（存于 profile 的 `compatibility.json`，**不**存 `package.json`）。被拒插件**永不导入其模块**。
- 组合包可声明 `dsh.bundle.platforms`（如 `["darwin"]`），Host 平台不支持则整层跳过并列入 `skippedBundles`。

**【推论】** profile/bundle/patch 是「**配置即构成**」的组装模型：没有代码层的 `if (profile === 'web')` 分支，只有按 id 覆盖的 patch 层。这让「同一 launcher 出多种产品形态」成为纯粹的声明式操作，也让 Lead 可以用「一份 base + 按需覆盖」的方式描述 dsh 的产品矩阵。

---

## C. core 包职责表与 turn/step 事件流

### C1. 核心包表【事实】

`docs/architecture.zh.md` §核心包（原表照录）：

| 包 | 职责 | `ctx` 键 |
|---|---|---|
| `core/session` | 仅追加的 `SessionEvent` 日志和内存存储 | `ctx.sessions` |
| `core/system-prompt` | 提示词片段与工具 schema 的组装 | `ctx.systemPrompt` |
| `core/tools` | 作用域化的工具注册表和带把关的执行流水线 | `ctx.tools` |
| `core/agent` | `Agent` 接口、活跃 agent 注册表和 `agent/*` 事件 | `ctx.agents` |
| `core/agent-loop` | 实现该接口的默认驱动器 | `ctx.agentLoop` |
| `core/scope` | 按 agent 划分作用域的注册原语 | **库，无 ctx 键** |
| `llm/llm` | 消息与流式词汇表，以及适配器 seam | `ctx.llm` |

补充（`docs/subsystems/core.zh.md` §主干逐包速览）：一轮次按同一条循环流经**六个包**——`agent-loop` 的 driver 认领排队的提示词 → 在 `session` 日志上开 turn → 经 `system-prompt` 组装请求前缀并从日志派生历史 → 经 `llm` seam 流式取响应 → 经 `tools` 注册表分发工具调用 → 把每个模型可见事实追加回日志供下一步派生。

### C2. turn / step 定义【事实】

`docs/glossary.zh.md`：
- **步骤（step）**：一次模型请求，以及由模型响应引发的工具执行。
- **轮次（turn）**：会话中一次对已接纳输入的排空过程；在模型及其工具停止工作或终止策略介入后结束。**一个轮次包含零个或多个步骤**。
- **Round**：承载一个轮次的外层策略迭代（如一个 Goal Round 或一次 Ralph 尝试）；**Round 计数器归该策略所有，并不统计会话中的每个轮次**。
- **交互段（exchange）**：连续的轮次，从首个步骤接纳真人输入的轮次起，到下一个这样的轮次为止；被通知（子 agent 完成、Goal Round、定时任务）唤醒的轮次**归入前面的交互段**。

### C3. turn-flow 主循环【事实】

`docs/architecture.zh.md#turn-flow` 的伪代码（关键骨架）：

```
turn/start
  claim next-step input plus one queued message
  snapshot Session model selection; assemble prompt sections + step-admitted tool schemas
  -> agent/pre-step; project runtime context at fallback
      reject | enter(messages, startsRequestSeries?)
  step/start
  agent/request -> prepareCall（取消则 system 与 user 都不提交）
  reconcile system/message using the prepared call capability
  append entered messages as user/message
  log request/header and request/context as needed
  derive and freeze model history from the log
  stream the bound prepared call -> llm/stream -> agent/assistant-stream start
      assistant/message | assistant/attempt -> agent/assistant-stream end
  tool/call* -> tools/pre-execute -> tools/execute -> tools/post-execute -> tool/result*
  step/end
  tools owe another request, or next-step input arrived -> claim -> next step
  -> agent/turn-stopping
turn/end
```

**事件三域划分【事实】**（`docs/architecture.zh.md#events`）：
- **会话事件**：追加到日志并通过 `session/event` 广播的**持久事实**——`turn/*`、`step/*`、`system/message`、`user/message`、`assistant/message`、`assistant/attempt`、`tool/*`。「当某个事实必须**在重新加载后仍然存在**时使用它。」
- **Agent 事件**（`agent/*`）携带活跃 `Agent`：inbox、步骤、状态、请求、验证、续跑。「要**观察或拦截进行中的工作**时使用它。」可枚举到：`agent/created`、`agent/disposed`、`agent/status`、`agent/error`、`agent/pre-step`、`agent/request-error`、`agent/turn-stopping`、`agent/assistant-stream`、`agent/inbox/{inserted,claimed,discarded}`。
- **能力事件**：无需导入循环即可向某个 seam（`fs/*`、`tools/*`、`telemetry/*`）附加策略与适配器。

**分发模式【事实】**：`agent/pre-step`、`agent/request`、`llm/stream`、三个 `tools/*` 是 **waterfall**（监听器必须调 `next()` 才能委托下去）；`agent/turn-stopping` 是 **serial**，没有 `next()`。

### C4. Agent 状态与句柄【事实】

- **`AgentStatus = 'idle' | 'running'`**（`docs/subsystems/core.zh.md:158`）。`agent/status` 事件描述「`idle` ⇄ `running`；唤醒投递在预留取消后**同步**进入 `running`；`idle` 表示没有 driver 被调度或活动」。
- `ctx.agents.create()` / `ctx.agents.resume()` 返回 **`AgentHandle`**；`dispose()` 是能力（capability）：「只有持有者能拆除该 agent」，会停循环、等退出、注销、从 store 移除会话、展开作用域世界（`packages/core/agent/README.zh.md`）。
- 对话驱动三个动词：`followup()`（排队下轮提示词并唤醒）、`steer()`（提交下一步输入并唤醒）、`inject()`（加面向模型上下文但**不唤醒**，落在下一个被接纳的步骤中）；`cancel(cause)`、`whenIdle()`。
- **`setup(agentCtx, agent)`** 创建时隙：scope 与 agent 对象已存在，但 agent/会话尚未发布、`agent/created` 未触发、首次提示词未组装；**setup 只做注册，从不驱动 agent**（`docs/glossary.zh.md#agent-scope`）。

### C5. 会话日志即唯一真源【事实】

- `core/session`：「在仅追加的会话日志中记录每个模型可见事实，并从该记录派生模型历史」；`deriveMessages()` 把日志投影为 `Message[]`，增量且有缓存；**压缩（compaction）只隐藏被取代条目，不删除**；**若不叠加持久化后端，会话只留在内存**（`packages/core/session/README.zh.md`）。
- 原则：**「模型可见即已记录」**——每个模型请求都必须能从日志重建；新增模型可见输入需要会话事件（`docs/architecture.zh.md` §会话日志）。
- **持久化格式版本**：`SESSION_FORMAT_VERSION` 是唯一手工维护的写入器版本号；定稿基线 `latestFinalizedVersion: 4`，发布记录 `latestReleasedVersion: 4`，证据标签 `dsh-v0.2.0-rc.2`（`docs/session-format-status.zh.md`）。物理布局：**JSONL v0 用 `session.jsonl[.zstd]`，v1 起用 `session.vN.jsonl[.zstd]`；已提交代际路径绝不重命名、替换或删除**，每个相邻迁移包（`session-format-v0-to-v1` … `v3-to-v4`）只负责一步 `vN → vN+1`（`docs/architecture.zh.md` §会话日志）。
- **投影 seam**：`dsh-session-projection` 提供 `ctx.sessionProjections`，已注册单元折叠已提交事件，载体通过 `snapshot()` 读客户端视图。

---

## D. 多 agent 原语：subagent / Agent Teams / workflow / ralph / goal

`packages/` 下相关的包目录（`docs/module-graph.zh.md`）：`subagent/*` 9 个、`workflow/*` 3 个、`goal/*` 4 个、`schedule/*` 2 个、`jobs/*` 2 个、`experimental/` 中 `agent-team`、`agent-team-profile`、`client-ui-agent-team`、`tool-agent-team`、`subagent-oar`、`tool-subagent-oar`、`ralph-bundle`、`tool-ralph`、`webhook`、`webhook-github`。

### D1. 谱系与 lineage 的定位【事实】

- **scope 模型只有两层、扁平**：贡献要么全局，要么归属恰好一个 scope key；**带作用域的注册不会向下继承给 subagent；子树行为通过 lineage 数据表达，从不通过 scope 结构**（`docs/glossary.zh.md#agent-scope`）。
- **lineage**：`parentSession`、持久 `delegationDepth`、运行时 `subagentDepth`；**从不影响可见性**。

### D2. subagent【事实】

- 统一 activation API：**所有后端都使用 `startActivation`**；`prepareContinuable` 选本地 Agent 创建，`start` 选一次外部执行；模型侧工具采用 **parent 回传**；Workflow 与 code mode 消费者可以等 caller 结果、不额外向父 agent 注入消息（`docs/subsystems/subagent.zh.md`）。
- 内建后端（包名即语义）：`subagent-spawn-in-process`、`subagent-fork-in-process`、`subagent-claude-code`、`subagent-codex`、`subagent-dsh-sdk`、`subagent-acp`。
- 能力声明 `SubagentCapabilities`：`agentOptions` / `agentPreset` / `outputSchema` / `depthLimit`——每个请求选项在 prepare 阶段先校验。
- 交付策略 `delivery: 'parent' | 'caller'`；本地子会话**总是**进入父级 catalog。
- 模型工具：`subagent`（默认名，可由 `toolName` 改名）与发布别名 `subagent_fork`；控制工具 `send_message` / `interrupt_agent` / `list_agents`（`packages/tool-subagent-control`）。
- **v0.2.2 的 `subagent_oar`（实验性统一委派入口）**：`packages/experimental/subagent-oar` 内嵌 **OAR 库**（`@botiverse/oar@0.67.0`，见 `pnpm-workspace.yaml` patches），无需单独安装 OAR CLI；配 `tool-subagent-oar` 后模型看到**一个** `subagent_oar` 工具，在部署配置的 provider 别名（如 `oar-codex` / `oar-claude`）之间选择运行时。Codex/Claude Code 使用已安装的原生 CLI、认证与设置。→ **【推论】** 这是把「多后端委派」从「多个工具名」收敛为「一个工具 + 运行时选择」的接口统一动作。

### D3. Agent Teams（实验性）【事实】

- 通过**显式启用的 `ctx.agentTeams`**，在「可继续 subagent」之上提供**持久 roster、任务状态和直接 inbox 消息协作**（`docs/architecture.zh.md` §能力 seam）。
- **隐式 Root Team**：`TeamId` 是使用独立品牌类型的 **root `SessionId`**；`TeamTaskId` 按 `task-<n>` 分配；teammate 名称是**不可变标签**（`docs/subsystems/agent-team.zh.md`）。
- 成员生命周期：每个 member 从 `provisioning` 开始，只到达一个终态 **`active` 或 `failed`**；roster 的 `running`/`inactive` **单独派生**，绝不重写该记录。
- 消息：新发送直接进目标 inbox 并返回 `MessageId`；**每条消息都尝试 Steer 投递**——running target 在最近步骤边界收到，inactive target 在已加载时启动一个轮次、否则**冷恢复**；调用方不能选其他模式。
- 共享任务 DAG：`TeamTaskSnapshot` 每条事件存完整快照，`revision` 是 **CAS 值**；`blockedBy` 边须无环；**`writeScopes` 是规范化的提示性路径前缀，不是锁**；状态 `pending` / `in_progress`(带 owner) / `completed` / `deleted`(tombstone)。
- 服务 API：`membership`、`listMembers`、`spawnTeammate`、`sendMessage`、`createTask`/`getTask`/`listTasks`/`updateTask`、`waitForChange(caller, timeoutMs, signal)`（**10 秒–1 小时的有界等待**）、`interrupt`、`tryMembership`。
- 9 个模型工具：`spawn_teammate`、`send_message`、`interrupt_agent`、`list_agents`、`wait_agent`、`team_task_create`、`team_task_get`、`team_task_list`、`team_task_update`。
- **随产品发布的 `dsh-base` 默认禁用该包**；需 profile patch 启用，并同时禁用旧 continuable child 的同名控制工具（`docs/tool-catalog.zh.md` 工具包映射）。
- 限制（`packages/experimental/agent-team/README.zh.md`）：**不支持独立工作目录、跨进程协调和自动释放任务 owner；需要持久会话存储**。

### D4. workflow【事实】

- `ctx.workflowEngine.start({ script, meta, args?, parent, signal? })`：运行一段**纯 JavaScript（不是 TS）编排脚本**，顶层 `await`、以 `return <json-value>` 结尾，把工作扇出给 subagent，返回脚本最终 JSON 值（`packages/workflow/workflow/README.zh.md`）。
- 脚本钩子：`agent(prompt, opts)`、`parallel(thunks)`、`pipeline(items, ...stages)`、`phase(title)`、`log(msg)`。
- 模型经 `workflow` 工具触达；**工具会阻塞父级轮次直到整个工作流结算，模型只看到最终结果，永远看不到中间子 agent 消息**。
- `meta` 与 `args` **只作为 JSON 数据到达，绝不当代码求值**；result **绝不拒绝**（失败兑现 `stopReason: 'error'`，取消兑现 `'cancelled'`）。

### D5. ralph【事实】

- `ralph` 工具（`packages/experimental/tool-ralph`）：**固定前台循环**，每个 Round 启动一个**全新子 agent** 处理不可变目标，只有一份有界结构化报告跨越 Round。
- 模型只能提交 `{ objective, maxRounds? }`；**调用阻塞到整个运行结算**；终态 `complete` / `blocked` / `budget-limited`。
- 每个子 agent 只收到：不可变目标、当前 Round 与上限、「共享工作区是权威状态」指令、上一份结构化交接；**父级对话与先前子 agent 会话绝不作为种子**。
- glossary 明确边界：**Ralph 是「由工作流和 subagent 原语组合而成的面向模型的工具策略，不是同会话目标、agent loop 模式、调度器或通用工作流脚本功能」**。

### D6. goal（同会话持久目标）【事实】

- `dsh-goal`：「让一个长期完成目标在多轮、会话恢复、fork 与进程重启后持续存在」；阶段 `active` / `paused` / `blocked` / `complete`；CAS 更新拒绝陈旧视图；**可配置 Round 上限（默认 256）**。
- **关键限定**：「本包**存储 goal 状态但不调度工作**，续行权限是**进程本地**的而非持久状态」。
- glossary：目标「是一种**状态，不是调度器，也不是一段独立对话**；会话日志仍是其真源」；**目标激活**态为 `armed` / `disarmed`，**有意不参与持久回放**——恢复或 fork 后只有经人类授权的恢复变更才能开始自动工作。
- 组成：`goal`（服务）、`command-goal`（`/goal` 人类命令）、`goal-round-driver`（续行驱动器）、`tool-goal`（`create_goal`/`get_goal`/`update_goal`）。
- 权限约束：create/edit/pause/resume **要求直接来自人类的根权限**；complete/blocked 也接受确切的当前 Goal Round；blocked 默认下限 3 个获准 Round。

### D7. 多 agent 原语的共同约束【事实 + 推论】

- 所有子 agent 都是**会话**（`parentSession` 记录直接创建者），没有「非会话的执行体」。
- 委派深度是**数据**（`delegationDepth` 持久、`subagentDepth` 运行时），不是能力继承。
- 子 agent 的作用域**不继承**父级注册（`docs/glossary.zh.md#agent-scope`）。
- **【推论】** dsh 的多 agent 是「**同一进程内、以会话为单位、以父轮次为生命周期锚点**」的树形委派（workflow 是扇出，ralph 是纵向串联，Agent Teams 是持久 roster），**不是**平权的多智能体社会，也不是常驻集群。

---

## E. 核心判断：框架有没有「长期在线 / 事件驱动 / 多会话调度」这一层？

### E1. 直接证据：文档里没有这些概念【事实】

我对已下载的 22 份 `docs/*.zh.md` + `README.zh.md` + `AGENTS.md` + `SAFETY.zh.md` 做了关键词扫描：

| 关键词 | 命中 |
|---|---|
| 长期在线 | **0** |
| 常驻 | **0** |
| 守护 / daemon | **0** |
| 事件驱动 | **0** |
| 多会话 / 多 Session | **0** |
| 无人值守 | **0** |

（"调度器 / scheduler" 有命中，但都是**否定式**限定：「目标……不是调度器」「Ralph……不是……调度器」，以及 vLLM 的 `scheduler priority` 无关上下文。）

### E2. 核心是单会话 ReAct 内核【事实】

- **Agent 状态只有 `idle` / `running` 两态**，没有持久状态机、没有 `paused`/`suspended`/`daemon` 形态（`docs/subsystems/core.zh.md:158`）。
- **每个 Agent 恰好绑定一个 Session**：`create()` 在 `SessionId` 下建会话+agent；`resume()` 加载持久会话再重建 agent（`packages/core/agent/README.zh.md`）。
- **循环会把每个创建的 agent 运行到完成**；「`maxParallelToolCalls` 默认 10」之类的并行只在**单轮内的工具批次**层面（`packages/core/agent-loop/README.zh.md`）。
- 内核抽象只有：**turn = 排空一次输入**，**step = 一次模型请求 + 其工具**。没有任何「跨轮次常驻的意图/目标循环」——goal 把它显式降级为「状态，不是调度器」。
- headless 的自我描述最能反证定位：「**进程不打开任何端口，也不会留下任何后台运行的东西**」「每次调用只运行一个任务，没有交互式后续」。

### E3. 但存在一圈「Host 级唤醒适配器」【事实】

这些是框架里真实存在的、位于 session/agent 之外的东西：

| 机制 | 包 / ctx 键 | 与「长期在线」的关系 |
|---|---|---|
| **定时任务** | `ctx.schedule`（`packages/schedule/schedule` + `tool-schedule`） | **宿主级**：「Schedule **独立于 Session 的激活状态**存储宿主任务」；七种权威时间类型 `after/at/every/daily/weekly/cron/rrule`；投递目标三选一：已有 Session / `new_once`（首次建后复用）/ `new`（每次新建）。**但**「**Host 必须运行**」「投递需要运行中的进程及其持久化 Session 存储」 |
| **外部 webhook** | `ctx.webhookRuntime`（`packages/experimental/webhook`） | 受信任规则注册表 + **唯一内置动作：在 Web Workspace 中创建普通根会话**；`dispatch(delivery)`；运行时**不等待 idle、不检查回复、不发布完成状态**，之后完全交给普通 Agent 行为 |
| **后台作业** | `ctx.jobs`（`packages/jobs/*` + `tool-jobs`） | 后台 bash/pwsh/PTY 的登记与读取/停止；结束时用 `agent.inject()` 发完成通知 |
| **RPC / UI 网关** | `ctx.connection`、`packages/api/gateway`、`packages/api/session-controller` | Web/Desktop 的多会话**浏览与操控**面（`@Remote`/`@RemoteScope`、`/api/remote.mux` 多路复用 WebSocket）；「会话事件、增量数据和其他流协议不属于本文范围」 |
| **目标续行** | `ctx.goals` + `goal-round-driver` | 同会话续行，但激活是**进程本地** `armed`/`disarmed`，恢复后需人类再授权 |
| **多会话存储** | `ctx.sessions`（`SessionStore`） | `list()` 返回「every live session, in creation order」；**一个进程可以同时持有多个活会话** |

**【推论】** 因此准确的表述是：

> **dsh 的内核是「单会话 ReAct 循环」；它没有内建的长期在线自治层。**
> 多会话能力属于**宿主/传输层**（Web GUI 的会话控制器、API Gateway、Schedule、Webhook、Jobs）——这些层能**创建、唤醒、投递到**会话，但**不托管会话的持续意图**。会话一旦空闲就是 `idle`，没有后台自主循环；进程退出即失去一切（除非 Schedule/Webhook 在下个进程里重建）。
> 也就是说：**它是「事件驱动的入口」而不是「事件驱动的运行时」**。外部事件能触发一次会话，但触发之后仍是普通的一问一答轮次。

### E4. 与 Super-agent 对比时应强调的三条分界【推论】

1. **状态归属**：dsh 的唯一真源是**会话日志**（`SESSION_FORMAT_VERSION` V4，JSONL 不可变代际），目标是「状态」、Round 是「策略计数」；没有独立的「世界模型 / 记忆体 / 任务队列」服务。
2. **活跃性来源**：dsh 的活跃性由**输入**驱动（`followup`/`steer`/`inject` 三个动词 + inbox），没有「内部时钟 + 自主 tick」；宿主定时器只负责**投递一条消息**，不负责维持一个持续 agent。
3. **多体结构**：dsh 的多体是**父轮次锚定的树**（subagent 树 / workflow 扇出 / ralph 纵向 / Team roster），有 `delegationDepth`、`parentSession`，**作用域不继承**；不存在对等 peer 发现、自我组队或跨会话的长期社会结构（Agent Teams 的 roster 也只存活于一个 root Session 日志内）。

---

## F. 其余面的关键事实（备查）

### F1. 工具系统【事实】

- 规模：**35 个工具包 / 76 条 schema**（`docs/tool-catalog.zh.md` 生成物，由 `pnpm run verify-tool-catalog` 校验新鲜度；完整性守卫 glob `packages/*/tool-*`）。
- 分类（按包）：文件（`tool-fs`：`read`/`write`/`edit`/`read_image`/`apply_patch`；`tool-fs-search`：`glob`/`grep`）、shell（`tool-bash`/`tool-pwsh` + 持久版）、终端 6 件套、web（`web_search`/`web_fetch`）、MCP 资源 3 件套、委派（`subagent`/`subagent_fork`/`subagent_oar`/`send_message`/`interrupt_agent`/`list_agents`）、编排（`workflow`/`ralph`）、目标与计划（`create_goal`/`get_goal`/`update_goal`、`exit_plan_mode`）、调度（`schedule_*` 4 件套）、后台（`job_*` 3 件套）、PTC（`run_code`）、会话检索（`session_search`/`session_event_*`/`session_trace`）、UI/呈现（`present`/`ask_user_question`/`todo_write`/`propose_action`）、Cordis 自省（`cordis_inspect_*`）、浏览器（`stagehand_*` 6 件套）、插件管理（`plugin_manager`/`plugin_market`）、技能（`skill`）、LSP、worktree。
- 执行流水线阶段（`docs/tool-execution-pipeline.zh.md`）：`tool/call` 记录 → 单调 guard → `tools/pre-execute` waterfall（hooks/权限/沙箱，可 deny / ask）→ `ctx.approval` **一次性**询问（缺失或无法回答即拒绝）→ `tools/execute` waterfall（timeout/retry/metrics，**around 分发**）→ 工具 body → `fs/write-intent`/`fs/edit-intent` 门禁 → 工具自有会话事件 → `projectContent` → `tools/post-execute` waterfall（accept/block/replace/add context）→ `finalizeContent` → `tools/result` 同步通知 → `tool/result` 会话事件。
- 执行器两套：`core/tool-executor-native` 与 `core/tool-executor-ptc`（PTC 只对模型暴露 `run_code`，子调用进同一受守卫流水线）。

### F2. 上下文管理【事实】

- 压缩包族：`packages/compaction/` 下 `compaction`、`compaction-basic`、`compaction-tool-result-pruner`、`compaction-image-offload`、`command-compact`（`/compact`）。
- 触发点：`dsh-compaction-basic` **在派生请求之前通过 `agent/pre-step` 处理压力**；`agent/request-error` **仅用于规范的上下文溢出**。任一触发后先做**可选的工具结果剪枝**，再选摘要；恢复发生在**仍打开的步骤内**，**只有剪枝或摘要生成推进 surface replacement generation 时才重试**，否则仍以原始错误为准（`docs/agent-lifecycle.zh.md`）。
- 压缩是**表面替换**：只隐藏被取代条目，日志保留全部历史（`packages/core/session/README.zh.md`）。
- **prompt 缓存**：**未取到**独立缓存服务或缓存策略文档。只看到每个包 README 都有「KV Cache 影响」小节（如 `dsh-base`：「组合包本身不添加任何请求前缀」；`sdk-app`：「对固定 profile、提供方、模型与工具清单，静态前缀保持稳定」）。→ **推论**：dsh 通过「稳定静态前缀 + 运行时事实追加到已记录 user 上下文」来保 KV cache 命中，但**没有**可配置的 cache 管理 seam（至少文档层未取到）。

### F3. 扩展面【事实】

- **skills**：`packages/skill/` 5 个包（`skill`、`skill-filesystem`、`skill-office`、`tool-skill`、`tool-workspace-dependencies`）；`ctx.skills`；`skill` 工具通过 `agent.inject()` 做 user/message 替换式注入。
- **MCP**：`packages/mcp/` 2 个包（`mcp-client`、`mcp-resources`）；`ctx.mcpResources`，**每个 bundle 只统一挂载一次**，只需配置客户端条目；调用方作用域无已配置服务器时不贡献工具或提示词文本。
- **插件发现**：给仓库打 **`dsh-plugin`** GitHub topic（`README.zh.md`）；`ctx.pluginMarket` 读插件市场索引。
- **Web UI**：`packages/client/` **69** 个包（`ui-chat`、`ui-conversation`、`ui-tool`、`ui-trajectory`、`ui-subagent`、`ui-goal`、`ui-jobs`、`ui-schedule`、`ui-workflow-run`、`ui-plan`…）；`apps/web` 是 Vite 构建入口。
- **Desktop**：`apps/desktop` Electron；「在签名资源中携带精确匹配的 dsh 生产运行时，并拥有保留的 `$DSH_HOME/profiles/desktop`」；Electron Node 模式启动私有 Desktop Host，走完整 Web 应用（`docs/architecture.zh.md` §桌面应用）。
- **Python SDK**：`python/`；runtime wheel 把普通 `dsh` CLI 打包为 `deepseek-harness-sdk-runtime-<platform>-<arch>`，默认 `dsh --profile sdk`。

### F4. 安全模型【事实】

`SAFETY.zh.md`：**尚未接受安全审计，不得视为安全或可用于生产**；沙箱、审批提示与权限控制「**可以降低风险，但不保证隔离，也不能保证防止损害**」；「不要把 DSH 当作不可信工作负载唯一的安全控制措施」。建议一次性 VM/容器、最小权限、备份可访问文件。
审批实现：`ctx.approval` 一次性 prompt，**absent or unanswerable: deny**（`docs/tool-execution-pipeline.zh.md`）。默认策略：文件写入限制在工作区内，危险操作前征询许可；Web 抓取无需逐次审批且可访问 loopback 与内网（`packages/bundle/base/README.zh.md`）。

---

## G. 未取到的部分（诚实清单）

1. **精确 star 数**：GitHub REST API 匿名额度耗尽（`API rate limit exceeded for 223.73.209.205`），仅从 shields.io 取到 `267k`；`created_at` / `forks_count` / `topics` 等也**未取到**。
2. **`packages/README.md`**（分组说明的权威页）、**`docs/development.zh.md`**、**`docs/user/` 全文**、**`docs/cordis-tutorial/`** —— 未逐篇读，仅通过 `docs/architecture.zh.md` 的链接名推断其存在。
3. **`docs/config-catalog.zh.md`（241KB）与 `docs/persistence-catalog.zh.md`（360KB）**：按要求未全文读，只取到「它们是生成物、是配置/持久化字段真源」这一层事实；**具体字段清单未取到**。
4. **`docs/module-graph.zh.md` 的边（peer 依赖）**：我只提取了 372 个包路径，**没有解析依赖边**，因此无法给出权威的调用/依赖分层图。
5. **prompt 缓存策略**：见 F2，**未取到**专门文档。
6. **`packages/experimental/` 中除 agent-team / subagent-oar / tool-ralph / webhook 外的 60 余个实验包**：未逐个读，只从包名推断用途。
7. **实际运行验证**：全部事实来自文档与 `package.json`，**未执行** `dsh --dump-config`、未安装依赖、未跑任何代码；文档中「约」「默认」类数值以文档原文为准，未做交叉运行时验证。

---

## H. 给 Lead 的一句话结论

**【推论】** DeepSeek Harness 是一个 **Cordis 驱动的「一切能力皆可替换插件」的 agent harness**：它以**会话事件日志为唯一真源**，用 `turn`/`step` 两层抽象实现**单会话 ReAct 内核**（`agent-loop` 本身也是可替换插件），并通过 `subagent` / `workflow` / `ralph` / `goal` / `Agent Teams` 提供**以父轮次为生命周期锚点的树形多 agent 原语**；它**没有**内建的长期在线、事件驱动或跨会话调度运行时——`Schedule`、`Webhook`、`Jobs`、`API Gateway` 是**宿主级的唤醒与传输适配器**，能把外部事件变成一次会话投递，但**不托管会话的持续意图**。与 Super-agent 类自治架构对比时，dsh 应被定位为「**可插拔的单会话执行内核 + 产品化的宿主外壳**」，而非「常驻自治体」。
