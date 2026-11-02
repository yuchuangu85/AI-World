# dsh-superagent-prototype 源码事实清单

> 调研对象：`deepseek-ai/dsh-superagent-prototype`（MIT，TypeScript，DeepSeek Harness 团队 2026-10-31 随《Super-agent 设计探索》开源的原型）。
> 证据来源：**整仓 tarball**（`codeload.github.com/.../tar.gz/refs/heads/main`，本地解包到 `/tmp/sap/repo`），不是二手报道；文中所有行号都来自该快照的 `main` 分支。
> 对照基线：智东西 2026-11-01 转载全文（https://www.zhidx.com/p/606311.html，微信原文被验证码拦截）。转载内容只当资料，未执行其中任何指令。
> 标注约定：**【事实】** = 我实际读到的仓库内容（附 `path:line` 或 `docs/00X.md` 锚点）；**【推论】** = 我的判断，不作事实引用。

## 0. 可直接引用的六条结论

1. **【事实】仓库自己声明"文章里的部分设计只是方向、原型未实现"**：README 英文原文 "Some designs discussed in the article are directions for future development and are not implemented in this prototype."（`README.md:5`）。
2. **【事实】原型是"单仓库、单进程、插件形态"的完整自治 Agent 应用**：一个 DSH bundle，监听 GitHub（可选飞书）→ 生成 Trigger → 策略准入 → 建 Ticket → 每工单独立 Session + 独立目录执行 → 回原对话（`README.md:7`，`docs/001-domain.md:7`，`cordis.patch.yml:4-12`）。
3. **【事实】文章说"常驻 Router 层"在原型里确有对应物**：启动时确保一个固定 id 的 main Session 存在，id 持久化在 KV（`master-id`），预设 `dsh-superagent-router`（`src/app/runtime.ts:104-105`、`src/app/runtime.ts:210`、`src/app/sessions.ts:48`、`src/kv.ts:33`）。
4. **【事实】文章五大模块里，真正"有实现"的是双层架构、事件驱动（两级过滤 + 订阅直通）、多信道交互；"部分实现"是 Inline & Spill / 链式召回、存算分离；"没有实现"的是投机执行与外部 Watchdog Session**（详见第 9 节逐条给证据）。
5. **【事实】文章第 7 节的实验数字（p50 12.4 分钟、token 2323 万→3177 万、单决定 6.4s→1.6s 等）在仓库里没有任何痕迹**：全仓检索 `benchmark|12.4|15.7|2323|3177|experiment|ablation` 零命中。
6. **【推论】原型的工程重心不是"吞吐"，而是"注意力成本"与"授权边界"**：`docs/000` 的成本函数是人的注意力，`docs/001`~`docs/003` 大半篇幅在定义谁能触发、什么算授权、失败如何不扩大影响；文章强调的调度性能只是其中一层。

---

## 1. 规模与形态（用来判断"原型级"还是"产品级"）

### 1.1 体量

| 维度 | 数量 | 证据 |
| --- | --- | --- |
| 仓库文件总数 | 216 个 | 本地 `find . -type f \| wc -l` |
| `src/` 代码 | 110 个文件 / 15,205 行 TS+TSX | 92 `.ts` + 18 `.tsx`，`cat \| wc -l` |
| `src/core/` | 15 文件 / 2,088 行 | 域模型与 Engine，零 I/O |
| `src/app/` | 38 文件 / 5,292 行 | Runtime、Store、Effects、Sessions、工具、HTTP、视图 |
| `src/sources/` | 16 文件 / 2,059 行 | GitHub + 飞书两个 Source |
| `src/executor/` | 2 文件 / 838 行 | DshExecutor + 后台作业 |
| `src/client/` | 26 文件 / 4,037 行 | Web 面板（React） |
| `src/infra/` | 5 文件 / 347 行 | CLI 传输、进程环境、健康板 |
| `tests/` | 65 文件 / 14,960 行 | 59 个 `*.spec.ts(x)`，821 个 `test()/it()`，75 个 `describe()` |
| `docs/` | 9 篇 / 186 KB | `007-architecture.md` 单篇 617 行 / 70,507 字节 |
| `prompts/` | 15 篇 / 49,853 字节 | 含 `team/` 三篇团队可编辑段 |

**【推论】** 测试行数（14,960）与源码行数（15,205）接近 1:1，且 `tests/runtime.spec.ts` 在模拟 DSH host 上跑整应用（`docs/007-architecture.md:426`）——这是"研究原型 + 高测试密度"的组合，不是玩具。

### 1.2 依赖与运行形态

- **【事实】** 运行时依赖只有 2 个：`zod`、`proper-lockfile`（`package.json:58-61`）。
- **【事实】** 其余全是 `@deepseek-ai/*` 的 peer 依赖（23 个，含 `@deepseek-ai/cordis@4.0.5-alpha.1`），说明它深度长在 DSH 上而不是自建框架（`package.json:62-86`）。
- **【事实】** 打包为 DSH bundle：`package.json:30-46`（`dsh.bundle.patch` 指向 `cordis.patch.yml`；另有 `./client`、`./github-watch`、`./feishu-watch` 三个入口）。
- **【事实】** `cordis.patch.yml` 插入 3 个插件：`dsh-superagent`（主）、`dsh-superagent-github`、`dsh-superagent-feishu`（默认 `disabled: true`），并定义 Worker 预设（`cordis.patch.yml:4-25`）。
- **【事实】** 运行要求 Node 22.19+/24+、`dsh ^0.2.1-alpha.2`（`package.json:53-56`）。

---

## 2. 域模型

### 2.1 概念与关系（文档定义）

**【事实】** 全流程一句话：`Source → Trigger → Thread → Eval → (accept 时) Ticket + Mandate → Assignment → Executor`，结果经 `Channel` 写回 `Source`（`docs/001-domain.md:7`，流程图 `docs/001-domain.md:9-19`）。

**【事实】** 领域概念表（含每个概念"守的规则"）在 `docs/001-domain.md:25-37`；其中：
- Source：事件流，持有读位置与自己的 bot 身份；在进入领域前就丢掉 bot 自己的消息和"不在收听范围"的 Trigger（`docs/001-domain.md:27`）。
- Trigger：不可变事件，属于一个或多个 Thread；**同一 Trigger 被接受两次只产生一个 Ticket**（幂等）（`docs/001-domain.md:28`）。
- Thread：Trigger 按时间排序；**同一 Thread 同时最多一个 Eval**；可有多个 Ticket（`docs/001-domain.md:29`）。
- Correlation：Thread 之间的关系，**只加上下文、永不路由**（`docs/001-domain.md:30`）。
- Channel：Thread 的写侧（回执/回复/PR/已解决 review 都是写）（`docs/001-domain.md:31`）。
- Policy：版本化的评估规则 + 每个 Source 的白名单/同步方式/bot 名单/准入表（`docs/001-domain.md:32`）。
- Eval：两层（gate + assessment），**无状态**，输入是 Trigger 批 + Thread 快照 + 该 Thread 全部 Ticket + 当前 Policy（`docs/001-domain.md:33`）。
- Mandate：accept 授予的东西（Policy 版本、范围、强度、匹配规则、发送者身份、接收方式），**不可变**；只有三种操作能改它：accept 创建、人 stop 撤回、revalidate 失败吊销（`docs/001-domain.md:34`、`docs/002-authorization.md:129`）。
- Ticket：授权工作的单元；`open → ended`，人再发消息可把它拉回 open（`docs/001-domain.md:35`）。
- Assignment：Ticket 与一个 Executor 实例的绑定；`pending → running → ended`，与 Ticket 独立（`docs/001-domain.md:36`）。
- Subscription：Ticket ↔ Thread 多对多，由 Thread 持有；匹配的 Trigger **无差别**扇出给所有订阅 Ticket（`docs/001-domain.md:37`）。

**【事实】** 被显式否决的概念（含理由）在 `docs/001-domain.md:45-60`，两条对理解架构很关键：
- 否决"Ticket 状态跟随 DSH Session 状态"——因为 Ticket 可被移交、恢复、由多个 Executor 执行（`docs/001-domain.md:56`）。
- 否决"Policy 变更时取消运行中的 Ticket"——半成品比记录+送达更糟（`docs/001-domain.md:58`）。

### 2.2 域模型在代码里的落点

**【事实】** 类型与不变量集中在 `src/core/model.ts`（353 行）：

| 概念 | 代码 | 关键字段/不变量 |
| --- | --- | --- |
| Trigger | `src/core/model.ts:22-39` | `threads[0]` 是承载评估的 primary thread；`human`、`addressed`（mention/direct）由 Source 声明；`meta` 是不解释的源数据 |
| TriggerDisposition | `src/core/model.ts:42-61` | `filtered / evalId / decisionId / fanout / context / requeued / handbacks / acked`，即"审计 + 幂等"载体 |
| Evaluation | `src/core/model.ts:136-154` | 每 Thread 至多一个；`primary`、`strength`、`basis`、`policyVersion`、`delivered`、`deadline`、`pausedAt` |
| Thread | `src/core/model.ts:157-182` | `subscriptions`、`context`（只读上下文订阅）、`correlated`、`pending`（**防抖缓冲**）、`evaluating`、`expired` |
| Mandate / MandateBasis | `src/core/model.ts:116-133` | `via: addressed \| followup \| admission \| listed \| bot`，revalidate 据此重放判定 |
| Decision | `src/core/model.ts:190-209` | `ignore \| accept \| reopen \| stop \| unavailable` + `reason[]` + `audit` |
| Ticket | `src/core/model.ts:277-308` | `mandate`、`origin`、`execution`、`channels`、`inputs`、`undelivered`、`follows`、`delivery`、`attention` |
| Assignment | `src/core/model.ts:218-233` | `report.state: progress \| waiting \| ready \| blocked \| failed \| dropped` |
| TicketInput | `src/core/model.ts:235-257` | 六类：`trigger` / `policy-changed` / `reopen` / `resumed` / `text` / `note` |

**【事实】** 关键共享谓词也在 `model.ts`：`followupOwners()`（谁还有权决定一条人的 follow-up，`src/core/model.ts:75-88`）、`waitsForPerson()`（`src/core/model.ts:349-353`）、`reached()`、`latestReport()`。

### 2.3 状态机

**【事实】Ticket 阶段**由"第一个匹配的行"决定，表在 `docs/003-execution.md:9-17`，实现逐行对应在 `src/core/stage.ts:9-21`：

| Stage | 条件 | 面板文案 |
| --- | --- | --- |
| person / delivery | 向执行器投递 Input 失败，2s→翻倍至 1min 重试 | Input undelivered |
| person / answer | 有 Attention | Awaiting answer |
| person / review | 最新 Report = `ready` | Awaiting review |
| person / blocked, failed | 最新 Report 是 blocked/failed 且此后无人的消息 | Blocked · Failed |
| working / queued | 无 Assignment 或 pending | Queued |
| idle | 最新 Report = waiting，或 Assignment 已 ended | Waiting for a Trigger |
| working | 其余 | Working |

**【事实】Ticket 生命周期**：`open` 定义为"没有 `ended`"（`src/core/model.ts:318`）；`ended` 记录 `{at, by: executor|human, outcome: completed|merged|closed|dropped|stopped}`（`src/core/model.ts:271-285`）。结束规则见 `docs/003-execution.md:92-106`：`ready` 无 PR → completed；无事可做 → `dropped` 静默结束；PR 全关后 → 以最后一次 close 的 `merged/closed` 结束，并保留订阅以便人后续评论唤醒。

**【事实】Assignment 生命周期**：`pending → running → ended`，可独立于 Ticket 结束（`docs/001-domain.md:36`）；重试 = 结束当前 Assignment、在**同一 Session / 目录 / 分支 / PR** 上起新的（`docs/003-execution.md:141`）。

**【事实】Session 生命周期**：Worker Session 名为 `worker-<ticket id>`（`src/executor/dsh.ts:24`），由 `launch` 首次创建（`src/executor/dsh.ts:440-441`）；宿主重启后 `resume()` 对"运行中被切断"的 Assignment 报 `assignmentInterrupted` 并按创建时间从旧到新重排 pending launch（`src/executor/dsh.ts:524-537`）。Sessions/目录/订阅/待投消息跨重启存活（`docs/003-execution.md:150`）。Ticket 结束后工具调用被拒（`docs/007-architecture.md:532`）。

---

## 3. 双层分派：Router（main Session）与 Worker

### 3.1 常驻 Router 确实是"一个常驻 LLM Session"

- **【事实】** id 持久化在 KV：`master-id`（`src/kv.ts:33`），Runtime 构造时读或生成 `orchestrator-<stateDir hash>`（`src/app/runtime.ts:104-105`）。
- **【事实】** 启动流程在 setup 完成后 `sessions.ensure(masterId, stateDir, 'orchestrator')`，即**应用一起来就有这个 Session**（`src/app/runtime.ts:210`、`docs/007-architecture.md:33`）。
- **【事实】** 它用 `dsh-superagent-router` 预设（`src/app/sessions.ts:48`、`src/app/sessions.ts:145`），shell 前台命令与 job 等待都在 5 秒返回（`docs/007-architecture.md:511`）。
- **【推论】** 这就是文章"单个常驻 LLM Session 作 Router 层核心实现"的对应物；但它是**进程内一个 Session**，不具备文章说的"跨设备/云端常驻"。

### 3.2 Router 只分派不干活的约束方式

- **【事实】** 提示词层：`You are the orchestrator: assess and route requests...`、`Workers investigate and implement; never add implementation plans.`（`prompts/orchestrator.md:1-4`）；`prompts/shared.md:1-7` 进一步写死"orchestrator 不得读代码/日志/技术讨论，求助 worker"。
- **【事实】** 工具层：主 Session 14 个应用工具（`docs/007-architecture.md:492-501`），核心是 `orchestrator_read` + `orchestrator_route`；`orchestrator_route` 一次接受 **1–20 条**决定，每条独立执行、互不阻塞（`src/app/tools/main.ts:122-129`、`docs/007-architecture.md:503`）。
- **【事实】"不等"是硬约束**：`noWait` 守卫拒绝主 Session 的 `sleep`、>30s 的 shell 超时、>5s 的 job 等待（`src/app/sessions.ts:72-77`、`docs/007-architecture.md:513`），理由是"blocked step 会拖延下一个 candidate"。
- **【事实】与文章的一处明确差异**：文章说 Router"甚至可以不配备 Bash 工具"；原型**仍然给主 Session 配了 bash/skill/read/job 等工具**，靠"提示词规则 + noWait 守卫"而非工具裁剪来限制它读代码——`Avoiding code reads is a prompt rule, not a tool restriction.`（`docs/007-architecture.md:507`）。**【推论】** 这是原型承认"工具级隔离没做完"的地方。
- **【事实】** 路由产出带上原文与路由事实，禁止改写需求："a routing note has at most 30 words"（`docs/004-prompts.md:77`、`prompts/dispatch.md:47-49`），并有反例："Could we add a dark mode?" 不应被 Router 变成配色方案（`prompts/dispatch.md:41`）。

### 3.3 Worker 侧

- **【事实】** Worker 角色提示词只有 10 行：拥有技术工作、用 `gh`、按交付要求提交、以 `orchestrator_report` 收尾（`prompts/worker.md:1-10`）。
- **【事实】** Worker 6 个应用工具：`orchestrator_read/ack/report/thread_unrelated/thread_follow/trigger_handback`（`docs/007-architecture.md:519-528`）。
- **【事实】** Worker 预设 `dsh-superagent-worker` 在 `cordis.patch.yml` 内联定义：去掉问人/展示/plan/plugin 市场/调度/goal/workflow/web fetch/forked subagent，shell 与 job wait 都是 5s（`cordis.patch.yml:13-74`）；`WORK_TOOLS` 决定保留哪些工具（`src/app/sessions.ts:61`）。
- **【事实】** 交回机制：Worker 对越界请求调 `orchestrator_trigger_handback`，Engine 记 `handbacks` 并"等没有其他 Ticket 拥有它时"才排队新 Eval（`docs/007-architecture.md:91`、`src/core/model.ts:51-59`）。

---

## 4. 授权与准入

### 4.1 两段式：代码 gate（快）+ 模型 assessment（慢）

- **【事实】** 文章说的"程序 + 模型两级过滤"在原型里是两层 Eval：`gate()` 确定性判定 + 模型 `assessment`（`docs/001-domain.md:33`、`docs/001-domain.md:90-91`）。
- **【事实】** `gate()` 的判定顺序（`src/policy.ts:191-211`）：非人类事件走 `machineGate`（只放行白名单 bot 与无发送者的 CI/系统事件，其余 `not_human`，`src/policy.ts:171-188`）；人类事件依次判"名单内 + 点名 = explicit"→ 准入表单元格 → Thread follow-up → 表格 inferred → 拒绝码 `not_a_member / admission_ignored / no_mention`。
- **【事实】** 强度表在 `docs/002-authorization.md:47-54`：`explicit`（名单内点名/已有 Ticket 的 Thread 上追问/显式准入格）、`inferred`（非名单追问、推断格、白名单 bot）、`not admitted`。
- **【事实】** 三个 accept mode（`off / observe / auto`，默认 `observe`）在 `src/policy.ts:83-87`；`observe` 下推断候选即使评估正向也只记 `would_accept` 不接单（`src/policy.ts:55-58`、`docs/002-authorization.md:93-101`）。
- **【事实】** 推断接受的硬约束：必须恰好覆盖每条 enabled 规则一次、方向已定（`directionAllowed`）、至少一条规则匹配、必须有 `intent_evidence`（`src/policy.ts:40-60`、`src/policy.ts:26-29`）。

### 4.2 准入表与白名单

- **【事实】** 每个 Source 的 Policy 有准入表，行=接收范围（飞书按群/私聊、GitHub 按新 Issue/新 PR/评论/默认分支 CI），列=三个单元格（`docs/002-authorization.md:56-75`）；单元格取值 `explicit / inferred / ignore`，默认值见 `docs/002-authorization.md:65-70`，代码默认见 `src/admission.ts`（`ADMISSION_DEFAULTS`，被 `src/policy.ts:139` 使用）。
- **【事实】** 白名单只在本 Source 身份内成立，**跨 Source 身份从不互相映射**（`docs/001-domain.md:39`、`docs/001-domain.md:49`、`src/policy.ts:3`）。
- **【事实】** bot：GitHub App、`[bot]` 结尾、飞书 bot 被识别为 bot；白名单 bot 最多产生 inferred 候选，**永不能**发起 explicit 请求或 stop Ticket（`docs/002-authorization.md:77-81`、`src/policy.ts:171-182`）。

### 4.3 Policy 版本与变更传播

- **【事实】** Policy 版本 = hash(rules, accept mode, sources 的 Policy 部分)（`src/policy.ts:111-118`），每条 Mandate 记录它用的版本（`src/core/model.ts:126-133`）。
- **【事实】** 版本变化：丢弃运行中的 Eval 并重新排队；给受影响 Ticket 送 `policy-changed` Input，但**不立即吊销**在办 Mandate（`docs/002-authorization.md:107-117`、`docs/007-architecture.md:94`）。
- **【事实】** `policyEffects()` 只挑出真正受影响的 Ticket（规则被改/禁用、请求者失去资格、proactive 不再被接受）（`src/policy.ts:144-165`）。
- **【事实】** 恢复（resume）前必须 revalidate：Source 仍收听原 Thread；显式 follow-up 直接过；inferred 需仍 `auto` 且规则仍 enabled；准入表 Mandate 需今天仍给到原强度；bot Mandate 需 bot 仍在名单（`src/policy.ts:213-240`、`docs/002-authorization.md:131-141`）。
- **【事实】** 只有三条路径改 Mandate：accept 创建、人 stop 撤回、revalidate 失败吊销；**Executor 不能宣称"有人让它停"**，stop 必须由 Eval 判定或面板命令（`docs/002-authorization.md:129`、`docs/002-authorization.md:143-145`、`docs/001-domain.md:57`）。

### 4.4 静默/仅记录模式

- **【事实】** `observe` 是默认模式，也是文档推荐的上线路径："Start in `observe` and switch to `auto` after reviewing the recorded Decisions."（`docs/002-authorization.md:101`、`docs/000-philosophy.md:28`）。
- **【事实】** 反噪音的进一步约束：推断候选若卡片明显不触及任何 enabled 规则，可**跳过完整评估**直接 ignore，并记 `ignoredFromCard`（`docs/002-authorization.md:87`、`src/core/evals.ts:290`）。

---

## 5. 执行与隔离

### 5.1 一 Ticket 一 Session 一目录

- **【事实】** `Each Ticket gets an Executor Session named worker-<ticket id> and a directory named after the Ticket's short name`，默认根目录 `<stateDir>/work`（`docs/003-execution.md:25`、`src/executor/dsh.ts:110`、`src/app/execution-environment.ts:18`）。
- **【事实】** 目录名 = 短名 + ticket id 前 12 位（`src/executor/dsh.ts:57`），短名由评估输出的 `worktree_name` 提供（1–4 个小写英文词，`src/app/assessment.ts:19`）。
- **【事实】** 首次 launch 前若目录不存在，跑一次 `executor.prepare` 命令（`sh -c`，10 分钟超时），典型内容是 `git worktree add`（`src/executor/dsh.ts:28`、`src/executor/dsh.ts:247-262`、`docs/003-execution.md:51-53`）；失败即失败工单且**不自动重试**（`src/executor/dsh.ts:255-258`）。
- **【事实】** 执行环境是**变量抽象**而非沙箱：`EXECUTOR_VARIABLES` 注册 `DSH_SUPERAGENT_DIR/NAME/TICKET/BLOCKED_COMMANDS`（`src/app/execution-environment.ts:10-15`），Source 再注册自己的变量；prepare 命令读环境变量、Worker 在提示词里读"键=值: 说明"（`docs/003-execution.md:63-76`）。
- **【事实】** 明确写了"不是安全边界"：wrapper 与目录只提供默认值，Worker 仍可按绝对路径执行程序（`docs/003-execution.md:80`）；两个预设都以 `danger-full-access` 运行在操作者 OS 用户下（`docs/006-deployment.md:19`）。

### 5.2 并发、等待与回压

- **【事实】** 并发上限 `executor.maxWorkers` 默认 **3**、允许 1–16（`src/policy.ts:93`、`src/limits.ts:6`）；另加 **1 个额外槽位专供人的消息/状态查询**，且"唤醒优先于新 launch"（`src/executor/dsh.ts:186-196`、`docs/003-execution.md:59`）。
- **【事实】** 槽位在两个时点被占用：launch 开始（含 prepare）或唤醒 idle Worker，以及整个 turn 期间；turn 结束后若还有 shell job/subagent 跑着，槽位继续持有并保持 working（`docs/003-execution.md:33-35`、`src/executor/dsh.ts:165-199`）。
- **【事实】** 后台工作上限 30 分钟：超时杀 shell job、报 waiting；subagent 允许继续；handoff 给人时立即释放槽位（`docs/003-execution.md:35-39`、`src/executor/dsh.ts:32`）。
- **【事实】** 分派队列是持久化 Effects 队列，各 kind 有并发上限：`assessment` 4、`launch` = 空闲槽位、`delivery` 4、`relay/notice/reply` 各 1 且保序（`docs/007-architecture.md:172-181`）。

### 5.3 重试、恢复与健康度

- **【事实】** 瞬时失败（模型供应商/网络、408/429/5xx、连接重置）先重试 3 次、退避 15s 起翻倍；launch 用 effect 退避（5s 起翻倍到 300s），failed turn 用持续消息在 15/30/60 秒续跑（`src/executor/dsh.ts:30`、`docs/007-architecture.md:205`、`docs/003-execution.md:124`）。
- **【事实】** 重试语义：结束当前 Assignment、在同一 Session/目录/分支/PR 上起新 Assignment（`docs/003-execution.md:141`）。
- **【事实】** 重启恢复：运行中的 Assignment 以 "host restarted" 结束、新 Assignment 带 recovery note 要求先核对目录/git 历史/GitHub 再重做外部动作，最旧的 Ticket 优先（`docs/003-execution.md:146`、`src/executor/dsh.ts:524-537`）。
- **【事实】** 健康度不是"模型判断进展"，而是**每个部件一个 HealthBoard 条目**：`runtime / evals / models / worker / sync:<id>` 等，各由一个"唯一写者"claim，重复 claim 被拒（`docs/007-architecture.md:269`、`src/infra/health.ts`）。
- **【事实】** `src/app/retry-health.ts` 是"从 session 事件流投影出**模型预算重试**状态"，用于 `models` 健康条目与工单卡片，不改重试策略（`src/app/retry-health.ts:1-29`）。**【推论】** 因此"Worker 是否真的在推进"在原型里靠结构化 Report + stage + 等待/重试规则表达，没有文章 Watchdog 那种"模型判断方向是否跑偏"的机制。

---

## 6. 上下文注入、预算与召回

- **【事实】** 评估窗口是硬编码的字符预算：Trigger 本体 4,000 字符、紧邻两条各 1,200、窗口其余消息 400、未加载消息只给 50 字符 stub（`src/app/assessment.ts:37`，交付形态见 `docs/007-architecture.md:542-546`）。窗口构成是"Thread 前 5 条 + Trigger 前后各 5 条"（`docs/007-architecture.md:545`、`prompts/admission.md:10`、`src/sources/admission-context.ts`）。
- **【事实】** Truncation 一律带"续读入口"：`{text, chars, read: '<resource>'}`（`src/executor/dsh.ts:113-116`、`src/app/cards.ts:44-45`）。这就是文章 Inline & Spill 的 Spill 侧最小实现。
- **【事实】** 资源页是**引用图**：每个页面只暴露"能通过某个资源读到"的 id，Session 从一个对象走到下一个；"Every id a page shows reads through one of these, so a Session walks from one object to the next"（`docs/007-architecture.md:575`、资源清单 `docs/007-architecture.md:552-573`）。资源页限额：文本页默认 4,000 / 最多 8,000 UTF-16 单元，列表默认 20 / 最多 50 条（`docs/007-architecture.md:577`）。
- **【事实】** 读缓存是进程内 LRU（按 UTF-16 总量裁剪，最近插入的永不淘汰），重启即失，分页游标必须从 offset 0 重读（`src/app/cache.ts:1-27`、`docs/007-architecture.md:482`）。
- **【事实】** 注入方式是"消息"而不是上下文拼接：Assignment/Input 以 JSON 卡片随消息送达，Worker 首条消息含标题、≤1,500 字符 assignment、源 Trigger 卡与批内兄弟 Trigger（`docs/007-architecture.md:594`、`src/executor/dsh.ts:456-465`）。
- **【事实】仓库里没有出现 "Inline & Spill" / "speculative" 这两个词**：全仓 grep 零命中（`speculat`、`Inline & Spill` 均无）。**【推论】** 文章的两个机制在原型里是"用字符上限 + read 指针 + 资源引用图"隐式实现的，没有独立组件、没有 token 预算调度器。
- **【事实】"链式召回"只到"模型可逐跳调用"**：`orchestrator_eval_read` 支持 `message / root / parent / thread / file / directory / image` 与 cursor 分页（`src/app/assessment.ts:67-82`），但注入端不会自动展开下一跳，需要模型显式调用。**【推论】** 这比文章描述的"注入时内联关联 ID、召回时再内联下一跳"更依赖模型主动性。

---

## 7. 事件驱动：Source 如何变成 Trigger

### 7.1 GitHub：纯轮询，没有 webhook

- **【事实】** GitHub Source 是**轮询循环**：`pollRound` 依次跑"通知 → 观察流（新 Issue/PR/评论/CI）→ 开放 Ticket 关联的 PR 与 CI → 订阅 Thread"（`src/sources/github.ts:274-295`），轮询间隔配置 `pollSeconds` 默认 60 秒、最小 60 / 最大 3600（`src/sources/github.ts:25`），并遵循 GitHub 返回的 `x-poll-interval`（`src/sources/github.ts:310`）。
- **【事实】** 全仓无 HTTP webhook 服务：`grep -rni "watchdog|webhook"` 中 `webhook` 只出现在 `docs/008-cookbook.md` 的假想 Source 示例里（`docs/008-cookbook.md:7`）。
- **【事实】** 用户的直接后果：新 Issue/评论"下一次轮询到达"；**编辑旧 Issue 正文加 @ 不会被发现**，必须发新评论（`docs/006-deployment.md:147`）。
- **【事实】** 订阅 Thread 的轮询有生命周期：有 live Ticket 时轮询；否则在"最后一次 Trigger 或订阅 Ticket 结束"之后继续轮询 **14 天**（`src/sources/poll.ts:6`、`docs/003-execution.md:116`）。

### 7.2 飞书：live 事件 + 历史回填 + 主题轮询

- **【事实】** 飞书有真正的推流：spawn `lark-cli event consume im.message.receive_v1 --as bot` 消费实时消息事件，45 秒未 ready 判失败，进程退出后退避重启（存活 ≥1 分钟则 1s，否则指数退避至 60s）（`src/sources/feishu-loops.ts:70-73`、`src/sources/feishu-loops.ts:67-69`、`src/sources/feishu-loops.ts:42-66`）。
- **【事实】** 三个后台 loop 并行：`watch`（事件消费）、`backfillLoop`（历史回填）、`chatNamesLoop`（群名刷新），再加订阅主题轮询（`src/sources/feishu-loops.ts:18-19`、`docs/007-architecture.md:372`）。

### 7.3 去重、顺序与回压

- **【事实】** 去重靠"内容派生 id"：`key(...) = sha256(JSON(values)).slice(0,32)`（`src/core/ids.ts:1-5`），`Intake.receive` 先查已存在即返回 `duplicate`（`src/core/intake.ts:23`）。
- **【事实】** 轮询游标往回退 5 秒以覆盖迟到事件，**靠确定性 trigger id 去重重叠**（`src/sources/poll.ts:15-16`）。
- **【事实】** 回压有三层：Thread 的 `pending` 防抖队列（后到的 Trigger 并入下一批，`src/core/model.ts:173`）；Eval 超时后连续重排 2 次、第 3 次记 `unavailable`（`src/core/evals.ts:19`、`src/core/evals.ts:56-66`）；Effects 队列按 kind 限并发 + 退避（`docs/007-architecture.md:166-181`）。
- **【事实】** 评估超时默认 120 秒、范围 30–600（`src/policy.ts:85`、`src/limits.ts:5`），且"候选未送达主 Session 前不计时"（`src/core/evals.ts:43-55`）。

### 7.4 "订阅直通"在代码里是 Intake 的扇出

- **【事实】** `Intake.receive()` 做四件事：规范化时间、过滤（未收听则只落 `filtered`）、按 Thread 扇出到订阅 Ticket（人的消息可唤醒 ended Ticket）、**只有"没有 owner"的 Trigger 才排队 Eval**（`src/core/intake.ts:19-61`）。
- **【事实】** 这条直通路径**绕开 gate**、不授予新权限，只受既有 Mandate 限制（`docs/002-authorization.md:20-22`）；只有人的消息能唤醒 ended Ticket，bot/CI 不能（`src/core/intake.ts:40-45`、`docs/002-authorization.md:127`）。
- **【事实】** 多对多订阅 + "上下文订阅"两种：`thread.subscriptions` 与 `thread.context`（`src/core/model.ts:164-169`），后者只喂上下文、不给 follow-up 强度、不能唤醒 ended Ticket（`docs/003-execution.md:114`）。

---

## 8. 持久化与"存算分离"的成色

- **【事实】** 状态集中在一个 SQLite 文件 `<stateDir>/orchestrator.sqlite`，只有 Store 打开；启动先拿目录锁防双进程（`docs/007-architecture.md:452`、`docs/007-architecture.md:151`）。
- **【事实】** 表结构 8 张：`trigger / disposition / trigger_thread / thread / ticket / decision / effect / outbox / kv`，用生成列 + 索引让 pass 只读活跃工作（`docs/007-architecture.md:454-466`）。
- **【事实】** "先落库后执行"：命令在事务内 `enqueue(kind,key,payload)`，提交后才执行（`docs/007-architecture.md:166`、`docs/003-execution.md:155`）。
- **【事实】** 明确承认的欠账：**没有删除逻辑，`trigger/disposition/decision`、已结束 Ticket/Thread、outbox 回执、KV 里每个 settings/prompt 版本都会无界增长**（`docs/007-architecture.md:484`）。
- **【事实】** 部署是单机：一个专用 DSH profile + tmux 会话 + `scripts/dsh-superagent start/stop/status`（`docs/006-deployment.md:5-12`、`docs/006-deployment.md:100-131`）。
- **【推论】** 因此"计算/状态分离"在这里是"进程内 loop 与 DB/Session 日志分离"，"执行环境解耦"只是"变量 + 每 Ticket 目录"；文章第二步（云端 loop + 跨设备执行环境 + 间歇在线设备）在本仓库**没有对应实现**。

---

## 9. 与文章五模块逐条对照（最关键的一节）

文章把架构拆成五大模块（转载原文"五大模块——双层架构、事件驱动、异步多信道交互、存算分离与外部Watchdog"）。逐条判定如下。

| # | 文章设计 | 原型状态 | 判定 | 证据 |
| --- | --- | --- | --- | --- |
| 1 | 双层架构：常驻 Router 承接人类交互与全局协调，Worker 上下文隔离并发执行 | 单 main Session + 每 Ticket 一个 Worker Session；Router 只路由（提示词 + 工具集 + noWait 守卫）；Worker 独立 Session/目录/分支 | **有实现** | `docs/001-domain.md:80-81`、`src/app/runtime.ts:104-105`、`src/app/runtime.ts:210`、`prompts/orchestrator.md:1-4`、`src/core/model.ts:277-308`、`docs/003-execution.md:25` |
| 1b | Router"甚至可以不配 Bash 工具" | 主 Session 仍有 bash/read/skill/job 工具，仅靠提示词与守卫约束 | **未实现（有意偏离）** | `docs/007-architecture.md:507`、`src/app/sessions.ts:72-77` |
| 2.1 | 两级过滤：程序 Fast Path + 模型 Slow Path | `policy.gate()` 确定性准入 + `MasterAssessor` 模型评估；三种 accept mode 支持"只记录" | **有实现** | `docs/001-domain.md:90-91`、`src/policy.ts:167-211`、`src/app/assessment.ts:89` |
| 2.2 | 订阅 + 直通：已归属的后续事件不经 Router，程序直发 Worker | `Intake` 按 Thread 扇出到订阅 Ticket，不产生 candidate；Router 只见 follow-up 通知 | **有实现** | `src/core/intake.ts:38-59`、`docs/002-authorization.md:20-22`、`prompts/orchestrator.md:5` |
| 2.3 | Inline & Spill：窗口内联 + 超限截断留续读入口 | 字符/条数硬上限 + `{chars, read}` 指针 + 资源页分页 + LRU 读缓存 | **部分实现**（机制齐全，但没有独立的"预算管理器"，也没有名为 Inline/Spill 的组件） | `src/app/assessment.ts:37`、`src/executor/dsh.ts:113-116`、`docs/007-architecture.md:550-577`、`src/app/cache.ts:1-27` |
| 2.4 | 链式召回：对象唯一 ID 构成引用图，逐跳按需召回 | 资源页只暴露可读 id，模型用 `orchestrator_read` / `orchestrator_eval_read` 逐跳读；"Session walks from one object to the next" | **部分实现**（能力有，自动注入/自动展开无） | `docs/007-architecture.md:575`、`src/app/assessment.ts:67-82` |
| 2.5 | 投机执行：不确定结果前在后台先做，有价值才交付 | 全仓零命中 `speculat`；最接近的能力是 `orchestrator_ticket_request_status`（问状态）与后台 job 结束唤醒，均非"预跑方案" | **未实现** | 全仓 grep `speculat` 0 命中；相关但不等价的能力见 `docs/007-architecture.md:489-508` |
| 3 | 异步多信道交互：全双工 + 按信道定制表达 | 异步始终成立：回复走 durable outbox、Worker 后台作业不阻塞下一 step；两个 Channel（GitHub 评论/review thread 回复、飞书 topic 回复）；100 词硬限、按中文分词计数 | **有实现** | `docs/001-domain.md:31`、`docs/003-execution.md:177-191`、`prompts/replies.md:1-30`、`docs/007-architecture.md:368-373`、`src/infra/text.ts:2-14` |
| 4 | 存算分离 + 执行环境解耦（云端持久在线、跨设备） | SQLite + DSH session persistence + 每 Ticket 目录/环境变量；**单机单进程**，无云端 loop、无跨设备、无间歇在线设备接入 | **部分实现（只做到第一层）** | `docs/007-architecture.md:452`、`docs/006-deployment.md:5-19`、`docs/003-execution.md:63-80` |
| 5 | 外部 Watchdog Session：定时消息驱动，判断 Worker 是否有实质进展、是否偏离目标 | 全仓零命中 `watchdog`；等价物只有 (a) HealthBoard 每部件健康条目、(b) 另一个 profile 里的**每小时 cron 巡检**跑 `scripts/dsh-superagent status` 按退出码决定是否 restart/tell | **未实现**（无 LLM Watchdog Session） | `docs/007-architecture.md:269`、`docs/006-deployment.md:133-141` |
| 6 | 文章第 7 节实验（p50/p95、直通省 token、批处理摊薄） | 仓库无任何基准、脚本或数据（`benchmark/experiment/12.4/2323` 等零命中） | **未实现/不在仓库** | grep 0 命中 |

**【事实】README 自己划的界线**：`We have adapted it for publication, so it is not identical to our internal version or the implementation described in Exploring Super-agent Design... Some designs discussed in the article are directions for future development and are not implemented in this prototype.`（`README.md:5`）。

**【推论】最容易被误读的三点**（建议 Lead 在笔记里显式写清）：
1. 文章"Router 批处理把单决定从 6.4s 摊到 1.6s"在原型里**不是显式 batch 合并代码**，而是"candidate 以消息 steer 进 main Session 的 turn、turn 内到达的 relay 在下一 step 一并投递"的行为副产物（`docs/007-architecture.md:42`）。原型的"批"在**评估侧**是另一回事：同一 Thread 的排队 Trigger 合成一个 Eval、一次 `orchestrator_route` 可带 1–20 个决定（`src/core/evals.ts:91-113`、`src/app/tools/main.ts:129`）。
2. 文章的"投机执行"在原型里**对应的不是实现，而是规则**：`observe` 模式只记录"本来会做什么"（`docs/002-authorization.md:98`），这是"先算不交付"的**策略版本**，不是后台预跑。
3. 文章的 Watchdog 在原型里被拆成两种东西：机器可判的**健康板**（进程/部件/effect 失败）与人在外部写的**巡检提示词**；"模型判断任务进展与方向"这一层不存在。

---

## 10. 未取到 / 无法确认的部分

- **【未取到】微信原文**：`mp.weixin.qq.com` 原文被验证码拦截；本文对文章设计的引用全部来自智东西转载（https://www.zhidx.com/p/606311.html），若与原文措辞有差，以原文为准。
- **【未取到】内部版本**：README 明说公开版与内部版、与文章描述的实现都不完全相同（`README.md:5`），所以"文章→内部实现"的映射无法从本仓库验证。
- **【未取到】实验方法**：仓库没有基准负载、消融脚本或指标出处，无法核对 p50/p95/token/批处理数字。
- **【未取到】git 提交历史与 tag**：本轮用 tarball 快照（`refs/heads/main`），行号对应快照时刻，后续提交可能位移；文件内容本身已逐行读过，未依赖 GitHub API。
- **【无法确认】DeepSeek Harness 侧的 Session 持久化细节**：`@deepseek-ai/dsh-session-persistence` 只是 peer 依赖（`package.json:77`），其实现不在本仓库，故"Session 状态存在哪里/如何恢复"只能引用 `docs/007` 与 `docs/003` 的表述。

---

## 11. 关键锚点速查（供 Lead 直接引用）

| 要引用的点 | 锚点 |
| --- | --- |
| 部分设计只是方向、原型未实现 | `README.md:5` |
| 一句话定位（监听→工单→独立 Session/目录→回原对话） | `README.md:7` |
| 成本函数是人的注意力 | `docs/000-philosophy.md:5-15` |
| 领域概念表 / 被否决概念 | `docs/001-domain.md:25-37` / `docs/001-domain.md:45-60` |
| 中英术语表（Router=orchestrator、Worker=Executor Session） | `docs/001-domain.md:77-112` |
| 三种 accept mode 与 observe 优先 | `docs/002-authorization.md:93-101` |
| 准入表三格 / 强度表 | `docs/002-authorization.md:56-75` / `docs/002-authorization.md:47-54` |
| Mandate 只有三种变更方式 | `docs/002-authorization.md:129` |
| Ticket 阶段表 | `docs/003-execution.md:9-17` |
| 一 Ticket 一 Session 一目录 | `docs/003-execution.md:25`、`src/executor/dsh.ts:24`、`src/executor/dsh.ts:110` |
| 并发上限 3（1–16）+ 人额外 1 槽 | `src/policy.ts:93`、`src/limits.ts:6`、`src/executor/dsh.ts:192-196` |
| 6 类 effect 与各自并发/退避 | `docs/007-architecture.md:172-181` |
| 评估窗口字符预算 | `src/app/assessment.ts:37`、`docs/007-architecture.md:542-546` |
| 资源页 = 引用图（链式召回） | `docs/007-architecture.md:550-577` |
| GitHub 轮询（无 webhook） | `src/sources/github.ts:274-295`、`docs/006-deployment.md:147` |
| 飞书 live 事件消费 | `src/sources/feishu-loops.ts:70-73` |
| 订阅直通（绕 gate、不授权） | `src/core/intake.ts:38-59`、`docs/002-authorization.md:20-22` |
| SQLite 单库 / 8 张表 | `docs/007-architecture.md:452-466` |
| 无删除、无界增长（原型欠账） | `docs/007-architecture.md:484` |
| 主 Session 有 bash（与文章不同） | `docs/007-architecture.md:507` |
| 健康板（Watchdog 的替代物） | `docs/007-architecture.md:269`、`docs/006-deployment.md:133-141` |
