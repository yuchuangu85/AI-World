---
title: Hermes Agent 源码深度解析：一个可复用的 Agent 运行模型
source: github.com/NousResearch/hermes-agent
commit: 8f5693af4182 (2026-10-19)
license: MIT
analyzed: 2026-10-19
---

# Hermes Agent 源码深度解析：一个可复用的 Agent 运行模型

> 分析对象：`NousResearch/hermes-agent`，MIT，约 25.5 万 star，提交 `8f5693a`（2026-10-19）。
> 代码规模：单仓 ≈20,400 个文件，其中 `agent/` 目录 319 个 Python 文件、约 6.5 MB；`tests/` 约 1,250 个测试文件。
> 本文结论来自源码与仓库内 `website/docs/developer-guide/*`；标注为"分析/推论"的段落是我的判断。

---

## 0. 结论先行

Hermes 不是"提示词套壳"，而是一台**回合制状态机 + 上下文经营系统 + 自我演化层**。三句话概括：

1. **回合是唯一执行单元**：一次用户输入 = 一个 turn；turn 内是"模型调用 → 工具执行 → 再调用"的内循环，外层受预算约束。
2. **上下文是最稀缺资源**：系统提示词字节级冻结以保缓存；压缩是唯一允许改写历史的行为；token 记账优先信 provider 真实用量。
3. **能力在边缘、核心收窄**：新能力默认走 skill / plugin / MCP，而不是加核心工具；学习靠"回合后 fork 一份自己复盘"。

架构定位一句话：**Prompt-cache-first 的同步 ReAct 内核，外套可插拔的能力与记忆外设。**

---

## 1. 系统分层

```text
┌──────────────────────────────────────────────────────────────────────────┐
│ L6 宿主面 (Surfaces)                                                     │
│   CLI · TUI(Ink) · Desktop(Electron) · Gateway(20+ 消息平台) · ACP · Cron│
├──────────────────────────────────────────────────────────────────────────┤
│ L5 会话与状态 (Session & State)                                          │
│   state.db (SQLite + FTS5) · 单一写者(gateway) · profile 隔离 · lineage  │
├──────────────────────────────────────────────────────────────────────────┤
│ L4 回合引擎 (Turn Engine)  ← AIAgent                                     │
│   preflight → prepare → assemble → call → intake → tool round → finalize │
├──────────────────────────────────────────────────────────────────────────┤
│ L3 上下文工程 (Context Engineering)                                       │
│   system prompt 三层冻结 · 双阈压缩 · 4 阶段算法 · Anthropic cache 断点   │
├──────────────────────────────────────────────────────────────────────────┤
│ L2 能力层 (Capability)                                                   │
│   Tool Registry(70+ 工具/28 toolset) · Skills · Plugins · MCP · Subagents│
├──────────────────────────────────────────────────────────────────────────┤
│ L1 模型接入 (Providers)                                                  │
│   3 种 API mode · 凭据池 · 失败分类(30+ 类) · fallback 链 · auxiliary 旁路│
├──────────────────────────────────────────────────────────────────────────┤
│ L0 运行时 (Runtime)                                                      │
│   deadline 层 · 中断控制 · 终端后端(7 种) · secret scope · 审批           │
└──────────────────────────────────────────────────────────────────────────┘
```

| 层 | 关键源码 | 职责 |
|---|---|---|
| L4 回合引擎 | `agent/conversation_loop.py`、`agent/turn_*.py` | 一次 turn 的全部控制流，不允许任何层越权写历史 |
| L3 上下文 | `prompt_builder.py`、`system_prompt.py`、`context_compressor.py` | 组装、冻结、压缩、缓存 |
| L2 能力 | `tools/registry.py`、`model_tools.py`、`toolsets.py` | 工具发现、schema、调度、执行 |
| L1 模型 | `auxiliary_client.py`、`provider_registry.py`、`error_classifier.py` | 路由、重试、降级 |
| L0 运行时 | `agent/deadline.py`、`interrupt_control.py`、`tools/environments/` | 不挂死、可打断、可隔离 |

设计哲学（原文）：

> **Per-conversation prompt caching is sacred.**（每会话的 prompt 缓存是神圣的）
> **The core is a narrow waist; capability lives at the edges.**（核心是窄腰，能力长在边缘）

这两句不是口号，而是**代码评审的硬性依据**——仓库里专门写了"会被驳回的提交清单"：没有真实消费者的 hook、为普通配置新建环境变量、terminal 能干的事却新增核心工具，一律驳回。

---

## 2. 回合引擎：一个 turn 的完整状态机

### 2.1 真实主循环

```python
# agent/conversation_loop.py::_run_conversation_turn（简化但忠实于控制流）
while ((s.api_call_count < agent.max_iterations and agent.iteration_budget.remaining > 0)
       or agent._budget_grace_call):
    begin_iteration()                      # 预算/重试状态初始化
    prepare_iteration()                    # 注入预算警告、处理 steer
    assemble_api_request()                 # 消息 + 工具 schema → 请求体
    if run_preflight_gate(): continue      # 调用前压缩闸门（也可能直接结束 turn）
    announce_api_call()

    retry_loop:                            # 单次 API 调用及其恢复
        nous_rate_limit_guard()            # 限流预判
        build_api_request()
        perform_api_call()                 # 可中断的真实 HTTP
        check_api_response()
        except → handle_api_error()        # 分类 → 重试 / 换 key / 换模型 / 压缩

    if response.tool_calls:
        run_tool_round()                   # 执行 → 追加 tool 消息 → 回到 while
    else:
        finish_text_response()             # 触发 stop gates，可能"拒绝收尾"
finalize_turn()
```

三个非教科书细节：

1. **`_budget_grace_call`**：预算耗尽后仍给一次"恩典调用"，让模型能体面收尾，而不是被硬切断在半句话上。
2. **`run_tool_round` 不是"执行完就继续"**：内部做参数校验、去重、截断、并发分段、结果分类，失败走 guardrail。
3. **`finish_text_response` 不保证收尾**：`apply_stop_gates` 能把"看起来完成的答案"降级为 interim，追加一条 user nudge 后 `continue`——这就是它的自我验收。

### 2.2 13 个阶段（每个阶段一个文件）

| 序 | 阶段函数 | 文件 | 干什么 |
|---|---|---|---|
| 1 | `build_turn_context` | `turn_context.py` | 建上下文、恢复/构建 system prompt、首轮压缩 |
| 2 | `play_prelude` | `turn_scripted_prelude.py` | 首次模型调用前先跑脚本化工具 |
| 3 | `begin_iteration` | `turn_iteration_prep.py` | 迭代计数、预算检查 |
| 4 | `prepare_iteration` | 同上 | 预算警告、`/steer` 插在最新 tool 结果之后 |
| 5 | `assemble_api_request` | `turn_request_assembly.py` | 组装 messages + tools + MoA 上下文 |
| 6 | `run_preflight_gate` | `turn_preflight_gate.py` | 调用前压缩/溢出预判 |
| 7 | `perform_api_call` | `turn_api_call.py` | 可中断 HTTP，流式/非流式 |
| 8 | `check_api_response` | `turn_response_check.py` | 空响应、截断、拒答检查 |
| 9 | `normalize_model_response` | `turn_response_intake.py` | 统一成内部 message 形状 |
| 10 | `run_tool_round` | `turn_tool_round.py` | 一轮工具调用 |
| 11 | `finish_text_response` + `apply_stop_gates` | `turn_final_response.py`、`turn_stop_gates.py` | 收尾前的四道闸门 |
| 12 | `finalize_turn` | `turn_finalizer.py` | 落盘、刷记忆、触发复盘、微压缩 |
| 13 | `handle_outer_loop_error` | `turn_loop_errors.py` | 循环级异常兜底 |

> 工程含义：**一个阶段一个文件，平均约 600 行**。改溢出不会碰到压缩，改压缩不会碰到工具调度。这是几十万行级 Agent 项目还能维护的关键组织手法。

### 2.3 stop gates：怎么防止"假完成"

模型输出纯文本时，四道闸门可能拦下这次收尾：

| 闸门 | 触发条件 | 行为 |
|---|---|---|
| `_late_lsp_verdicts` | 本轮编辑过代码，语言服务器有延迟诊断 | 追加 nudge，继续回合 |
| `_verify_on_stop_nudge` | 开启 verify-on-stop | 要求先自证再收尾 |
| `pre_verify` hook | 插件在代码编辑后介入 | 同上 |
| `_kanban_stop_nudge` | 看板 worker 未走终态 | 强制补齐终态 |

被拦下的答案不丢——存进 `pending_verification_response`，预算耗尽时作为兜底返回。**质量闸门与"永不空手而归"的折中。**

---

## 3. 上下文工程（最硬的部分）

### 3.1 系统提示词三层冻结

```text
stable   ── 身份(SOUL.md) → 工具使用准则 → 编码守则 → 平台提示
context  ── 调用方 system_message → 项目上下文(.hermes.md / AGENTS.md /
            CLAUDE.md / .cursorrules) → git worktree 快照 → 运维指令
volatile ── 技能索引 → MEMORY.md 快照 → USER.md 快照 → 外部记忆块
            → 时间/会话/模型/供应商 → 运行时环境(host / home / cwd)
```

拼接顺序 = `stable → context → volatile`，且**整个会话期间字节不变**。

顺序有讲究：
- 项目共享上下文排在依赖 worktree 的内容之前 → 同项目不同 worktree 的会话共享同一段 prompt 前缀，正好是最长前缀缓存能复用的部分。
- 运行时环境块永远压最后，避免项目文件里的路径样例被误认为"当前目录"。
- 平台提示（CLI 别说 Markdown、Telegram 别发长消息）**不算身份字段**：桌面切 TUI 时系统提示不动，改用一条一次性 user 通道的 note 传达。

### 3.2 缓存不变式

代码里写死的不变式：

> 系统提示词在会话生命周期内字节稳定；**唯一的上下文变更是压缩**。
> 任何必须中途注入的内容，走 **user 消息或 tool 结果**，绝不进 system prompt。

后果很具体：
- `/skills install` 默认**下个会话才生效**，要立刻生效得加 `--now`。
- 子目录 `AGENTS.md` 的提示不插进 prompt，而是追加到 tool 结果尾部（超 32,000 字符头尾截断）。
- 插件 `pre_llm_call` 返回的上下文，拼进本回合的 user 消息。
- 记忆在会话中被写入只落盘，**不改已构建好的 prompt**。

代价：产品上必然出现"我改了设置怎么没反应"。收益：长会话成本线性而非平方级。Hermes 明确选了后者。

### 3.3 双层压缩

```text
        入站消息
           │
           ▼
  ┌────────────────────┐  85% 上下文  ← 网关会话卫生（粗估，回合间兜底）
  │ Gateway Hygiene    │
  └─────────┬──────────┘
            ▼
  ┌────────────────────┐  50% 上下文  ← Agent ContextCompressor（真实 token）
  │ ContextCompressor  │
  └────────────────────┘
```

为什么两层：网关那层是"防炸"安全阀，处理 Telegram 里过夜堆积的会话；Agent 这层才是日常管理，能拿到 provider 返回的准确 usage。把网关也设成 50% 会让每个长会话一开始就被压缩——源码注释明确写了这是踩过的坑。

### 3.4 压缩四阶段算法

```text
[0..2]   保护头（system + 首轮）
[3..N]   ── 中段：摘要 ──
[N..end] 保护尾（按 token 预算回走，上限 = 上下文窗口 20%）
```

**Phase 1｜剪枝（不调 LLM）**
保护尾之外的旧 tool 结果，超过 200 字符就替换成 `[Old tool output cleared to save context space]`。文件内容、终端输出、搜索结果通常占大头，这一步能省掉近一半。
细节：模型"还没回应的那一轮工具"结果必须原样保留——它刚请求的数据不能被立刻抹掉。

**Phase 2｜定边界**
尾部保护**按 token 预算走**，不是按条数。`protect_last_n`（默认 20）只是下限；边界要对齐到 `assistant(tool_calls) → tool...` 的完整组，绝不切断配对。

**Phase 3｜结构化摘要**
用 auxiliary 小模型生成固定八段式：

```markdown
## Goal / ## Constraints & Preferences / ## Progress(Done|In Progress|Blocked)
## Key Decisions / ## Relevant Files / ## Next Steps / ## Critical Context
```

预算 = `被压缩内容 × 0.20`，下限 2,000 token，上限 `min(上下文 × 5%, 12,000)`。

**Phase 4｜拼装**
头 + 摘要 + 尾，再跑 `_sanitize_tool_pairs()`：孤立 tool_result 删掉，丢了结果的 tool_call 补 stub。

**迭代压缩**：第二次压缩不是重来，而是把上次摘要交给模型"更新"——条目从 In Progress 挪到 Done，新进展追加，过期信息删除。这是长会话不丢线索的关键。

### 3.5 token 记账：先信 provider，再信估算

每个压缩闸门先问 **usage anchor**：provider 上次返回的 prompt/completion tokens + **仅对"此后新增消息"的粗估**。

这个设计解决三个现实问题：
- 网关每回合从 DB 重读历史 → 用内容指纹绑定"计价的那份 transcript"，重读后仍对得上。
- `--resume` 或桌面按回合起进程 → anchor 持久化在 session 行上。
- 没有 anchor 时（首请求、rewind）→ **先等一次请求**，让 provider 自己裁决，而不是凭估算就压缩。估算再大也不能证明这次请求会失败。

图片价格按 `model@host` 记录**从 provider usage 反推**的单图成本，而不是套厂商公式；未知时先按 1,500 估。

### 3.6 失败冷却阶梯

摘要调用失败不能每回合重试，源码维护分级冷却：

| 失败类型 | 冷却 |
|---|---|
| 超时 / 流停滞 | 60s → 300s → 900s（指数，且不短于 `context_timeout_seconds`） |
| 输出被长度截断 | 独立计数，同样 60/300/900s |
| JSON 解析 / 流关闭 / 空内容 | 固定 30s |

连续两次同因停滞 → **跳过 LLM，直接提交确定性兜底摘要**，宁可丢中段，也不让会话涨到 `compression_exhausted` 后被网关整段重置。

### 3.7 三种压缩形态

| 形态 | 机制 | 适用 |
|---|---|---|
| In-place compaction | 旧行 `active=0` 归档，保留内容以 `active=1` 插入 | 保持单一稳定 session id |
| Micro-compaction | 回合后对尾部做小步清理 | 不惊动主压缩 |
| Native compaction | 交给 OpenAI Responses / Codex 服务端 | 支持的 provider 路由 |

### 3.8 Anthropic 缓存断点

`system_and_3` 策略：system 块 + 最近 3 个 user 轮打 `cache_control`。配合上面的"冻结"不变式，长会话命中率才立得住。

---

## 4. 工具层

### 4.1 注册与暴露是两件事

```python
registry.register(
    name="example_tool", toolset="example",
    schema={...},                       # 送进模型的 JSON Schema
    handler=lambda args, **kw: ...,     # 返回 JSON 字符串
    check_fn=check_requirements,        # 可达性 / 是否具备条件
    requires_env=["EXAMPLE_API_KEY"],
)
```

- `tools/*.py` 只要在模块级调过 `registry.register()` 就被自动发现——没有手工 import 清单。
- **注册 ≠ 暴露**：只有被某个 toolset 命名的工具才会送给模型，默认包是 `_HERMES_CORE_TOOLS`。
- `check_fn` 只回答"这个 profile 能不能用"，**绝不按会话面区分**（GUI 专属工具要走独立 toolset）。

28 个 toolset：`browser / clarify / code_execution / cronjob / debugging / delegation / discord / discord_admin / feishu_doc / feishu_drive / file / image_gen / kanban / memory / messaging / moa / rl / safe / search / session_search / skills / terminal / todo / tts / video / vision / web / yuanbao`。

### 4.2 一次工具调用的 8 步

```text
模型返回 tool_calls
 → ① 解析参数（容错 JSON 修复）
 → ② 查 INLINE 表：todo / memory / session_search / delegate_task 由 agent 直接处理
 → ③ 否则 registry 找 handler
 → ④ pre_tool_call 插件 hook
 → ⑤ 危险命令审批（approval）
 → ⑥ 执行（带 deadline，可取消）
 → ⑦ post_tool_call hook（一个 tool_call_id 只发一次）
 → ⑧ 结果追加进历史（按原文顺序，与完成顺序无关）
```

**结果契约**：所有 handler 返回 JSON 字符串；需要回图给模型时返回特殊信封
`{"_multimodal": True, "content": [text, image_url...], "text_summary": str}`。

### 4.3 并发分段规划器（关键设计）

不是"多个工具就并发"，而是先把整批切成有序段：

```python
_plan_tool_batch_segments(tool_calls)
# -> [("parallel", [..]), ("sequential", [..]), ...]
```

规则：
- 遇到 `_NEVER_PARALLEL_TOOLS`（含交互型 `clarify`）、参数解析失败、或非并发安全的 MCP 工具 → **屏障**，切断当前并行段。
- 带路径的工具做**读写冲突检测**：读读可并发；任一方是写且路径重叠 → 关闭当前段，该调用另起一段排在后面。
- 一段不足 2 个调用 → 降级为顺序执行。
- 相邻顺序段合并。

结论：**结果顺序与副作用边界等价于全顺序执行**，但读多写少的典型场景仍能提速。

### 4.4 循环护栏

| 指标 | 警告 | 硬停 |
|---|---|---|
| 完全相同的失败 | 2 次 | 5 次 |
| 同工具失败 | 3 次 | 8 次 |
| 无进展轮次 | 2 次 | 5 次 |

交互平台默认只警告；**无人值守平台（gateway / cron）默认开硬停**——因为没人能打断它。

### 4.5 能力放置的阶梯（Footprint Ladder）

新增能力时按"最省"的一级解决，不越级：

```text
扩展现有代码 → CLI 命令 + Skill → 服务门控工具(check_fn) → 插件
             → 目录里的 MCP server → 新核心工具
```

最后一级门槛极高，理由是物理的：**每个工具 schema 在每次 API 调用里都要发一遍**，工具越多，固定成本越高、注意力越散。

---

## 5. 能力演化层：记忆、技能、复盘

### 5.1 记忆 vs 技能：一条清晰的职责线

| | Memory | Skill |
|---|---|---|
| 回答 | 用户**是谁**、当前处境 | 这类任务**怎么做** |
| 存储 | `MEMORY.md`（agent 笔记）+ `USER.md`（用户画像） | `skills/<name>/SKILL.md` + `references/ templates/ scripts/` |
| 进入 prompt | 会话开始冻结快照（volatile 层） | 索引常驻；正文按需 `skill_view` |
| 会话中写入 | 只落盘，不改 prompt | 通过 `patch` / `write_file`，下轮生效 |

源码原话：*"Memory is the narrow exception for facts that apply to EVERY session regardless of task."*

### 5.2 SKILL.md 的结构与硬标准

```yaml
---
name: pdf-report
description: Generate a paginated PDF report from CSV.   # ≤60 字符，一句话，句号结尾
version: 1.2.0
author: <human first>
platforms: [macos, linux]
metadata:
  hermes: { tags: [...], category: ..., related_skills: [...], config: {...} }
---
```

硬性标准（有测试强制）：
1. `description` ≤ 60 字符、一句话、句号结尾、不用营销词。
2. 正文只能用原生工具名（`read_file` 而不是 `cat`，`patch` 而不是 `sed`）。
3. `platforms:` 要与脚本实际用的 POSIX 原语对得上。
4. 章节顺序固定：When to Use → Prerequisites → How to Run → Quick Reference → Procedure → Pitfalls → Verification。
5. 技能加载工具**不允许分页**（`offset/limit`）——模型读了第一页就会跳过剩下的。

### 5.3 技能生命周期（Curator）

```text
active ──(stale_after_days 无活动)──▶ stale ──(archive_after_days)──▶ archived
  ▲                                                                     │
  └──────────────────────── restore ◀───────────────────────────────────┘
        pinned：豁免一切自动流转与 LLM 复审
```

- 触发方式是**闲置触发**，不是 cron 守护进程：agent 空闲且距上次运行超过 `interval_hours` 才跑。
- **只动 `created_by: "agent"` 的技能**；内置与 hub 安装的免谈。
- **永不删除，只归档**（`~/.hermes/skills/.archive/`，可恢复）。
- 每次运行前打 tar.gz 快照，支持 rollback。
- 用量记在 `.usage.json`：`use_count / view_count / patch_count / last_activity_at / state / pinned`。

### 5.4 学习闭环：回合后 fork 一份自己

每回合结束后，主线程可起一个 daemon 线程，**fork 一个 AIAgent**，重放本回合快照，只问一个问题：

> "**Review the conversation above and update the skill library. Be ACTIVE — most sessions produce at least one skill update, even if small. A pass that does nothing is a missed learning opportunity, not a neutral outcome.**"

关键设计：

| 机制 | 作用 |
|---|---|
| fork 继承父的 provider/model/凭据/缓存提示词 | 命中同一段前缀缓存，复盘几乎不额外花钱 |
| 只写 memory / skill 存储 | **绝不碰主会话与主 prompt 缓存** |
| 派发侧工具白名单 | fork 不能执行危险工具 |
| 优先级：改已加载技能 → 改已有伞形技能 → 加支撑文件 → 才新建 | 防止技能库退化成一堆一次性条目 |
| Read-before-write 强制 | patch 前必须本轮 `skill_view` 一次，防止凭记忆改写 |
| "Nothing to save." 是合法答案，但不该是默认 | 明确对抗"复盘即空转" |

**用户不满是最高优先级信号**：*"stop doing X"、"this is too verbose"、"just give me the answer"* 这类话必须落到技能正文，而不只是记忆里——记忆记"你是谁"，技能记"这类活该怎么给你干"。

---

## 6. 委托层（Subagent）

```python
delegate_task(tasks=[{goal, context, output_schema}, ...])   # 单任务或并行批
```

| 维度 | 父 | 子 |
|---|---|---|
| 上下文 | 完整历史 | **全新**，只从 goal + context 构建 |
| task_id | 有 | 独立（终端会话、文件缓存隔离） |
| 工具 | 全量 | 父的 toolset **减去禁止子用的** |
| 预算 | `max_iterations` 默认 500 | `delegation.max_iterations` 默认 50，独立计 |
| 可见性 | 只见调用与摘要 | 中间工具调用、推理过程**不上浮** |

对外暴露的是**不可变契约**而不是 `AIAgent` 对象：`SubagentLaunchRequest / SubagentHandle / SubagentResult / SubagentStatus`。插件只能通过这些值对象操作，拿不到内部实例。

并发上限 `delegation.max_concurrent_children`、嵌套深度 `max_spawn_depth` 会**动态重写进工具 schema 的 description**——让模型看到的限制就是真实限制，避免它按旧数字规划。

---

## 7. 模型接入层

### 7.1 三种 API mode

| mode | 用于 | 客户端 |
|---|---|---|
| `chat_completions` | OpenAI 兼容端点（OpenRouter、自定义） | `openai.OpenAI` |
| `codex_responses` | OpenAI Codex / Responses API | `openai.OpenAI`（Responses 形状） |
| `anthropic_messages` | Anthropic 原生 | `anthropic.Anthropic` + 适配器 |

解析优先级：显式参数 > provider 判定 > base_url 启发式 > 默认。
三者**进出都收敛到同一套内部 message 形状**（OpenAI 风格），所以回合引擎不需要知道底下是哪个协议。

### 7.2 失败分类：30+ 种，而不是 try/except

`FailoverReason` 枚举是这套系统最"脏"也最值钱的部分——把真实世界的烂摊子编码成了策略表：

| 类别 | 例子 | 策略 |
|---|---|---|
| 可刷新 | `auth`(401/403) | 刷凭据 / 轮换 key 后重试 |
| 该退避 | `rate_limit`、`overloaded`(503/529)、`server_error` | 指数退避 |
| 该换模型 | `model_not_found`、`model_entitlement`、`upstream_rate_limit` | 走 fallback 链 |
| 该压缩 | `context_overflow`、`payload_too_large`(413) | 压缩而非切模型 |
| 该改写请求 | `image_too_large`（缩图）、`image_corrupt`（剥图）、`role_alternation`（合并同角色）、`multimodal_tool_content_unsupported`（降级文本） | 本地修复后重试 |
| 别重试 | `content_policy_blocked`、`ssl_cert_verification` | 直接失败并给指引 |
| 特殊 | `incomplete_response`（Codex 连续三轮只吐 reasoning） | 换 provider |

现实意义：**"重试"这个动作至少有 8 种不同语义**，混成一个 `except: retry` 就是线上事故来源。

### 7.3 Auxiliary 旁路模型

压缩、视觉、标题、`session_search`、curator、embedding 都走 `auxiliary_client`，每个任务能在 `config.yaml` 的 `auxiliary:` 下独立钉 provider / model / base_url / reasoning_effort。

两条纪律：
- 辅助调用的 `pre/post_api_request` 事件**不得**触发（只有主循环发）——否则监控指标被污染。
- 压缩用的摘要模型**上下文必须 ≥ 主模型**，否则中段一次性塞不进去，压缩会静默丢内容（源码注释点名这是"压缩质量下降最常见的原因"）。

---

## 8. 可靠性层

| 机制 | 实现 | 解决的问题 |
|---|---|---|
| Deadline 层 | `deadline.py`：`run_bounded_sync/async`、`clamp_timeout`、`kill_process_tree` | 一个统一的有界执行原语，避免每个工具各写一套超时 |
| 中断控制 | `interrupt_control.py`：`/stop`、`/steer`、重定向 | API 调用在后台线程跑，主线程等"响应/中断/超时"三者之一 |
| 预算 | `iteration_budget.py`：线程安全 consume/refund | 父子 agent 各自记账，父的额度不被子吃光 |
| 恢复阶梯 | `turn_recovery.py` + `turn_recovery_autorecover.py` | 401 刷凭据、Unicode 编码错、图片过大、格式错……逐类恢复 |
| 重复检测 | `repetition_guard.py` | 模型陷入复读时打断 |
| 脱敏 | `redact.py`，错误串强制 `force=True` | 错误信息回写 DB 前必须脱敏 |

中断的关键取舍：**被放弃的响应直接丢弃，绝不把半截响应塞进历史**。否则下次请求会带着一个"幽灵助手轮次"，污染角色交替。

---

## 9. 状态层

```text
~/.hermes/
├── config.yaml          # 设置（行为）
├── .env                 # 只放密钥
├── state.db             # SQLite：sessions / messages / FTS5 / usage
├── SOUL.md              # 身份
├── MEMORY.md USER.md    # 记忆快照来源
├── skills/              # 技能（含 .archive/ 与 .usage.json）
└── profiles/<name>/     # 命名 profile：完全隔离的上述一切
```

三条纪律：

1. **单一写者**：一个 profile 的 gateway 是唯一规范写者；CLI / TUI / Desktop / ACP / cron 都是通过 `/api/ws` 提交输入的客户端。进程外 worker 走 `RuntimeSessionStore` RPC，**不自己开库写 transcript**。
2. **append-only**：历史只追加。序列坏了用 `repair_message_sequence` 修，而不是重写。
3. **压缩用世代标记**：in-place 压缩把旧行标 `active=0`、新行标 `active=1`。同一条消息在两代里同时出现是**正常现象**，不要当重复数据删。

`session_search` 直接在 FTS5 上做四种查询形状——discovery（query）/ scroll（session_id + around）/ read（整段或头尾）/ browse（无参）。**全程不调 LLM**，返回真实 DB 消息。

---

## 10. 抽象成通用 Agent Model

剥掉实现细节，可以提炼出 9 个核心对象和 12 条不变式。换个语言、砍掉 90% 功能，这套东西依然成立。

### 10.1 九个对象

| 对象 | 职责 | 生命周期 | 持久化 |
|---|---|---|---|
| **Agent** | 无状态执行器 + 配置（provider/预算/回调） | 进程级，可跨 turn 复用 | 否 |
| **Turn** | 一次用户输入的完整生命周期 | 一次调用 | 结果落库 |
| **Message** | OpenAI 形状 + `reasoning` + 元数据 | 追加式 | 是 |
| **Tool** | schema + handler + toolset + check_fn | 进程级注册 | 否 |
| **ContextEngine** | 何时压、怎么压、用量跟踪 | 会话级 | 摘要落库 |
| **MemoryProvider** | initialize → prefetch → sync_turn → shutdown | 会话级 | 外部存储 |
| **Skill** | SKILL.md + 支撑文件 + 用量状态 | 跨会话 | 是（文件） |
| **Route** | provider / model / api_mode / 凭据 | 可热切换 | 配置 |
| **Delegation** | 子任务契约（请求 / 句柄 / 结果） | 单次 | 子会话独立 |

### 10.2 十二条不变式

1. **系统提示词字节稳定**；唯一合法变更是压缩。
2. **中途注入只走 user / tool 通道**，不进 system prompt。
3. **角色严格交替**；只有 `tool` 可以连续。
4. **历史 append-only**；坏了修序列，不改写内容。
5. **压缩不切断 `tool_call ↔ tool_result` 配对**。
6. **工具结果必须结构化**（JSON 字符串或明确的多模态信封）。
7. **token 记账先信 provider usage**，估算仅作 fallback。
8. **记忆快照会话开始冻结**；会话中写入只落盘。
9. **子代理上下文隔离、预算独立、父只见摘要**。
10. **能力默认放边缘**（skill / plugin / MCP），核心工具门槛极高。
11. **永不删除，只归档**；危险动作必须审批。
12. **密钥读取走 scope，失败关闭**（fail-closed），绝不退化成 `os.getenv` 兜底。

### 10.3 反模式清单（源码里被明确拒绝的）

- 给指令类工具加 `offset/limit` 分页
- 为普通配置新建环境变量（行为进 `config.yaml`，`.env` 只放密钥）
- 没有真实消费者的 hook
- "修复"把原本要保护的功能干掉的补丁
- 默认开启的外部遥测
- 插件去改核心文件

---

## 11. 复刻路线：从 300 行到这套系统

| 阶段 | 你需要的 | Hermes 对应层 |
|---|---|---|
| **M0 最小内核** | `messages[]` + 一个 while 循环 + 3 个工具（读 / 写 / 跑） | `conversation_loop` 的 while |
| **M1 可用** | system prompt 组装 + 工具 registry + 错误重试 | `prompt_builder` + `registry` + `error_classifier`（先做 5 类） |
| **M2 长会话** | 上下文估算 + 剪枝 + 摘要压缩 | `context_compressor` Phase 1-4 |
| **M3 稳定** | 预算 + 中断 + deadline + 落盘 | `iteration_budget` + `deadline` + `state.db` |
| **M4 会成长** | 记忆文件 + 技能库 + 回合后复盘 | `memory_tool` + `skills` + `background_review` |
| **M5 规模化** | 多 provider 降级 + 子代理 + 缓存断点 | `FailoverReason` + `delegate_task` + `prompt_caching` |

**建议的砍法**：M0–M2 之间不要碰多 provider 和插件系统；先把"一轮对话稳定跑到 50 次工具调用还不炸"做出来，其余都是此之上的外设。

---

## 12. 这套模型的代价

必须说清楚它换来了什么、失去了什么：

| 代价 | 具体表现 |
|---|---|
| **复杂度** | `agent/` 319 个文件 / 6.5 MB Python；新人按文档阅读顺序要走 10 篇才能上手 |
| **缓存不变式反噬产品** | 改技能 / 改设置默认下个会话才生效，用户会觉得"没反应" |
| **同步循环 + 线程池** | 不是 asyncio，并发靠 `ThreadPoolExecutor` + 分段规划器；高并发不是设计目标 |
| **压缩有损** | 摘要模型窗口小于主模型会静默丢中段；源码用"警告 + 兜底"缓解，不能根治 |
| **provider 税** | 30+ 种失败分类、大量 `*_adapter.py`，都是现实世界不整齐的代价 |
| **学习闭环可能噪声化** | 复盘 prompt 强制"Be ACTIVE"，长期有技能库膨胀风险，靠 curator 归档对冲 |

**适用边界**：适合"**单用户 / 小团队、长会话、要长期记忆、要操作真实机器**"的个人 agent；不适合"高并发多租户 API 服务"或"毫秒级延迟"场景。

---

## 13. 源码索引

| 想读什么 | 去哪 |
|---|---|
| 主循环 | `agent/conversation_loop.py::_run_conversation_turn` |
| 回合各阶段 | `agent/turn_*.py`（13 个文件，按阶段名找） |
| 系统提示词 | `agent/system_prompt.py` + `agent/prompt_builder.py` |
| 压缩算法 | `agent/context_compressor.py`（330 KB，看 `compress()` 与 4 个 Phase mixin） |
| 工具注册 | `tools/registry.py` + `model_tools.py` + `toolsets.py` |
| 并发调度 | `agent/tool_dispatch_helpers.py::_plan_tool_batch_segments` |
| 循环护栏 | `agent/tool_guardrails.py` |
| 失败分类 | `agent/error_classifier.py::FailoverReason` |
| 复盘 fork | `agent/background_review.py`（含三段 review prompt 原文） |
| 技能生命周期 | `agent/curator.py` + `tools/skill_usage.py` |
| 子代理 | `tools/delegate_tool.py` + `agent/subagent_lifecycle.py` |
| 会话存储 | `hermes_state*.py` |
| 开发文档 | `website/docs/developer-guide/architecture.md` 起 10 篇 |

---

*本文基于 `8f5693af4182` 的公开源码分析；标为"工程含义 / 现实意义 / 适用边界"的段落是分析结论，其余均有源码或仓库文档依据。*
