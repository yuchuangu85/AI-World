# 本机 DSH 运行时实况（Desktop 0.2.1-rc.5）

> 目的：把本机安装的 DeepSeek Harness 桌面版当作"官方资料的运行时事实"，供 `docs/architecture.zh.md` 交叉验证。
> 约束：全程只读；未修改 App、未改任何用户配置；未打印任何密钥/token/凭据值（只记字段名）。
> 标注约定：**【事实】**＝直接读到的内容（附路径锚点）；**【推论】**＝由事实推出的判断；**【未取到】**＝本机无法证实。

## 0. 元信息与读取方法

- 【事实】应用：`/Applications/DeepSeek Harness.app`，`CFBundleIdentifier=com.deepseek.dsh`，版本 `0.2.1-rc.5`（`Contents/Info.plist`；`Contents/Resources/runtime/primary-runtime/runtime.json` 的 `desktopVersion` 同为 0.2.1-rc.5）。
- 【事实】Electron 44.7.0 / Node 24.21.0 / pnpm 11.28.5（`~/Library/Logs/DeepSeek Harness/crash-2026-10-31T05-22-41-236Z-host.log` 头部；`runtime/versions.json`）。
- 【事实】更新通道：`provider: generic`、`url: https://download.deepseek.com/dsh-desk/feeds/mac-arm64/`、`channel: nightly`（`Contents/Resources/app-update.yml`）。
- 【事实】asar 未解包也能读：`app.asar` 是 8 字节头 + pickle(JSON 文件表)。本机实测参数（可复现）：偏移 0 的 u32=4；偏移 4 的 u32=4761012（第二个 pickle 总长）；偏移 12 的 u32=4761003（JSON 字符串长度）；JSON 起点 16；**数据区起点 8+4761012=4761020**。文件表共 **18237** 个文件。
- 【事实】`app.asar` 大小 182804145 B（≈174 MiB）；`app.asar.unpacked/` 只放原生件（`node-pty/prebuilds`、`@vscode/ripgrep-darwin-arm64`、`node-addon-require-builtin-darwin-arm64`、`pako` 等），完整 JS 代码在 asar 内。
- 【推论】下文所有 `app.asar/...` 路径都指"asar 内路径"，用上述头部解析（或 `node:fs` 按 `offset` 读）取得，不是磁盘上的真实目录。

## 1. 版本与组成（问题 1）

- 【事实】桌面壳包：`@deepseek-ai/dsh-desktop` 0.2.1-rc.5，`main: lib/main.js`（`app.asar/package.json`）；构建提交 `dshBuildCommit=d4db512a969fccf0c5cc7ec8fe38e9c1d27af916`、`dshBuildDirty=false`（同上）。
- 【事实】运行时包：`@deepseek-ai/dsh-desktop-runtime` 0.2.1-rc.5，依赖 **305** 个，其中 **297** 个是 `@deepseek-ai/dsh-*` 插件包；Cordis 版本 **4.0.5**（`@deepseek-ai/cordis`），配套 `cordis-plugin-{group,include,loader,timer}` 1.0.5/1.0.10/1.0.6/1.1.7（`app.asar/dsh/package.json`；`app.asar/dsh/desktop-runtime.json` 的 `sharedPackages` 305 条同值）。
- 【事实】`app.asar/dsh/node_modules/@deepseek-ai/` 下实际落了 **308** 个包目录（含 cordis、cosmokit、schemastery、libreoffice-kit 等非 dsh-* 包）。
- 【事实】运行时清单 `app.asar/dsh/desktop-runtime.json` 字段：`release{schemaVersion,version,hostProtocolVersion:4,nodeVersion,pnpmVersion}`、`platform:darwin`、`arch:arm64`、`sharedPackages[305]{name,version,path}`、`files[17190]{path,bytes,sha256,executable}`——即"随桌面版出厂的精确 payload 清单（含哈希）"。
- 【事实】`Contents/Resources/runtime/` 内容：`versions.json`（node 24.21.0 / pnpm 11.28.5）、`bin/node`（壳脚本：`ELECTRON_RUN_AS_NODE=1` + `--expose-internals`，`bin/node.cmd` 同构）、`cli/{bin/dsh,command-manager.js,command-path.ps1,link-entry}`、`office-skills/`（7 个技能目录，均含 `SKILL.md`：office、office-docx、office-xlsx、office-pptx、office-export-pdf、office-render、office-recalculate）、`primary-runtime/{runtime.json,dependencies/{node,pnpm,python}}`。
- 【事实】`primary-runtime/runtime.json`：python **3.12.14** + 预装包 numpy 2.3.5 / pandas 3.0.1 / python-docx 1.2.0 / python-pptx 1.0.2 / openpyxl 3.1.5 / Pillow 12.3.0 / lxml 6.1.3 / XlsxWriter 3.2.9 等（这就是 `load_workspace_dependencies` 返回的那套）。
- 【推论】桌面版 = "Electron 壳（`lib/` + `renderer/`）+ 打包的 dsh 运行时（`dsh/`）+ 一份带哈希的 payload 清单 + 独立 python/node 工具链"，四者都在同一 asar/Resources 内，便于离线与校验。

## 2. 代码在 asar 里怎么组织（问题 1 续）

| asar 顶层 | 内容 | 关键文件 |
|---|---|---|
| `package.json` | 桌面壳清单（main、更新策略、构建 commit） | `app.asar/package.json` |
| `lib/` | Electron 主进程/preload | `lib/main.js`（516716 B，13200 行）、`lib/preload-app.cjs`、`preload-mandatory/platform-account/update-dialog/welcome` |
| `renderer/` | 桌面壳自带页面（欢迎、强制更新、更新对话框） | `renderer/welcome.html`、`renderer/mandatory-update.html` |
| `node_modules/` | 壳自身依赖（electron-updater、semver、ws、zod…） | — |
| `dsh/` | **真正跑起来的 DSH**：`@deepseek-ai/*` 305 包 + `node_modules` | `dsh/package.json`、`dsh/desktop-runtime.json` |

- 【事实】壳与运行时的边界在代码里也可见：`lib/main.js` 通过 IPC `dsh-desktop:boot` 请求启动，失败时把诊断写 `~/Library/Logs/DeepSeek Harness/*-{host,web-boot}.log`（`app.asar/lib/main.js`；崩溃日志样例见 §0 与 §6）。
- 【事实】`app.asar/dsh/` 就是本会话运行时上下文里说的"实现 checkout 路径"，其内容与 `@deepseek-ai/*` 各包的发布形态一致（每包 `lib/index.js` + `README*.md` + 可选 `cordis.patch.yml`）。

## 3. 工具清单（问题 2）

- 【事实】工具注册的代码特征：`ctx.tools.register(defineTool({ name, description, parameters }))`；系统提示词分段用 `ctx.systemPrompt.section({name:"tool:<x>"})`（样例 `app.asar/dsh/node_modules/@deepseek-ai/dsh-tool-fs/lib/index.js:245-251`）。
- 【事实】扫描 `dsh/node_modules/@deepseek-ai/dsh-tool-*`、`dsh-experimental-tool-*`、`dsh-plan-mode`、`dsh-plugin-manager` 的 `defineTool` 定义，静态名字共 **53** 个；再加 5 个"名字由 config 决定"的动态工具（`subagent`、`subagent_fork`、`subagent_claude_code`、`subagent_codex`、`workflow`）与 `exit_plan_mode`，**代码内可定义上限 ≈59**。
- 【事实】本会话的会话头（`~/.dsh/sessions/--Users-yuchuan.gu-Workspace-MX-AI-World--/2c199e8f-…/session.v4.jsonl.zstd` 首行，zstd 解后原文）：`{"type":"session","version":4,"cwd":"/Users/yuchuan.gu/Workspace/MX/AI-World","parentSession":"session-a63741e6-…","isSeeded":false,"origin":"subagent","delegationDepth":1,"agentPreset":"standard"}`——即本会话是**子代理会话**（depth 1，preset=standard）。
- 【事实】该 preset 下实际拿到 **47** 个工具。差集是下列"未挂载到本会话"的：`plugin_manager`、`cordis_inspect_list/query`、`str_replace_editor`、`terminal_*`（6 个）、`pwsh`、`list_subagent_models`。
- 【推论】"工具数"要分三层说：仓库可定义 ≈59 / 该 profile 默认挂载 ~50 / 本会话可用 47；差异来自 profile patch 与 agent preset 的按会话授权。

按族分组（★ = 本会话可用）：

| 族 | 工具 | 提供包 |
|---|---|---|
| 文件读写 ★ | `read` `write` `edit` `read_image` | `dsh-tool-fs` |
| 文件检索 ★ | `glob` `grep` | `dsh-tool-fs-search` |
| 命令执行 ★ | `bash`（`pwsh` 仅 Windows） | `dsh-tool-bash`（另有 `dsh-tool-bash-persistent` 变体，sdk-minimal 用）/ `dsh-tool-pwsh` |
| 终端会话 | `terminal_open/send/read/signal/close/list` | `dsh-tool-terminal` |
| 工作目录 ★ | `working_directory`、`create_worktree` | `dsh-tool-working-directory`、`dsh-experimental-tool-worktree` |
| 后台任务 ★ | `job_list` `job_output` `job_kill` | `dsh-tool-jobs` |
| 会话查询 ★ | `session_search` `session_trace` `session_event_search/read/trace` | `dsh-tool-session-query` |
| 目标与计划 ★ | `create_goal` `get_goal` `update_goal`、`exit_plan_mode` | `dsh-tool-goal`、`dsh-plan-mode` |
| 待办/进度 ★ | `todo_write` | `dsh-tool-todo` |
| 提醒 ★ | `schedule_create/update/delete/list` | `dsh-tool-schedule` |
| 委派 | `subagent` `subagent_fork` `subagent_claude_code` `subagent_codex` `list_subagent_models` | `dsh-tool-subagent`（同一包的多个 config 实例）+ profile 插件 |
| 团队 ★ | `spawn_teammate` `send_message` `list_agents` `wait_agent` `interrupt_agent` `team_task_create/get/list/update` | `dsh-experimental-tool-agent-team` |
| 工作流/迭代 ★ | `workflow`、`ralph` | `dsh-tool-workflow`、`dsh-tool-ralph` |
| Web ★ | `web_search` `web_fetch` | `dsh-tool-web` |
| 人机交互 ★ | `ask_user_question` `present` | `dsh-tool-ask-user`、`dsh-tool-present` |
| 技能/环境 ★ | `skill`、`load_workspace_dependencies` | `dsh-tool-skill`、`dsh-tool-workspace-dependencies` |
| 插件自省 | `plugin_manager`、`cordis_inspect_list` `cordis_inspect_query` | `dsh-plugin-manager`、`dsh-tool-cordis` |
| 文本编辑（备用） | `str_replace_editor` | `dsh-tool-str-replace-editor` |

- 【事实】工具呈现模式有开关：`tools` 插件有 `mode: native|ptc`，web-app 层从 `DSH_TOOLS_MODE` 环境变量读取（`app.asar/dsh/node_modules/@deepseek-ai/dsh-web-app/cordis.patch.yml` 的 `- id: tools` 行；PTC 引擎为 `dsh-workflow-ptc`/`dsh-ptc-runtime-node`）。
- 【事实】**工具可见性的第三层是 agent preset**：随 web-app 出厂 4 个 preset patch——`app.asar/dsh/node_modules/@deepseek-ai/dsh-web-app/presets/{standard,minimal,ptc,cordis}.patch.yml`。每个 insert 一行 `@deepseek-ai/dsh-agent-preset`，其 `config.plugins` 是一棵**独立插件子树**（按 id 分组，`cordis:group` + `isolate` realm，如 `planning`→`isolate.planMode`、`compaction`→`isolate.compaction`）。本会话 header 的 `agentPreset: "standard"` 即命中 `presets/standard.patch.yml`，其 `delegation` 组授权 `tool-subagent-control`（`send_message`/`interrupt_agent`）、`tool-subagent-control/list-agents`、`tool-subagent`（`subagent`）、`tool-subagent-fork`（`subagent_fork`）、`workflow-ptc` + `tool-workflow`；另含 `tool-bash`、`tool-fs`、`tool-fs-search`、`tool-jobs`、`tool-schedule`、`tool-skill`、`tool-goal`、`plan-mode`、`tool-ask-user`、`tool-todo`、`tool-web`、`tool-present`、`tool-plugin-manager`、`tool-plugin-market`。
- 【推论】本会话 47 个工具 ≈ standard preset 的工具子集 + desktop profile 实验 bundle 追加（`session-query` 5 个、`create_worktree`、`ralph`、Team 9 个、`subagent_codex`/`subagent_claude_code`）+ 桌面自有 `desktop-office` 追加的 `load_workspace_dependencies`，再减去未授权的 `plugin_manager`/`pwsh`/`terminal_*`/`str_replace_editor`。

## 4. 插件面：机制、清单、能挂什么（问题 3）

**机制【事实】**（`app.asar/dsh/node_modules/@deepseek-ai/dsh-app-boot/lib/index.js:484-509` 注释即完整规格）：

1. profile 目录 `$DSH_HOME/profiles/<name>/`，内 `package.json` 声明 `dsh.profile.bundles`（有序组合包）与树外插件依赖；`cordis.patch.yml` 是用户 patch 层。
2. **组合包（bundle）**＝npm 包，manifest 里 `"dsh": { "bundle": { "patch": "./cordis.patch.yml" } }`（可多文件）。
3. 层序（原文）：各 bundle 的 patch 按 `bundles` 顺序叠加到**空条目列表** → profile 自己的 patch → home patch（`$DSH_HOME/cordis.patch.yml`，`dsh-app-boot/lib/index.js:1131`）→ launcher 层（`--patch` 与 flag 派生 patch）。`dsh-app-boot/lib/index.js:3207` 注释："Bundle, profile, home, and CLI layers apply in that order. **A patch config replaces the whole config.**"
4. 模块解析是"双锚点"：先从 dsh 安装（launcher 自身包），再从 profile 目录（`dsh-app-boot/lib/index.js:499-503`）。

**本机实际清单【事实】**：

| 位置 | 内容 |
|---|---|
| `~/.dsh/profiles/desktop/package.json` | `dsh.profile.bundles` **11** 个：`dsh-base`、`dsh-web-app`、`dsh-experimental-{agent-team-profile, auto-review, inspector-profile, tool-worktree, cot-translation-bundle, session-search, ralph-bundle}`、`dsh-subagent-codex`、`dsh-subagent-claude-code`；树外依赖 3 个（含 `dsh-subagent-cli` → **link** 到 `/Users/yuchuan.gu/Workspace/AI/dsh-subagent-cli`） |
| `~/.dsh/profiles/desktop/cordis.yml` | 只有 `[]` + 注释："The tree is composed as patches: each bundle in package.json's dsh.profile.bundles, then cordis.patch.yml, then any --patch overlays. Edit cordis.patch.yml, not this file." |
| `~/.dsh/profiles/desktop/cordis.patch.yml` | 4 条按 id 覆盖：`ui-settings-account`、`ui-chat`、`ui-settings`、`locale: zh` |
| `~/.dsh/cordis.patch.yml`（home 层） | 1 条 insert：`id: loongsuite-pilot-observability`、`name: file:///Users/yuchuan.gu/.loongsuite-pilot/plugins/dsh/plugin.mjs`（带 pilothash 注释） |
| `app.asar/dsh/node_modules/@deepseek-ai/dsh-base/cordis.patch.yml` | base 层 **96** 行（LLM 适配器、session/日志、tools 注册表、agent-loop、沙箱/审批、技能、subagent、workflow、web、spill、compaction…） |
| `.../dsh-web-app/cordis.patch.yml` | web 层 **117** 行（webserver、web-runtime、connection、51 个 `ui-*` 客户端行、office-to-pdf、terminal/files 侧栏…） |
| 7 个实验 bundle | 合计 20 行（agent-team 7、ralph 3、session-search 3、inspector 2、worktree 2、cot-translation 2、auto-review 1） |
| bundle 层并集 | **203** 个唯一 entry id（96+117+20 去重）；再叠加 2 个 provider bundle（各 2 行）=≈207，加用户/home patch 后 ≈210 条目 |
| `~/.dsh/profiles/desktop/.plugin-manager/logs/operation-*/pnpm.log` | 插件管理器的 pnpm 事务日志（壳内置 pnpm 执行插件事务） |
| `~/.dsh/profiles/{web,superagent}/` | 另两个 profile：`web`=base+web-app（`patchReload: live`）；`superagent`=base+web-app+本地原型 `dsh-superagent`（link `/Users/yuchuan.gu/Workspace/dsh-superagent-prototype`，写死 `port: 3082`） |

**插件能挂载什么【事实】**：Cordis 插件就是一个 ESM 模块 `export default function apply(ctx, config)`（本机第三方插件原文：`/Users/yuchuan.gu/.loongsuite-pilot/plugins/dsh/plugin.mjs:55`，只用 `ctx.on('session/created'|'session/event')` + `ctx.logger`）。可挂载面（源码/README 归纳）：

- **工具**：`ctx.tools.register(defineTool(...))`（`dsh-tool-fs` 等）。
- **提示词**：`ctx.systemPrompt.section({name,order,text})`。
- **服务**：任意 `ctx.<service>`；文档里的 seam 键：`ctx.sessions` `ctx.systemPrompt` `ctx.tools` `ctx.agents` `ctx.agentLoop` `ctx.llm` `ctx.sessionProjections` `ctx.skills` `ctx.commands` `ctx.jobs` `ctx.fs` `ctx.shell` `ctx.sandbox` `ctx.terminals` `ctx.agentTeams`（均在 asar 内被引用）。
- **事件**：`ctx.on(...)`（`session/*`、`agent/*`、`fs/*`、`tools/*`、`telemetry/*`）。
- **渠道/传输**：`dsh-host-webserver`（HTTP 路由注册）、`connection`、`api-gateway`。
- **UI**：双面包声明 `dsh.client`（`platform/inject/external/immediately`），node 半边把 roster 组装成 `window.__DSH_BOOT__`（`dsh-client-modules/lib/index.js` 注释与 `client.js`）。
- **模型**：`ctx.llm` 适配器（`dsh-llm-deepseek-api-key`、`dsh-llm-deepseek-account`、`dsh-llm-pi-ai`）。
- **技能**：`ctx.skills.registerProvider`（`dsh-skill-filesystem`：project/custom/user 根 + `bundledSkillDir`/`DSH_BUNDLED_SKILL_DIR`）。
- **审批/权限**：`dsh-user-approval`、`dsh-permission-presets`（见 §7）。
- 【事实】桌面版还额外挂一个"应用自有组合"插件 `desktop-office`：`dsh-desktop-host/lib/index.js:78-88,331-335`，把 `dsh-tool-workspace-dependencies` + `dsh-skill-office`（assetRoot=`runtime/primary-runtime/office-skills`、node=payload 里的 node、cli=libreoffice-kit/lib/cli.js）挂进 desktop profile。
- 【推论】"插件能挂什么"没有白名单——只要 Cordis 上下文里有该 seam 就能挂；真正约束是"服务可用性驱动激活"（base patch 注释：Row order carries no load semantics (activation is service-availability driven)）。

## 5. 子代理、Agent Teams、workflow、ralph 的实际形态（问题 4）

- 【事实】subagent 是**一个包 + 多实例**：`dsh-tool-subagent` 用 `config.toolName` 决定工具名、`config.provider` 决定后端（`dsh-tool-subagent/lib/index.js:359,382,489-493`；base patch 的两行 `provider: spawn → toolName: subagent`、`provider: fork → toolName: subagent_fork`）。
- 【事实】profile 追加的两个 provider（`~/.dsh/profiles/desktop/node_modules/@deepseek-ai/dsh-subagent-{codex,claude-code}/cordis.patch.yml`）：各自 insert 一个 provider 行 + 一个 `dsh-tool-subagent` 行，`provider: codex|claude-code`、`toolName: subagent_codex|subagent_claude_code`、`maxDepth: provider-managed`。
- 【事实】`dsh-subagent-cli` 是本地原型 bundle（link 到 `~/Workspace/AI/dsh-subagent-cli`），只注册 provider（`driver: claude`、`providerName: claude`），注释写明"Host plane only；模型侧工具由 Agent Preset 单独授予"。
- 【事实】子代理语义（`dsh-subagent/README.zh.md`）：本地 spawn/fork 保留持久会话、可继续对话；ACP / SDK / Codex / Claude Code 执行一次；所有子级共享准入、取消、所有权、发现机制。**子代理审批固定为 never**：`dsh-subagent/lib/index.js:413` `approvalPolicy: parent.ctx.get("approval")===undefined ? undefined : "never"`。
- 【事实】Agent Teams 层（`app.asar/dsh/node_modules/@deepseek-ai/dsh-experimental-agent-team-profile/cordis.patch.yml`）：先**禁用** `tool-subagent-control`、`tool-subagent-list-agents`、`tool-subagent`、`tool-subagent-fork`，再 insert `agent-team`（`maxMembers: 8`、`maxTasks: 256`、`maxMessageBytes: 65536`、`disposalTimeoutMs: 5000`）、`tool-agent-team`（`freshProvider: spawn`、`forkProvider: fork`）、`ui-agent-team`。
- 【事实】Team 语义（`dsh-experimental-agent-team/README.zh.md`）：一个 Lead + 具名 teammate + 直接 inbox 消息 + 持久共享任务板；发送走目标 agent 的 inbox，可冷恢复已存储 teammate；需要持久会话存储。任务板工具是 `team_task_{create,get,list,update}`（CAS 用 revision；带 `blocked_by` 与 `write_scopes` 提示）。
- 【事实】workflow：`dsh-tool-workflow` 默认 `toolName: "workflow"`（`config.toolName` 可改，`dsh-tool-workflow/lib/index.js:67,262-271`），config 另有 `maxResultChars`、`enableRunInBackground`；执行引擎是 `dsh-workflow-ptc`（`provider: spawn`）+ `dsh-ptc-runtime-node`，脚本内 hooks 为 `agent()/pipeline()/parallel()/phase()/log()/args`。
- 【事实】ralph（`dsh-experimental-ralph-bundle/cordis.patch.yml`）：包在一个隔离组里（`name: cordis:group`、`group: true`、`isolate: { workflowEngine: true }`），内含 `workflow-ptc(provider: spawn)` + `tool-ralph(subagentProvider: spawn, maxRounds: 64)`。语义（README.zh.md）：前台跑多轮**全新**子 agent，每轮只拿"上一轮有界报告 + 共享工作区"，父对话与旧子会话永不复制；完成/阻塞/达上限即返回；报告不独立验证。
- 【事实】本机实况：`~/.dsh/state/superagent/{orchestrator.sqlite,superagent.lock}` 存在且 09:51 有更新，`3082` 端口也在监听（superagent profile 的固定端口）——说明**还有一个 superagent 原型进程在跑**；但本会话所在的 GUI 是 `127.0.0.1:59904`（见 §6），即 desktop profile 那条链路。
- 【推论】"subagent 还是 Agent Teams"在本机不是二选一：base 挂 subagent 工具、agent-team bundle 默认把它们禁掉并改挂 Team 工具；而本会话同时看到两套（47 个工具里既有 `subagent*` 又有 `spawn_teammate` 等），说明**该 preset 把两套都授权了**（工具可见性最终由 agent preset 决定，不是 profile 默认值）。

## 6. 渠道与 UI：web 与 desktop 的关系（问题 5）

- 【事实】CLI 只有一个启动器 `dsh [--profile] <name>`，`dsh <name>` 是缩写；`dsh web` = `dsh --profile web`；`dsh plugin` 转发 pnpm；`--dump-config/--dump-default-config/--dump-config-schema/--patch/--from-default-profile`（`app.asar/dsh/node_modules/@deepseek-ai/dsh/lib/bin.js:39-49,105`）。
- 【事实】launcher 明确**拒绝** desktop profile：`profile "desktop" is managed exclusively by the Electron application`（`dsh/lib/bin.js:35-37`）。
- 【事实】内置模板只有 5 个：`web:[base,web-app]`、`headless:[base,headless]`、`sdk:[base,sdk-app]`、`sdk-minimal:[sdk-minimal]`、`acp:[base,acp-app]`（`dsh-app-boot/lib/index.js:552-556`）。
- 【事实】web 服务：`webserver` 行 `host: ctx.webStartup.host ?? '127.0.0.1'`、`port: ctx.webStartup.port ?? 3080`（`dsh-web-app/cordis.patch.yml:179-184`）；`web-startup` 注入 `cmdlineArgs` 并提供 `webStartup` 服务；`--no-open`、`--port`、`--host` 是 web 表层 flag（`dsh-web-app/README.zh.md` §使用本包：启动时打印带 token 的 URL，浏览器取签名 cookie 后重定向）。
- 【事实】鉴权真实存在：`curl http://127.0.0.1:59904/` → **401**，body `"dsh web authentication required; reopen the URL printed by dsh web."`（字符串出处 `dsh-client-connection/lib/index.js`）。
- 【事实】desktop ↔ web 的实链路（`dsh-desktop-host/lib/index.js:218-250,342-349`）：Electron 以 Electron-Node 模式起子进程 Host → Host `runProfile({ profile: "desktop", args: ["--no-open","--host","127.0.0.1","--port","0"] })` → 启动后 `ctx.connection.authenticatedUrl(...)` 生成带凭据 URL，经 IPC 回 `{type:"ready", url}` 给 Electron → 窗口 loadURL 该 URL。`--port 0` 即"系统分配端口"，本机结果就是 **59904**。
- 【事实】壳侧页面：注册特权 scheme `dsh-app`（`lib/main.js:7009,11880`），强制更新页 `dsh-app://shell/mandatory-update.html`、更新对话框 `dsh-app://shell/update-dialog.html`，主窗口原点 `dsh-app://app`；`lib/preload-app.cjs` 承载 `dsh-desktop:boot` 等 IPC。
- 【事实】`window.__DSH_BOOT__`：由 `@deepseek-ai/dsh-client-modules` 的 node 半边扫描 Loader 条目里声明 `dsh.client` 的双面包，组装成启动清单注入窗口；浏览器半边 `client.js` 负责物化（`dsh-client-modules/lib/index.js:35,105-126`；`dsh-web-app/cordis.patch.yml` 注释直说 "`dsh.client` rows are the browser roster the modules node half scans into `window.__DSH_BOOT__`"）。
- 【推论】desktop 不是"另一个 UI"：它是 web 应用的一个**载体（carrier）**——同一份 profile（`desktop`）跑同一套 web 运行时，Electron 只负责窗口、菜单、更新、文件选择、签名 cookie 注入与 `dsh-app://` 本地资源；因此 web 与 desktop 的能力差异来自 **profile 层的几行 patch**（如 `desktop-product-telemetry`、`product-analytics` 仅在 `profileContext.name === 'desktop'` 时启用）。

## 7. 沙箱与审批（问题 6）

- 【事实】文件沙箱模式只有 3 个取值：`SANDBOX_MODES = ["read-only","workspace-write","danger-full-access"]`（`app.asar/dsh/node_modules/@deepseek-ai/dsh-sandbox-policy/lib/index.js:26-30`）；默认 `"read-only"`（同文件 schema `.default("read-only")`），而 base 层实际注入的是 `process.env.DSH_PERMISSION_MODE ?? 'workspace-write'`、`workspaceRoot: process.cwd()`（`dsh-base/cordis.patch.yml` 的 `- id: sandbox-policy`）。
- 【事实】执行侧：`dsh-fs-sandbox` 按会话模式放行文件变更——`read-only` 全拒（`FS_SANDBOX_DENIED`）、`workspace-write` 只允许 workspace root 或平台临时区（`/tmp`、`os.tmpdir()`）、`danger-full-access` 不限制；读取不受限；被拒时工具层回 `[sandbox: file access denied under <mode> mode]` + 一次"更宽模式重试"提示（`dsh-fs-sandbox/README.md`）。
- 【事实】进程沙箱后端：`dsh-sandbox-local`（`- id: sandbox`）+ `dsh-bash-sandbox`（mac/linux，`timeoutMs: 60000`）/`dsh-pwsh-sandbox`（win）。
- 【事实】审批策略只有 2 个取值：`APPROVAL_POLICIES = ["ask","never"]`（`dsh-user-approval/lib/index.js:37`）；`never` 时模型侧看到固定句 `NEVER_SENTENCE = "Approval prompts are disabled in this session: actions that require approval are rejected automatically — do not request sandbox escalation (do not set \`sandbox_permissions\`)."`（同文件:39）——与本会话 runtime context 的句子一致。
- 【事实】preset 表（`dsh-base/cordis.patch.yml` 的 `- id: permission`）：`read-only → {sandbox:read-only, approval:ask}`、`workspace-write → {workspace-write, ask}`、`danger-full-access → {danger-full-access, never}`。base 里 approval 行用表达式：`danger-full-access ? 'never' : 'ask'`。
- 【事实】本会话为何是 never：子代理强制注入 `approvalPolicy: "never"`（`dsh-subagent/lib/index.js:413`），且子代理权限面在启动时即固定、不能从内部提权（本会话 runtime context 原文）。
- 【事实】沙箱提权的唯一路径是"同一轮内一次性更宽模式重试 + 审批"，审批不可用时自动拒绝（`dsh-user-approval` + `dsh-fs-sandbox/README.md` 的 escalation 段落）。

## 8. 与 `docs/architecture.zh.md` 的一致性抽查（问题 7）

抽查对象：`/Users/yuchuan.gu/Workspace/AI/deepseek-harness/docs/architecture.zh.md`（本地 checkout，master `ce7073d5`）。
【前置提醒】**该 checkout 不含 App 的构建提交 `d4db512a`**（`git cat-file -e` 失败），所以"文档 ✔"只代表该主张在安装版里成立，不保证逐字同版本。

| # | 文档论断 | 结论 | 本机证据 |
|---|---|---|---|
| 1 | §Cordis：产品每一部分都是插件（含模型适配器、工具注册表、会话日志、agent loop），注册是可在卸载时撤销的副作用 | **成立** | base patch 里 `llm-deepseek`、`tools`、`session-persistence-jsonl`、`agent-loop` 都是普通行（`dsh-base/cordis.patch.yml:138,499,511,525`）；撤销可见 `dsh-tool-subagent/lib/index.js:489-493`（`const unregister = ctx.tools.register(...)` → `unregister()`） |
| 2 | §Profile 与组合包：层序 = bundles → profile `cordis.patch.yml` → home patch → `--patch`；"一条 patch 按 id 定位条目并替换其整个 config" | **成立（逐字命中）** | `dsh-app-boot/lib/index.js:1131`（home 层 `join(context.home,"cordis.patch.yml")`）；`:3207` 注释 "Bundle, profile, home, and CLI layers apply in that order. A patch config replaces the whole config."；本机 `~/.dsh/cordis.patch.yml` 确实作为 home 层存在 |
| 3 | §Profile：template profile 为 `web`/`headless`/`sdk`/`sdk-minimal`/`acp` | **成立，但有补充** | `dsh-app-boot/lib/index.js:552-556` 完全一致；但本机还有 `desktop`（Electron 独占，CLI 拒绝）与 `superagent`（本地原型）两个非模板 profile，文档未列（合理，属非发行模板） |
| 4 | §Profile：HMR 由 YAML 控制——base 启用"仅监视配置"、Headless/SDK/ACP 禁用、sdk-minimal 不含 | **成立** | base `- id: hmr` + `config.root: []`；`dsh-headless/cordis.patch.yml:32-33`、`dsh-sdk-app`、`dsh-acp-app` 均为 `id: hmr` + `disabled: true`；`dsh-sdk-minimal` 不依赖 base |
| 5 | §桌面应用：Electron-Node 模式起私有 Desktop Host；Host 调共享 CLI profile runner 与**完整 Web 应用**；Desktop 默认监听系统分配端口 | **成立** | `runtime/bin/node`（`ELECTRON_RUN_AS_NODE=1`）；`dsh-desktop-host/lib/index.js:226-240`（`runProfile({profile:"desktop", args:[… "--port","0"]})`）；`:342` `authenticatedUrl`；本机实测端口 59904（非 3080/3082 固定值） |
| 6 | §核心包：`ctx.sessions`/`ctx.systemPrompt`/`ctx.tools`/`ctx.agents`/`ctx.agentLoop`/`ctx.llm` | **成立** | asar 内逐一命中（如 `ctx.systemPrompt` 82 处、`ctx.agents` 161 处、`ctx.agentLoop` 命中 `dsh-agent-loop/lib/index.js`） |
| 7 | §能力 seam：实验性 Agent Teams 通过显式启用的 `ctx.agentTeams` 提供 roster/任务/inbox | **成立** | `dsh-experimental-agent-team/lib/index.js` 暴露 `agentTeams`；本机通过 `dsh-experimental-agent-team-profile` bundle 显式加入（默认 base 不含） |
| 8 | §会话日志：v1 起用 `session.vN.jsonl[.zstd]` | **成立** | 本机会话目录里：`session.v4.jsonl.zstd` 84 个、`session.v3.jsonl.zstd` 98 个、无版本号 `session.jsonl.zstd` 11 个（`find ~/.dsh/sessions -name "session*.jsonl*"`）——与"v0 无版本号、v1 起用 `session.vN`"一致；本会话 header `version:4` |

## 9. 未取到 / 限制

- 【退化方案】`npx @electron/asar` 解包未使用（避免拉重型依赖）：改用 node 自写 asar 头解析（8 字节头 + pickle，数据区起点 4761020），并从文件表按 offset 读取任意文件，结论等价。
- 【未取到】任何凭据值：`~/.dsh/.credentials.yaml` 只记录"存在该文件（0600）"，未读取内容；本报告不含 token/API key。
- 【未取到】进程级证据：`ps` 被沙箱拒绝（`/bin/ps: Operation not permitted`），无法直接列 DSH 进程命令行；端口事实用 `curl` 401 响应与端口分配逻辑间接证明。
- 【未取到】`dsh --profile desktop --dump-config`：CLI 拒绝 desktop profile，且启动新实例会写用户配置，故未执行；profile 状态由其 `cordis.yml`/`cordis.patch.yml`/bundle patch 静态重建。
- 【未取到】`subagent_oar`：**App asar 内 0 命中、profile 内 0 命中**（`grep -a -F subagent_oar app.asar`），本机不存在该工具；实际存在的是 `subagent`/`subagent_fork`/`subagent_claude_code`/`subagent_codex`。
- 【未取到】前端构建产物细节（`dsh-web-frontend` dist 的 hash 文件名、`renderer/assets` 清单）未逐条核对，只确认了 `dsh-app://app/assets/index-BXV-oxvX.js` 这一条运行时引用（来自崩溃日志）。
- 【限制】`docs/architecture.zh.md` 与安装版不是同一 commit（见 §8 前置提醒）；跨版本比较需另取对应 tag 的源码。
