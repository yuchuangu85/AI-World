# AI 世界日报 · 索引

> 定期巡检主流 AI 站点，**技术优先**，落盘为本仓库的日报笔记。
> 运行手册（信息源清单 / 优先级 / 流程）：[[News/RUNBOOK|RUNBOOK]]｜模板：[[News/_template|_template]]

## 最新一期

**[[News/2026-11/2026-11-02|2026-11-02 第 2 期]]** — 13 条
- 头条：Claude 文本水印扩到 4 个旧模型（11-03 生效，改采样参数即失效）
- 成本变化：DeepSeek Search/Fetch 11-01 起按 $5/千次计费
- 分发变化：Qwen 接入 Apple Intelligence（11-04 开发者 beta）
- 明天硬节点：11-03 水印生效 · 11-04 Kimi K3.1 权重截止

## 索引

| 日期 | 期号 | 条目 | 头条 | 备注 |
| --- | --- | --- | --- | --- |
| [2026-11-02](2026-11/2026-11-02.md) | #2 | 13 | Claude 水印扩大；DeepSeek Search 计费；Qwen×Apple | 补报 Mistral Large 4（10-29 遗漏） |
| [2026-11-01](2026-11/2026-11-01.md) | #1 | 25 | DeepSeek Super-agent 架构原型开源；Claude API 2026-10-21 | 基线期，回补 10-20~10-31 |

## 目录结构

```
News/
├── README.md          # 本索引（每期在此加一行）
├── RUNBOOK.md         # 运行手册：信息源、优先级、流程、红线
├── _watchlist.md      # 待跟项（日历驱动，每期开头先看）
├── _template.md       # 日报模板
└── YYYY-MM/
    └── YYYY-MM-DD.md  # 每日一期
```

## 怎么用

- **看最新**：打开「最新一期」；**按时间翻**：看索引表。
- **改巡检范围/优先级**：编辑 [RUNBOOK.md](RUNBOOK.md)，下一期自动生效。
- **改频率**：本会话的定时提醒驱动（默认每天 09:00 Asia/Shanghai），用 `schedule_list` / `schedule_update` 调整。

## 信息源快速入口

| 线 | 主入口 |
| --- | --- |
| Claude / Anthropic | [anthropic.com/news](https://www.anthropic.com/news) · [Claude Code changelog](https://code.claude.com/docs/en/changelog.md) · [support 发布说明](https://support.claude.com/en/articles/12138966-release-notes) |
| OpenAI / Codex | [openai.com/news](https://openai.com/news/) · [Codex releases](https://github.com/openai/codex/releases) · [模型发布说明](https://help.openai.com/en/articles/9624314-model-release-notes) |
| DSH / DeepSeek | [deepseek-harness releases](https://github.com/deepseek-ai/deepseek-harness/releases) · 微信公众号检索 `site:mp.weixin.qq.com DSH` |
| GitHub | [trending daily](https://github.com/trending?since=daily) · [trending weekly](https://github.com/trending?since=weekly) |
| 其他主流 | [DeepMind/Gemini](https://blog.google/technology/ai/) · [Kimi](https://www.moonshot.cn/) · [Mistral](https://mistral.ai/news/) · [HF](https://huggingface.co/models) · [arXiv cs.CL](https://arxiv.org/list/cs.CL/recent) |
