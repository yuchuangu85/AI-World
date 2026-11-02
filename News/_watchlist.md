# 待跟项（滚动更新）

> 每期巡检**开头先看这里**，逐条核对是否落地；落地了就移到当期的「今日必看」并在此打勾或删除。
> 格式：`日期 | 事项 | 怎么核 | 状态`

| 日期 | 事项 | 怎么核 | 状态 |
| --- | --- | --- | --- |
| **2026-11-03** | Claude 文本水印在 Mythos 5 / Opus 4.7 / Opus 4.6 / Sonnet 4.6 生效 | 抓 support.claude.com `16266773-how-claude-marks-ai-generated-content` 模型表是否补勾；社区有无实测 | 待生效 |
| **2026-11-04** | **Kimi K3.1 权重发布最后期限**（官方 blog 措辞 "The full model weights will be released by Nov 4"） | kimi.ai/blog/kimi-k3-1-pro + HF/魔搭是否出现权重 + 许可证 | 倒计时 |
| **2026-11-04** | Qwen × Apple Intelligence 开发者 beta 开放 | 新华社/China Daily 后续 + Apple developer release notes 是否点名 Qwen | 待观察 |
| 11 月中（估） | Gemini 4 Carbon API 开放（按 Argon 13 天先例推算） | blog.google + ai.google.dev changelog（本机常超时，备第三方观测） | 未定 |
| **2026-11-30** | Opus 4.8 fast mode 移除，**不回落**标准模式，需迁 Opus 5.5 | code.claude.com changelog 是否出现 2.1.3xx 相关下线提示 | 倒计时 |
| 持续 | Claude API `2026-10-21` 迁移（`temperature`/`top_p`/`top_k` 传了即 400） | 项目侧自查；官方 migration doc | 用户侧待办 |
| 持续 | DeepSeek Search/Fetch 计费后的成本与限流表现（$5/千次；QPS 100/25 按账户；同域名 Fetch 单独约束） | 实测 429 频率；官方额度页 | 观察 2 周 |
| 每周抽查 | 上期 4 个热门 Skill 项目（hallmark / security-audit-skill / SkillSpector / sandbox-runtime）零工程增量 | GitHub `pushed_at` 是否变化；连续 2 周无 push 则降级 | 观察中 |
| 每周抽查 | DSH `dsh-superagent-prototype` 是否从「星数热度」转为「有提交」 | GitHub `pushed_at`（当前仍停在 10-31 建仓日） | 观察中 |
| 每周抽查 | Gemini / Meta / xAI Grok 5 / Qwen 是否有新模型（本期均无） | 官方 blog + changelog | 观察中 |
