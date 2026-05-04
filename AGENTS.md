# AGENTS.md — Multi-Agent Trading Desk

This repository runs a **multi-agent trading workflow** inspired by [TauricResearch/TradingAgents](https://github.com/TauricResearch/TradingAgents). It mirrors the structure of a real trading firm: specialist analysts gather evidence, researchers debate the thesis, a trader writes a proposal, a risk team stress-tests it, and a portfolio manager makes the final call.

Each role lives as a Claude Code subagent under `.claude/agents/`. The main agent in this repo acts as the **orchestrator** that wires them together for a given `(ticker, date)` input.

---

## Architecture

```
                       INPUT: ticker + analysis date
                                   │
                ┌──────────────────┼──────────────────┐
                ▼                  ▼                  ▼              ▼
         fundamentals-       sentiment-          news-         technical-
            analyst            analyst          analyst          analyst
                └──────────────────┴──────────┬──────┴──────────────┘
                                              ▼
                                     ANALYST REPORTS
                                              │
                       ┌──────────────────────┼──────────────────────┐
                       ▼                      ▼                      ▼
                 bull-researcher  ⇄  research-manager  ⇄  bear-researcher
                                  (max_debate_rounds)
                                              │
                                              ▼
                                    RESEARCH SYNTHESIS
                                              │
                                              ▼
                                           trader        ── BUY / SELL / HOLD proposal
                                              │
                       ┌──────────────────────┼──────────────────────┐
                       ▼                      ▼                      ▼
                risk-aggressive  ⇄  risk-neutral  ⇄  risk-conservative
                                  (max_risk_discuss_rounds)
                                              │
                                              ▼
                                    portfolio-manager   ── final decision + sizing
                                              │
                                              ▼
                                          OUTPUT
```

---

## Agent roster

| Stage | Subagent | Inputs | Output | Model tier |
|---|---|---|---|---|
| Analyst | `fundamentals-analyst` | ticker, date | Financials report | quick |
| Analyst | `sentiment-analyst` | ticker, date | Sentiment report | quick |
| Analyst | `news-analyst` | ticker, date | News & macro report | quick |
| Analyst | `technical-analyst` | ticker, date | Technicals report (MACD, RSI, etc.) | quick |
| Research | `bull-researcher` | All analyst reports + prior debate turns | Long thesis turn | deep |
| Research | `bear-researcher` | All analyst reports + prior debate turns | Short / risk thesis turn | deep |
| Research | `research-manager` | Full debate transcript | Synthesis + recommended stance | deep |
| Trading | `trader` | Research synthesis | BUY/SELL/HOLD proposal with size & rationale | deep |
| Risk | `risk-aggressive` | Trader proposal + prior risk turns | Aggressive critique | quick |
| Risk | `risk-conservative` | Trader proposal + prior risk turns | Capital-preservation critique | quick |
| Risk | `risk-neutral` | Trader proposal + prior risk turns | Balanced critique | quick |
| Decision | `portfolio-manager` | Trader proposal + risk transcript | Final approve/reject + sizing | deep |

Model tiers map to the TradingAgents `quick_think_llm` / `deep_think_llm` knobs:

- **quick** — fast, cheap (e.g. `claude-haiku-4-5-20251001`). Used for narrow-scope analyst and risk turns.
- **deep** — high-reasoning (e.g. `claude-opus-4-7` or `claude-sonnet-4-6`). Used for debate, synthesis, and final decisions.

---

## Workflow

A single run of the desk for one `(ticker, date)`:

1. **Analyst fan-out (parallel).** The orchestrator launches all four analyst subagents in a single message with multiple tool calls. Each writes its report against the same `(ticker, date)`.
2. **Researcher debate (sequential, looped).** For `r in 1..max_debate_rounds`:
   1. `bull-researcher` writes turn `r`, given all analyst reports and the prior transcript.
   2. `bear-researcher` writes turn `r`, given the same inputs plus the bull's turn `r`.
3. **Research synthesis.** `research-manager` reads the full transcript and emits a synthesis + recommended stance.
4. **Trader proposal.** `trader` reads the synthesis and writes a concrete BUY / SELL / HOLD proposal with target size and rationale.
5. **Risk debate (sequential, looped).** For `r in 1..max_risk_discuss_rounds`:
   1. `risk-aggressive`, then `risk-conservative`, then `risk-neutral` each write turn `r` against the proposal and prior risk transcript.
6. **Final decision.** `portfolio-manager` reads the proposal + risk transcript and outputs the final decision: `APPROVE` / `REJECT` / `MODIFY`, with final sizing and a one-paragraph justification.

The orchestrator is responsible for passing each subagent the inputs listed in the roster table — never assume a subagent has memory of earlier turns; always pass the relevant transcript explicitly in the prompt.

---

## Running multiple agents

This repo is designed to be driven from a Claude Code session. The orchestrator (the main agent) invokes each role via the `Agent` tool with `subagent_type` set to the role name.

**Parallel fan-out** (analyst stage, risk stage within a single round):

> Send a single message containing one `Agent` tool call per subagent. Claude Code runs them concurrently, and you get all results before the next stage.

**Sequential chaining** (debate rounds, trader → risk → PM):

> Wait for the previous subagent's result, then call the next with the accumulated transcript in the prompt.

**Prompting subagents.** Each subagent is briefed cold — it does not see the orchestrator's conversation. Always include in the prompt:

- the ticker and analysis date,
- any upstream reports / debate transcript it must read,
- the exact output schema you expect back.

A canonical orchestrator request looks like:

> *"Run a full TradingAgents-style analysis on `NVDA` for `2026-05-04`. Use `max_debate_rounds=2` and `max_risk_discuss_rounds=1`. Return the portfolio manager's final decision."*

The orchestrator then executes the workflow above and returns only the final decision, with the transcript available on request.

---

## Configuration

Configuration is passed in the orchestrator's prompt (no config file is required). The knobs mirror TradingAgents:

| Knob | Default | Effect |
|---|---|---|
| `max_debate_rounds` | `2` | Bull/bear turns per side in the research debate |
| `max_risk_discuss_rounds` | `1` | Full A/C/N cycles in the risk debate |
| `online_tools` | `true` | If `false`, analysts must rely on cached/Read-only context instead of `WebFetch` / `WebSearch` |
| `deep_think_llm` | `claude-opus-4-7` | Model used by debate / trader / PM subagents |
| `quick_think_llm` | `claude-haiku-4-5-20251001` | Model used by analyst / risk subagents |

**Per-role model overrides.** When invoking a subagent via the `Agent` tool, pass `model` to override the default for that call (e.g. promote a risk turn to `deep` for a high-stakes ticker).

**Data-source environment variables** (only needed if you wire analysts to live APIs):

- `ALPHA_VANTAGE_API_KEY` — fundamentals & technicals
- `FINNHUB_API_KEY` — fundamentals & news
- `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` — LLM provider keys (handled by the host)

The default subagent definitions in this repo use `WebFetch` / `WebSearch` and do not require any of the above; swap to API-backed tools when running at scale.

---

## Extending

- **Add a new analyst.** Drop a new `.claude/agents/<name>-analyst.md` file with the same frontmatter shape, then add it to the analyst fan-out step in the workflow above.
- **Swap the debate format** (e.g. add a third researcher). Add the new role under `.claude/agents/`, update the **Researcher debate** loop in the workflow, and update the inputs in the roster table.
- **Plug in new data sources.** Replace `WebFetch` / `WebSearch` in an analyst's `tools:` list with a custom MCP tool, and update the analyst's prompt body to call it.

---

## Disclaimer

This is a research scaffold. It is **not** investment advice, it does not place real orders, and it must not be used to drive real capital without human review.
