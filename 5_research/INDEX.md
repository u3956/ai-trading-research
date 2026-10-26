# Study index

Every trading idea gets its own study folder. Each study follows the same steps:

1. **Plan before results.** The question, the exact rules, the controls and the bar a result must clear are written
   down (and, for the later studies, frozen by hash) before any outcome is computed.
2. **Controls that can say no.** A placebo line or day for every level claim, same-day look-alike stocks, each name's
   own drift removed, two halves of the data that must agree, standard errors clustered by session or week.
3. **An independent check.** A second build or a set of AI reviewers that did not write the code tries to break the
   result before it is read as real. Several verdicts below changed at this step.
4. **A dated verdict.** **keep** (survived the bar, or kept as a reference), **kill** (did not), **park** (a lean too
   weak or too rare to decide), **open** (still running).

Most ideas are killed. That is the point of the workbench: it is cheaper to kill an idea on 11 years of data than in
a live account. Killed ideas are listed with the reason they died; kept, parked and open ones with their verdict only.

The full worked example in this repo is [supply_chain_contagion](example_supply_chain_contagion/README.md).

| # | Study | Question | Verdict | Why it died |
|---|---|---|---|---|
| 1 | ema_4cross | Daily 4-EMA gate plus an intraday 9/21 EMA cross | open | |
| 2 | fresh_pullback | First VWAP pullback on names that just entered the screener's lists | kill | the move is spent by the time the pullback confirms (3 versions) |
| 3 | open_signal | Do the first minutes of the session predict the close? | kill | accuracy under 50% everywhere; no stock responds better than chance |
| 4 | qqq_lead | Does a QQQ bar lead single stocks' next 1/5/15-minute bar? | kill | real at 1 minute, but it is microstructure catch-up and nets negative by 15 minutes |
| 5 | adr_buckets | Forward excess return by ADR% bucket | keep | |
| 6 | level_reactivity | Which EMAs and VWAPs does each stock respect? | keep | |
| 7 | level_regime | Does the market regime decide which level holds? | keep | |
| 8 | early_low_entry | A limit buy at the early-session low | park | |
| 9 | regime_gate | A market-regime gate on the daily list | keep | |
| 10 | overnight_hold | Which stocks can carry a position overnight? | keep | |
| 11 | downday_reversal | A down-day washout and reclaim: does the rebound carry forward? | keep | |
| 12 | confirm_vs_anticipate | Base entries: wait for the confirming close, or buy inside the base? | keep | |
| 13 | down90_reversal | Stocks down 90%: is there a turnaround trade? | park | |
| 14 | weekly_ema_touch | Buy the touch of the weekly 9/21 EMA | kill | ~82% reclaim the line within 5 sessions, but a placebo line does the same; 0 of 2,250 stocks pass a false-discovery test |
| 15 | gap_up | Pre-market gap-ups: do they keep going after the open? | kill | measured from the open they trail a quiet-stock control |
| 16 | index_low_pullback | Buy the EMA pullback only while QQQ/SPY sits at its intraday low | kill | the lift is beta to the index's own rebound |
| 17 | breakdown_short | Short the undercut of a tight, declining EMA bundle | keep | |
| 18 | analyst_upside | Buy-rated stocks with large upside to the consensus target | keep | |
| 19 | run_trend_break | A weekly trend break after a steep run: does it predict the next weeks? | park | |
| 20 | squeeze_long | Buy the day after an underside rejection (the squeeze cell) | kill | no edge over the same-state population; the headline was one macro day |
| 21 | industry_rotation | Does today's industry up-share predict tomorrow? | keep | |
| 22 | ema150_band_regime | A close near the daily EMA150 | keep | |
| 23 | adr_exhaustion_short | Is "down one ADR from the open" a short? | keep | |
| 24 | open_flush_bounce | Do hard opening drops bounce? | keep | |
| 25 | theme_rs_divergence | Buy the theme name that is up while its group is down | kill | relative strength inside a group mean-reverts |
| 26 | green_lower_close | Two green candles that still close lower | kill | a coin flip the next day |
| 27 | contraction_at_cluster | Contraction on a tight cluster of moving averages | park | |
| 28 | counter_trend_poke_short | Short a counter-trend rally's failed poke above the prior high | park | |
| 29 | first_hour_high_fade | Short the morning high: can the fade day be picked before the open? | keep | |
| 30 | green_streak | N green candles in a row | kill | the ordinary short-term reversal, both ways |
| 31 | premarket_915 | Read at 09:15: will the open be up or down? | keep | |
| 32 | high_adr_short | Are stocks with 10%+ ADR good shorts? | kill | 0 of 6 frozen legs pass; in a weak tape the short loses |
| 33 | adr_run_short | Short an outsized intraday run | keep | |
| 34 | theme_leaders | Group "leaders" columns and price-only theme clusters | keep | |
| 35 | catalyst_dates | Scheduled catalyst dates: the run-in, holding through, buying the news | park | |
| 36 | forward_test_design | Which forward test fits a trade a paper account cannot simulate? ([protocol](../1_agents/forward_logs/PROTOCOL.md)) | open | |
| 37 | crashed_gap_basket | Buy 10%+ gap-ups in crashed names as a basket | kill | the lean was a baseline artifact: crashed names compared with their own later decay |
| 38 | catalyst_movers | Does reading the catalyst on the +10% movers list pick the runners? | keep | |
| 39 | hot_pack_movers | Does a hot peer pack (companies with similar businesses) pick the runner? | kill | a mover in a hot pack doubles ×1.02 as often as one in a cold pack |
| 40 | runner_profile | "Market aligned at the start of a runner" as a pick rule | kill | the wrong sign: aligned-market starts did worse over 12 months; a one-runner checklist is a season bet |
| 41 | movers_fundamentals | Do day-of fundamentals separate the next big runner from the duds? | keep | |
| 42 | earnings_movers | Do consistent earnings beaters run further? | kill | steadier, not bigger |
| 43 | revenue_acceleration | Does accelerating revenue growth beat steady reporters? | open | |
| 44 | [supply_chain_contagion](example_supply_chain_contagion/README.md) | Does a breadth collapse in one link of a supply chain predict underperformance in the rest over 10–20 sessions, and is a chain-wide washout a floor? | kill | about 0 once the matched control is built correctly; a control-day exclusion had faked −74 to −255 bp |
| 45 | bellwether_lead | Is there a bellwether whose moves lead its group? | kill | the fitted pick leads no better than an average candidate; its in-sample score equals the luck level (6 of 6) |
| 46 | first_mover | Does the first name to cross a line carry direction for its group? | kill | no calibrated cell of 118 clears the family-wise bar |
| 47 | movers_grid | Which day-of traits make a +10% mover likelier to reach 5x? | kill | every trait that lifts the 5x odds lifts the halvings with it (0 of 8,821 rows) |

## Layout of a study

```
<study>/
├── README.md     the plan (written first), then Results and a dated Verdict
├── *.py          build -> analyze -> review scripts; every script has --limit N for a smoke run
└── out/          logs and result tables; smoke runs only ever write to out/_smoke_limitN/
```

Shared pieces: [template/](template/README.md) (how every study starts, with excerpts of the starter script and
the shared bar loader), the data downloaders in [../2_pipelines/market_data/](../2_pipelines/market_data/README.md),
and [LESSONS.md](LESSONS.md), the rules every new study must apply. Study code is not published; the worked example
shows excerpts in [CODE.md](example_supply_chain_contagion/CODE.md).
