# Subagents — Multi-Agent Trading Desk

This directory holds the Claude Code subagent definitions used by the trading desk workflow described in [`../../AGENTS.md`](../../AGENTS.md). The orchestrator (the main Claude Code agent in this repo) invokes each role via the `Agent` tool with `subagent_type` set to the file name (without `.md`).

See [`AGENTS.md`](../../AGENTS.md) for the full architecture, workflow, and configuration knobs.

## Roster

| File | Role | Stage |
|---|---|---|
| `fundamentals-analyst.md` | Company financials & valuation | Analyst (parallel) |
| `sentiment-analyst.md` | Retail / social sentiment | Analyst (parallel) |
| `news-analyst.md` | Company, sector, and macro news | Analyst (parallel) |
| `technical-analyst.md` | Price action, MACD/RSI, key levels | Analyst (parallel) |
| `bull-researcher.md` | Long thesis in research debate | Researcher (looped) |
| `bear-researcher.md` | Short / avoid thesis in research debate | Researcher (looped) |
| `research-manager.md` | Adjudicates the debate, issues stance | Researcher (synthesis) |
| `trader.md` | Writes the trade ticket | Trading |
| `risk-aggressive.md` | Press-the-trade voice in risk debate | Risk (looped) |
| `risk-conservative.md` | Capital-preservation voice in risk debate | Risk (looped) |
| `risk-neutral.md` | Balanced/centrist voice in risk debate | Risk (looped) |
| `portfolio-manager.md` | Final approve / reject / modify decision | Decision |
