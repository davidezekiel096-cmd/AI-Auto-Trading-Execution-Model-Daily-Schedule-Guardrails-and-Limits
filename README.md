# AI Auto Trading Execution Model: Daily Schedule, Guardrails and Limits

Reference notes on how a rules-based ai auto trading system can be structured so that automation stays predictable, auditable and under the investor's control. The model described here follows the execution approach used by [Tactfolio](https://tactfolio.com/ai-trading), where an AI assistant helps build and test strategies but cannot start or trade them.

This document describes architecture and guardrails. It is not trading advice.

---

## Design Principles

| Principle | Implementation |
|---|---|
| Human start | Only the investor can start a strategy |
| Fixed rules | Running strategies follow saved rules exactly |
| Low frequency | Rules checked once per trading day, near the close |
| Own custody | Orders placed in the investor's own Alpaca account |
| Own credentials | Investor's own API keys |
| No fund custody | Tactfolio states it never holds funds |
| No trade cut | Tactfolio states it takes no cut of trades |
| Bounded AI | Assistant cannot trade, publish, change settings or hold unnamed assets |

---

## Roles

| Role | Can build | Can test | Can start | Can trade |
|---|---|---|---|---|
| Investor | Yes | Yes | Yes | Through started strategies |
| AI assistant (via MCP) | Yes | Yes | No | No |
| Scheduler | No | No | No | Executes started strategies only |

Assistants that connect through MCP include Claude, ChatGPT, Claude Code, Codex and Cursor. Sign-in is OAuth, with no API key needed for the assistant.

---

## Lifecycle States

```
DRAFT -> BACKTESTED -> PAPER -> LIVE -> STOPPED
   ^         |           |
   |_________|___________|   (revisions return to DRAFT; undo supported)
```

| State | Entered by | Money involved |
|---|---|---|
| DRAFT | Investor or assistant | None |
| BACKTESTED | Platform | None (hypothetical) |
| PAPER | Investor start | Simulated, on real prices |
| LIVE | Investor start | Real, in own Alpaca account |
| STOPPED | Investor | None going forward |

---

## Daily Execution Loop (Conceptual)

```
schedule daily at near_close(trading_day):
    for strategy in strategies where state in {PAPER, LIVE}:
        assert strategy.started_by == "investor"
        prices = daily_prices(strategy.universe)
        target = evaluate(strategy.rule_tree, prices)

        assert all(asset in strategy.named_assets for asset in target)
        assert all(weight >= 0 for weight in target.values())   # long only

        if strategy.state == PAPER:
            simulate_rebalance(strategy.paper_book, target, prices)
            queue_daily_email(strategy)                         # Plus plan
        elif strategy.state == LIVE:
            alpaca(strategy.owner.keys).rebalance_to(target)

        log(strategy.id, trading_day, target)
```

---

## Guardrails

| Guardrail | Purpose |
|---|---|
| Investor-only start | Prevents AI from committing money |
| Named-asset check | Prevents holding assets outside the rules |
| Long-only check | Enforces no short positions |
| No margin | Avoids borrowed exposure |
| Daily cadence | Removes intraday noise and high-speed risk |
| Own API keys | Keeps account control with the investor |
| Undoable revisions | Makes experimentation reversible |

---

## Asset Scope

| Allowed | Blocked |
|---|---|
| US-listed stocks | Options |
| US-listed ETFs | Forex |
| Leveraged ETFs | CFDs |
| Inverse ETFs | Crypto |
| Daily prices | Short selling |
| | Margin |
| | Intraday / high-speed |

---

## Paper Mode

Paper mode lets the same strategy run without real money, which is where the platform functions as ai paper trading software.

| Paper feature | Detail |
|---|---|
| Simulated money | On real prices |
| Daily email | Plus plan |
| Alpaca paper accounts | Supported |
| Max strategies on paper (Plus) | 25 |

Recommended pattern:

```
strategy.state = PAPER
run_for(period=meaningful_duration)
if behavior_matches_expectations(strategy) and investor_approves():
    strategy.state = LIVE         # investor action only
```

---

## Pre-Live Validation

Before a strategy can sensibly reach LIVE, the backtest and variant process should already have run.

| Check | Available on Tactfolio |
|---|---|
| Decades of real prices | Yes |
| Split and dividend adjustment | Yes |
| Trading costs | Yes |
| Drawdowns | Yes |
| Benchmark | Yes |
| Next-day-open line | Yes |
| "Reality checks" table | Yes |
| Up to 12 variants | Yes |
| Winner checked on unseen years | Yes |

```
winner = best_of(variants, period=older_years)
holdout = backtest(winner, period=unseen_years)
flag_if(holdout.score << winner.score, "possible overfit")
```

---

## Observability

| Signal | Source |
|---|---|
| Daily position changes | Execution log |
| Paper results | Daily email (Plus) |
| Live positions | Investor's Alpaca account |
| Published strategy track record | Date rules were fixed + out-of-sample line |

---

## Failure Modes and Handling

| Situation | Expected behavior |
|---|---|
| Market closed | No evaluation; next trading day resumes schedule |
| Rule references unnamed asset | Blocked by named-asset check |
| Target includes negative weight | Blocked by long-only check |
| Assistant attempts to trade | Not permitted; assistant has no trade capability |
| Investor stops strategy | No further orders from that strategy |
| Revision made in error | Revision can be undone |

The intent is that automation never does anything the investor did not explicitly start, and never holds anything the rules did not explicitly name.

---

## Plans

| Plan | Price | Execution |
|---|---|---|
| Free | $0 | Build and backtest only |
| Plus | From $12/month | Paper and live; 2,000 AI assistant backtests a day |
| Pro | See pricing page | Higher tier |
| Trial | 7 days | Plus and Pro |

---

## Comparison Notes for Any AI Trading App

When reviewing any ai trading tool for automation, map it against this model:

- Who can start a strategy?
- Where are funds held?
- Whose API keys are used?
- What cadence do rules run on?
- Are asset and position limits enforced?
- Is there a paper mode with identical rules?
- Is every change logged?

---

## Disclaimer

Tactfolio describes itself as a research and automation tool, not a broker or adviser. Its FAQ makes no promise that AI trading will be profitable. Backtests are hypothetical. Investing involves risk, including loss of principal. This document is not financial advice. Tactfolio cites the CFTC and SEC investor alerts on AI trading claims.

Reference: [Tactfolio's AI trading overview](https://tactfolio.com/ai-trading).
