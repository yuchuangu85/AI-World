# AI 世界日报 · 运行手册（RUNBOOK）

> 本文件是自动巡检的**唯一契约**。定期任务触发时，按本文件执行即可复现同样质量的日报。
> 修改巡检范围、优先级、节奏，只需改本文件（或 Schedule 提醒）。

## 1. 定位

定期巡检主流 AI 官方站点与社区，抓取**最新**动态，技术优先，落盘为本仓库的日报笔记。
读者画像：AI 应用开发者 / Agent 工程从业者。

## 2. 优先级

| 级别 | 内容 | 处理 |
| --- | --- | --- |
| **P0 技术** | 模型/能力发布、API·SDK·协议变更、Agent harness 架构、release notes、benchmark、技术报告 | 必收，占条目主体 |
| **P1 工程与生态** | 官方工程博客、工具链、Skills/MCP、开源项目爆发式增长、论文 | 应收 |
| **P2 商业** | 融资/IPO/诉讼/人事/合作 | 最多 1 条，且标注「商业」 |

> 判断标准：一条消息若**不改变开发者的做法或技术选型**，就不是 P0。

## 3. 信息源清单

每次巡检至少覆盖全部 P0 源；P1/P2 源轮换抽查。

### 3.1 Claude / Anthropic

| 入口 | 说明 |
| --- | --- |
| https://www.anthropic.com/news | 官方新闻 |
| https://www.anthropic.com/research | 研究/技术报告 |
| https://support.claude.com/en/articles/12138966-release-notes | 产品发布说明（Claude Apps、Cowork 等） |
| **https://code.claude.com/docs/en/changelog.md** | Claude Code changelog（纯 markdown，最易抓，含版本+日期）✅ |
| https://platform.claude.com/docs/en/release-notes/overview | API release notes（部分地区 307 到 region 页，抓不到就往上一层用搜索） |
| https://github.com/anthropics/claude-code/releases | Claude Code 版本 |
| https://github.com/anthropics | 新仓库（Agent SDK、skills 等） |

### 3.2 OpenAI / Codex

| 入口 | 说明 |
| --- | --- |
| https://openai.com/news/ · https://openai.com/index/ | 官方新闻/博客/研究 |
| https://developers.openai.com/codex/changelog | Codex changelog |
| https://platform.openai.com/docs/changelog | API changelog（需登录时用搜索代理） |
| https://github.com/openai/codex/releases | Codex CLI/SDK 版本 |
| https://github.com/openai | 新仓库（Agents SDK 等） |

### 3.3 DSH（DeepSeek Harness）与 DeepSeek

| 入口 | 说明 |
| --- | --- |
| `site:mp.weixin.qq.com DSH` / `"DeepSeek Harness"` | 微信公众号文章（重点） |
| `site:mp.weixin.qq.com DeepSeek` | DeepSeek 官方公众号技术发布 |
| DSH 官网 / GitHub 仓库 / Release | 版本与架构更新 |
| https://github.com/deepseek-ai | 开源模型与仓库 |
| https://api-docs.deepseek.com | API 文档/变更 |

> 微信文章常抓不到正文：用搜索结果标题+摘要，并明确标注「仅标题/摘要」。

### 3.4 GitHub 开源生态

| 入口 | 说明 |
| --- | --- |
| https://github.com/trending?since=daily | 日榜（AI 只挑技术硬货） |
| https://github.com/trending?since=weekly | 周榜 |
| GitHub 搜索 | `agent framework`、`agent harness`、`MCP server`、`LLM inference`、`skills` |

### 3.5 其他主流 AI 站点

Google DeepMind / Gemini API changelog、Meta AI、Qwen（阿里）、Moonshot Kimi、xAI Grok、Mistral、Hugging Face（模型榜）、Cursor、arXiv（cs.AI / cs.CL 高热论文）。

### 3.6 抓取避坑（2026-11-01 实测）

| 现象 | 正确做法 |
| --- | --- |
| `curl` 取 openai.com / developers.openai.com 返回 **403**（Cloudflare） | 改用 `web_fetch` 工具（同为 200）；changelog 类改抓 GitHub Releases / help.openai.com |
| `platform.openai.com/docs/changelog` 需登录 | 改抓 `openai/openai-python` Releases（API 面变更最全） |
| `platform.claude.com/docs/*` 本网络 **307 到 region 页 / 超时** | 用 `code.claude.com/docs/en/changelog.md`、`support.claude.com`、GitHub Releases；API 版本细节用社区实测交叉验证并标注 |
| `docs.claude.com/en/release-notes/api` | 301 → `platform.claude.com/docs/en/release-notes/overview`（同样受地区限制） |
| `mp.weixin.qq.com` 返回「环境异常」验证码页 | 只取搜索结果标题/摘要，正文与时间标「未验证」 |
| 需要 star 数 / 建仓时间 | GitHub REST API（`api.github.com/repos/...`）比 HTML 稳 |
| 链接校验 | `curl -s -o /dev/null -w '%{http_code}' -L -A "Mozilla/5.0"`，200 才算过；GitHub/arXiv/blog 一般 200 |

## 4. 执行流程

0. **先看待跟项**：读 `News/_watchlist.md`，逐条核对是否落地；落地的移到当期「今日必看」，并在表里更新状态。
1. **并行分派**（3 路）
   - 路 A：Claude/Anthropic（teammate `claude-desk`）
   - 路 B：OpenAI/Codex（teammate `codex-desk`）
   - 路 C：GitHub + DSH/公众号 + 其他厂商（teammate `oss-desk`）
   - 队友不可用时，退化为 `subagent` 并行的同题任务；再不可用则由 Lead 自行分批 `web_search`。
2. **汇总去重**：同一事件多源合并，保留最权威/最早来源。
3. **校验**：URL 可访问（抽查 `web_fetch`）、标题与日期真实、非旧闻。
4. **落盘**：`News/YYYY-MM/YYYY-MM-DD.md`（模板 `_template.md`）。
5. **更新索引**：在 `News/README.md` 索引表顶部插入当日一行。
6. **汇报**：3–5 条一句话摘要 + 当日文件链接；无重大更新也要说明「今日无 P0」。

> **防重复出刊**：若本次触发与当日已出那一期间隔 < 2 小时（常见于定时任务的「补投」），**只做增量检查**（Claude Code changelog 最新版本号、各仓库 latest release、news 首页首条），无新增即回复「无增量」并跳过写文件，绝不生成第二条同日日报。

## 5. 产出规范

- 文件名：`News/YYYY-MM/YYYY-MM-DD.md`，月目录 `YYYY-MM`（如 `2026-11`）。
- 条目数量：8–15 条；确实无事则 ≤5 条并说明原因。
- 每条结构：中文短标题 / 原文标题 / 来源链接 / 日期 / 技术要点（2–3 句）/ 分类。
- 语言：中文，专有名词保留英文（Claude Code、Codex、MCP、Agent SDK…）。
- 去重：同一事件连续多日不重复收录；有新进展时标注「**进展**」。

## 6. 质量红线

- **不编造**：只写实际抓取或两次搜索交叉验证的内容；不确定标注「未验证」。
- **日期**为发布/更新时间，不是抓取时间。
- 不收纯营销、无技术增量的内容。
- 商业类 ≤ 1 条。
- 链接必须真实可点，禁止拼凑 URL。

## 7. 节奏与变更

- 默认：**每天 09:00（Asia/Shanghai）** 巡检一次，由本会话的 Schedule 提醒驱动。
- 并行分工用 durable teammates：`claude-desk`（Claude 线）、`codex-desk`（OpenAI 线）、`oss-desk`（GitHub/DSH/其他），Lead 负责汇总去重、校验与落盘。
- 改频率：`schedule_list` → `schedule_update`（或删除后重建）。
- 改信息源/优先级：直接编辑本文件，下一期自动生效。
