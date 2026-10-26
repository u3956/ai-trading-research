# Supply-chain contagion: do breadth shocks propagate along economic chains?

## Summary

Economically linked firms show return lead-lag at the monthly horizon (Cohen & Frazzini 2008; Menzly & Ozbas 2010).
This study tests whether the same propagation appears in daily breadth over 10–20 sessions, across 12 economic
chains (linked groups of industries, such as the AI compute chain from chip makers to data-center hosting and power)
and 11 years of US equities. It asks two questions:

1. When one link's breadth collapses while the rest of the chain and the market still hold, does the rest
   underperform?
2. When every link has collapsed in a weak market, is that a floor?

The plan was frozen by hash before any return was computed. The first full run showed −74 bp at 10 sessions
(z −1.83) and −208 bp at 20 (z −3.09). An independent rebuild and audit traced the effect to a look-ahead in how
the control days were selected. Corrected, the effect is −6 bp (z −0.15), and both hypotheses were killed.

The study's code (working name `chain_canary`) is not published; [CODE.md](CODE.md) shows the parts that enforce
this plan as short excerpts.

**Started:** 2026-10-23
**Symbols:** the toggle workbook's book on the 11-year layer (the theme_leaders study's panel:
2,896 names, daily bars 2015-06-04 → 2026-08-24 + 155 ADR/MLP names from the daily store);
live layer `out/panel_live.npz` from the screener's nightly daily store (2,925 names, 2021-08-09 →
2026-10-22), descriptive only.
**Data used:** the two panels, the SPY and QQQ benchmark files (11-year and live), the workbook's
group labels (`groups.assign()`: themes.csv → HAND → FAMILY). All read-only.

Links: [PROFILE.md](../../6_working_with_ai/PROFILE.md) · [LESSONS.md](../LESSONS.md)

## Why this study

The question (2026-10-23): can momentum turns be read from the news, the macro tape and the
stocks together? The example was the AI-stock selloff of October 2026, with CoreWeave as the bellwether.
A read the same day (scratch, the screener untouched) found that two similar-looking
moments ended in opposite ways:

| | Sep 15, 2026 | Oct 8, 2026 |
|---|---|---|
| SPY / QQQ vs their 21 and 50 EMAs | under both | over both |
| CoreWeave | under both, −27% off its 60-day high | under both, −24% |
| Neoclouds / AI semis / semicap: share of names over their 21 EMA | 8% / 7% / 0% | **0%** / 43% / 45% |
| Next 10 sessions, AI semis / semicap / neoclouds (median name) | **+16% / +20% / +8%** | **−18% / −26% / −29%** |

The shares come from that scratch read. The study's own replay, with its eligibility rules, gives
9% / 8% / 0% on Sep 15 and 0% / 38% / 45% on Oct 8 (`out/replay.log`).

The bearish news, the Fed hike and a 2007-high 10-year yield were all present on Sep 15, and that
moment ripped. Oct 8 was different: **the most leveraged link of the chain (neoclouds) had fully
broken while the rest of the chain and the index still held.** This study asks whether that
pattern, a breadth shock in one link spreading to the rest, carries information in general, across 12 economic
chains and 11 years.

The example pair generated the hypothesis, so it is **not evidence**. It sits in the live layer,
outside the 11-year panel's dates, and the live layer is never voted.

The question has a second half: is a washout a floor? Sep 15 looked like a washout
(every AI link broken, SPY under its 50) and it bounced. That is arm W. (Corrected in review: at
the pre-registered WASH 0.15 the W rule does **not** fire on Sep 15. Semis (other) was at 0.22,
Datacenter builders at 0.17 and AI servers at 0.16. It fires on 2026-10-16 instead, six sessions
into the window after the Oct 8 break.)

## Hypotheses

- **H1 contagion.** When one link of a chain freshly breaks (≤ 10% of its names over their 21 EMA)
  while the rest of the chain holds (median link ≥ 40%) and SPY is over its 50 EMA, the rest of
  the chain does worse over the next 10 sessions than on the same chain's days with the same
  conditions and no broken link.
- **H1b the high-beta link breaks first.** The effect is concentrated where the broken link is the chain's most
  volatile one (the CoreWeave/neocloud reading: the most leveraged end breaks first).
- **H2 capitulation floor.** When every link of a chain is broken (≤ 15% over their 21) and SPY is at or under
  its 50 EMA, the chain does better over the next 10 sessions than the chain's other days in the
  same weak tape. Read as a do-not-short rule.
- **M reverse propagation (diagnostic, not voted).** A link freshly ≥ 90% over its 21 while the rest
  lags and SPY is weak is followed by the rest catching up. If H1 is a lead-lag effect, M should be its
  mirror image.

**Prior from LESSONS and the house record (written before any outcome):**
- **Daily industry breadth has no memory** (`industry_rotation`). The one lean: industry-wide
  100%-down days lean up next day (+26 bp, sub-bar). That is a capitulation bounce, so it is a prior
  for H2.
- **Group strength has no continuation at the month grain, and no price key flags a theme early**
  (`theme_leaders`). The groups-tab IC is slightly negative, which is the stock-level one-month
  reversal.
- **A leader's break is followed by nothing in its group** (scratch read 2026-10-13, no study
  folder). The theme_leaders panel had 3,849 top-3 leader breaks under the 21. Group-mates were
  down 46/44/42% of the time at 5/10/20 sessions, against 44/45/42% on random group-days.
  - Leaders break late: 58% of the median group is already under its 21 at the break.
  - H1 differs in two ways. It looks at a whole link of an economic chain going to zero breadth
    (not the strongest name), and the outcome is the *other* links.
- **Regime gates are beta pass-through** (`regime_gate`, `index_low_pullback`). Any H1 effect that
  is just "the market fell" must show up in the cross-chain placebo too.
- **High ADR in a weak tape is capitulation, never a short**: under SPY's 50 EMA the short lost in
  3 of 4 half-cells (`high_adr_short`). That is a prior for H2.
- **Today's curated theme lists are survivorship machines** (`theme_rs_divergence`). Every
  forward line here is relative to the same-day pond and to the chain's own control days, and any
  claim must hold on the family-only chains.

**Outside priors:**
- Economically linked firms show lead-lag at the month grain. Customer returns predict supplier
  returns (Cohen & Frazzini 2008), and the effect extends to linked industries along the supply
  chain (Menzly & Ozbas 2010).
- Inside an industry, big firms lead small ones (Hou 2007).
- Industry momentum is a known effect (Moskowitz & Grinblatt 1999).
- Momentum crashes happen in rebounds after market declines (Daniel & Moskowitz 2016), a prior
  for H2.

**Expected:** H1 is uncertain. The literature leans yes at the month grain, the house record leans
no at the day-to-month grain. H2 leans yes but may be sub-bar (one-tape episodes).

## Rules (pre-registered 2026-10-23, before any forward return was computed)

| Item | Definition |
|---|---|
| layers | **us11** (voted): the theme_leaders study's panel, calendar 2015-06-04 → 2026-08-24 (the 08-25 bench row is a mid-session print, dropped). **live** (descriptive only): `out/panel_live.npz` from `build_live.py` |
| eligible(i, t) | as theme_leaders: true close ≥ $5, median close × volume over the last 20 rows ≥ $5M, ≥ 60 prior rows, valid r20 |
| EMA | pandas `ewm(span=n, adjust=False, ignore_na=True)` on each name's adjusted close over its whole history; the index EMA50 on the index's own bench history |
| link | a `groups.assign()` group. Valid on t with **≥ 6** eligible names (`LINK_MIN`) |
| b21(g, t) | share of g's eligible names whose close is over their EMA21 (b50 likewise, sensitivity) |
| chain | the 12 groups-of-groups in `chains.py` (table below). Each group is in at most one chain; Agriculture & fertilizers, Consumer staples, Funds & trusts and "(other)" sit in none (they still serve as placebo groups) |
| chain kind | family / curated / mixed by the majority kind of each link's names; **family-only chains are the hindsight-free check** |
| index up | SPY close > SPY EMA50 on t |
| fresh break | b21(g, t) ≤ **0.10** and max b21(g, s) ≥ **0.50** over the 20 sessions before t (valid sessions only) |
| **C event** | index up; ≥ 3 valid links; ≥ 1 link freshly broken; the rest = valid links with b21 > 0.10, ≥ 2 of them, median b21 ≥ **0.40**. One event per chain per **20** sessions (first one kept) |
| C control day | the same chain, index up, ≥ 3 valid links, the event's broken link(s) valid, **no valid link ≤ 0.10**, the other links' median ≥ 0.40, ≥ 2 other links |
| fragile link | the chain's valid link with the highest stdev of its daily median-name return over the prior 250 sessions (≥ 120 days), on t |
| **P placebo** | cross-chain: a group from **outside** the chain freshly broken while the chain holds (index up, ≥ 2 valid links, none ≤ 0.10, median ≥ 0.40); one per (chain, group) per 20 sessions. Its control: the chain's days with the same holding condition |
| **W event** | SPY close ≤ its EMA50; ≥ 3 valid links; **every** valid link b21 ≤ **0.15**; one per chain per 20 sessions. Control: the chain's days with SPY ≤ its EMA50 and ≥ 3 valid links (any breadth) |
| **M event** | SPY ≤ its EMA50; ≥ 3 valid links; a link with b21 ≥ 0.90 that was ≤ 0.50 in the prior 20; the rest (b21 < 0.90, ≥ 2) median ≤ 0.60. Control: same chain, same index state, excluded link valid, no link ≥ 0.90, the other links' median ≤ 0.60 |

**The 12 chains** (`chains.py`; link validity on the 11-year calendar from `out/events_us11.log`):

| Chain | Links (groups) | Kind |
|---|---|---|
| AI compute capex | 7 (one valid from 2024) | mixed |
| Software | 7 (one valid from 2021) | mixed |
| Internet & media | 4 (one valid from 2021) | mixed |
| Speculative growth | 3 (all valid from 2021) | curated |
| Oil & gas | 5 | curated |
| Metals & materials | 4 (one valid from 2025) | mixed |
| Clean energy | 3 (valid from 2020, 2021 and 2024) | curated |
| Banks & credit | 5 | family |
| Rate-sensitive | 4 | family |
| Consumer discretionary | 6 | family |
| Industrial cycle | 4 | family |
| Healthcare | 4 | family |

The full link lists are the workbook's group map and are not reproduced here; the groups named in
the text are examples from it.

**Outcomes.**
- fwd_k(i, t) = close(t+k) / close(t) − 1 for k = 5, **10** (primary), 20, winsorized ±50%, for
  names eligible on t with a close at t+k. Entry at the event close: the breadth is known from
  closes at about 15:55.
- core(e) = the eligible names of the rest links (C, M), of the whole chain (P, W).
- y_raw(e) = median fwd_k over core(e). pond(t) = median fwd_k over every eligible name.
  **y_ex(e) = y_raw − pond** is the verdict line; y_raw is reported beside it (it is the short's P&L
  before borrow and the 6 bp).
- **Contrast d(e) = y_ex(e) − baseline**. The baseline is the mean y_ex over the arm's control days
  for the same chain (and the same excluded link set for C / M) in the event's calendar year. If
  that year has < 20 control days, the chain's all-year control mean is used (counted). Raw
  contrasts are built the same way.
- Also reported per event: the broken link's own y_ex, SPY and QQQ fwd_k.

**Statistics.**
- Mean d over events, its median, and P(d < 0).
- The SE is clustered by the event's calendar month (CR1), because chains fire together in
  market-wide moves. z = mean / SE.
- Halves split at 2021-03-19 (the theme_leaders median session). A leave-one-year-out sign count
  is reported (LESSONS §1: the episode is the dependence unit for regime claims).
- **Real − placebo:** C's mean d minus P's mean d. Its SE comes from a calendar-month block
  bootstrap over the union of months, 2,000 draws, seed 20261023.

**The bar** is the house one: |z| ≥ 2.5 and the same sign in both halves.

**Verdict rules (pre-registered).**
- **H1 keep** needs all of these at k = 10 on y_ex:
  - mean d < 0 with z ≤ −2.5, and both halves < 0;
  - family-only chains < 0;
  - real − placebo < 0 with bootstrap z ≤ −2.0;
  - k = 5 and k = 20 the same sign.
- If H1 holds, the claim is "a chain-link break is a reason to lean against the rest of the chain
  for about two weeks". It would become a radar or workbook flag only on a separate go-ahead.
- **H1b keep** needs the same bar on the fragile-link events alone, plus fragile − non-fragile < 0.
- **H2 keep** needs mean d > 0 with z ≥ 2.5 and both halves > 0. If it holds, the do-not-short
  reading gets a number. A positive raw median is reported beside it, because "don't short" needs
  the raw line too.
- Everything else is reported, never voted:
  - M;
  - the per-chain and per-year tables;
  - sensitivities: BREAK 0.0 / 0.20, HOLD 0.30 / 0.50, b50 instead of b21, QQQ instead of SPY,
    entry one session later, LINK_MIN 10, cooldown 10 / 40;
  - the live replay of Aug–Oct 2026.

## Amendment 1 (2026-10-23, before any forward return was computed)

Three independent reviewers read the rules above and the code before any outcome existed: one
for look-ahead and spec-vs-code consistency, one for statistics, one for economic design. They
used information up to each event date only. **Where this section differs from the Rules above,
it wins.** The definitions are frozen by hash: analyze.py refuses to run unless `chains.py` and
`events.py` match these:
- `chains.py` sha256 `649e0e2a1718a03387b7dac4b9a1615341280ca6829242b0e9340cd26ea49e16`
- `events.py` sha256 `7dc562a4573edcbf1119db171f3a28fea8df59c1dd2bc587cfff616ddb2d7445`

**Events.**
1. **The cooldown is per (chain, broken link)**, not per chain. Links that break on the same day
   merge into one event. Under the chain-level rule every neocloud break in the 11-year layer was
   swallowed by an earlier break of another AI link, so the exact motivating case was never tested.
   The chain-level rule is kept as a sensitivity.
2. **No stale-broken link on the event day.** Every valid link at ≤ 0.10 must be a fresh break.
   Before this change, 8 events had a long-dead link that dropped out of the core. M follows the
   same rule for links ≥ 0.90.
3. **The fresh-break window needs ≥ 15 size-valid sessions** of the prior 20. Before, it could
   rest on 1–3.
4. **A link takes part in its chain only if it co-moves with it.** On t, the link needs ≥ 6
   eligible names (`LINK_MIN`) *and* a correlation ≥ **0.30** between two series over the 250
   sessions before t (≥ 120 paired days): its median-name return minus the pond, and the mean of
   the other links' median-name returns minus the pond. Hand-map links that were not economically
   linked at the time drop out by themselves. Datacenter builders vs the AI rest ran −0.09…−0.01
   in 2017–21 and 0.41–0.83 from mid-2024. The static map is a sensitivity.
   - Cost: Industrial cycle (Freight 11%, Marine 0%, Aerospace 2% of sessions valid) and most of
     Rate-sensitive (Utilities 5%) almost never qualify. That is the point: those links do not
     move together beyond the market.
5. **Fragile link = the valid link with the highest beta** of its daily median-name return on
   the mean of the other valid links' median-name returns, over the 250 sessions before t
   (≥ 120). Beta is not inflated by a thin link's noise, so it does not favor small links. The
   first definition, the stdev of the link's median-name return, picked the smallest link in 151
   of 296 events.
   - Sensitivity: the link whose names have the highest median own-return stdev over the prior
     250 sessions.
   - Both use t − 1 and earlier.
   - Events with no fragile label are excluded from both sides of H1b.

**Outcomes and baselines.**
6. **One event set for every horizon**: events need t + 20 inside the panel, so the last
   usable event date is 20 sessions before 2026-08-24.
7. **Overlap exclusion.** A control day s enters a baseline only if no *qualifying* day of the same
   arm and chain lies in [s − k, s + k]. Qualifying days are counted before the cooldown, with
   any broken set (P: per outside group h).
8. **Matched baseline (C, P, B, M; voted).** The core's trailing 20-session excess is the median
   over the core's names of clip(close(t)/close(t−20) − 1, ±0.5), minus the pond's. Events and
   control days are cut into terciles of it, with the cut points taken on that cell's control
   days pooled over all years. The baseline is the mean y over the control days in the event's
   year and tercile. If that year has < 20 of them, all years in the same tercile are used
   (counted).
   - Why: on event days the rest of the chain had already lagged its control days by −238 bp over
     20 sessions (curated chains −652). The house one-month reversal would build in a result by
     itself. Terciles cut that gap to about −57 bp; the residual is reported as a balance line.
     These are the reviewers' pre-outcome estimates. On the final event set the gap is −372 bp
     unmatched and −99 bp after matching (Results).
   - The unmatched, same-year baseline is reported beside it.
9. **W control** = the chain's days with SPY at or under its EMA50, ≥ 3 valid links, and **at
   least one valid link > 0.15** (so not a washout), with the overlap exclusion.
   - Before, 12.8% of W control days were washouts.
   - The second control, "breaking, not washed" (median link share in (0.15, 0.40]), is reported.
10. **P control** for outside group h = the chain holding with h valid and > 0.10, the overlap
   exclusion applied to h's qualifying days. **P is collapsed to one row per (chain, date)**, its
   d = the mean over the outside groups that broke that day. Before, one broad sell-off day
   counted up to 12 times.
11. **B, same-date bystanders (new).** On each C date, every other chain that holds that day
   (index up, ≥ 3 valid links, none ≤ 0.10, median ≥ 0.40). The core is the whole chain; the
   baseline is its own holding days, matched the same way, with its own C-qualifying days
   excluded. **C − B** pairs each C event with the mean d of its bystanders. The same date nets
   out the market path.
12. **y_beta (new)** = the median over core names of (fwd_i − beta_i × SPY fwd_k). beta_i is the
   name's beta to SPY over the 250 sessions before t (≥ 120). It is the beta-neutral version of
   the outcome.
13. **Closes are split-adjusted price closes, dividends not adjusted** (us_universe, the daily
   store, both benches). Ex-dividend drops lower close vs EMA21, and fwd_k is price-only. A
   sensitivity drops events whose broken group is Marine shipping, Midstream & LNG, REITs,
   Utilities or Oil producers & royalties.

**Statistics.**
14. **Voted z's come from a cluster bootstrap that re-estimates every baseline in every draw**
   (2,000 draws, seed 20261023). The clusters are calendar months for C / P / B / M, and **SPY-
   at-or-under-EMA50 episodes for W** (spells less than 20 sessions apart merge). CR1 z with fixed
   baselines is reported beside them. C − P and C − B draw the same months for both arms.
15. **C − P is chain-reweighted**: Σ_c w_c (mean d_C(c) − mean d_P(c)), where w_c is chain c's
   share of the C events. P's own mean is reported as the null-center check.
16. **Power**, from a pre-event proxy (the review): the effect detectable at 80% power and |z| 2.5
   over 10 sessions is about 70 bp for C, 45 for the family chains, 90 for fragile-link events and
   110 for W. A kill rules out only contagion effects bigger than that. It is not evidence against
   month-grain lead-lag of the literature's size, about 50 bp per 10 sessions.
17. **k = 10 was chosen with the Oct 8 outcome in view; k = 20 is the literature's horizon.** Both
   must agree in sign.

**Verdict rules (these replace the ones above).**
- **H1 keep** requires all of the following at k = 10 on the matched d_ex:
  - (i) mean < 0 with bootstrap z ≤ −2.5, and both halves < 0;
  - (ii) the five family chains **on TradingView-only labels** (groups.FAMILY[industry], no
    themes.csv, no HAND, so today's theme winners return to their families): mean < 0,
    bootstrap z ≤ −2.0, both halves < 0;
  - (iii) C − P (chain-reweighted) bootstrap z ≤ −2.0;
  - (iv) C − same-date bystanders bootstrap z ≤ −2.0;
  - (v) k = 5 and k = 20 the same sign;
  - (vi) the sign holds with any one calendar year dropped;
  - (vii) the d_beta mean < 0 in both halves.
- **Labels for partial passes:**
  - (i) fails → **kill**;
  - (i) holds but (iii) or (iv) fails → "a market-break effect, not a chain effect; no chain flag";
  - (i) holds but (ii) fails → "curated-only, hindsight; no flag";
  - (i) holds but (vii) fails → "beta / market timing";
  - all hold but BREAK 0.00 / 0.20, HOLD 0.30 / 0.50 or index QQQ flips the sign → "lean,
    threshold-fragile".
- **H1b** is tested only if H1 keeps. It needs:
  - fragile-link events (beta definition) meeting bar (i);
  - fragile − non-fragile within chain (per-chain differences weighted by min(n_fragile,
    n_non-fragile), chains with both types only) with bootstrap z ≤ −2.0;
  - the name-vol definition giving the same sign.
  - Otherwise: "not distinguishable from H1".
- **H2 keep** ("washouts bounce relative to other weak-tape days") is voted on **d_raw** with the
  unmatched baseline, because a do-not-short decision is about the raw P&L. It needs:
  - mean > 0 with episode-bootstrap z ≥ the two-sided 0.0124 critical value of t with G − 1
    degrees of freedom (G = episodes with a W event, quoted);
  - both halves > 0;
  - the sign kept with any one episode dropped;
  - the family TradingView-label chains > 0;
  - k = 5 and k = 20 the same sign;
  - the matched version the same sign with z ≥ 2.0. If only the matched line fails, H2 is
    written as "the generic one-month reversal, already in LESSONS", not as a washout property;
  - the same sign with z ≥ 2.0 after dropping the survivorship windows: Banks & credit
    2023-03..05 (SVB, FRC, SBNY, PACW are missing), Oil & gas 2015-12..2016-04 and
    2020-03..06 (shale deaths).
- **Wording of H2:** "do not short a washed-out chain" needs the mean y_raw > +6 bp in both halves
  and a median y_raw ≥ 0. Otherwise: "washout shorts earn less than other weak-tape shorts".
- **Reported, never voted:**
  - M;
  - by chain, by year, the broken link's n tercile, single vs multi-link;
  - the broken link's own d: own d ≤ 0 with a rest d < 0 reads "contagion", own d > 0 reads
    "convergence";
  - the equal-weight market gate (share of all names over their 50 ≥ 0.5);
  - the precision-weighted mean;
  - the point-in-time baseline (control days in the 250 sessions before t whose windows closed
    by t);
  - the halves restricted to chains with ≥ 3 events in each half;
  - W preceded by C within 20 sessions, and C followed by W within 10 (raw lines);
  - the sensitivities: key b50, entry one session later, static map, chain-level cooldown,
    control with every link ≥ 0.30, broken link n ≥ 8, no dividend groups, LINK_MIN 10,
    cooldown 10 / 40;
  - the motivating case;
  - the live replay.

**The motivating case (never voted).** Every neocloud-broken C event:
- us11: 2025-11-11, 2025-12-12, 2026-02-04. 2024-12-23 fails the co-movement test;
- live: adds 2026-10-08.

Each is shown with the rest's raw and matched lines, the neocloud link's own line, whether QQQ was
over its 50, and Crypto's breadth that day. The neocloud link was mostly ex-bitcoin miners until
2025 and co-moved more with Crypto than with the AI rest on every us11 episode. Only on
2026-10-08 is it AI 0.75 vs Crypto 0.58. **n ≤ 4, all in half 2: this question is answered
only descriptively.**

**Event counts after the amendment** (`events.py`, no forward returns read):
- **us11:** C 196 (87 months; halves 75 / 121). Of those, 88 are fragile under the beta
  definition and 54 under the name-vol one, and 45 are multi-link. By kind: curated 81, mixed 59,
  family 56. P has 4,129 rows on 2,533 chain-dates; B 801; W 142; M 66.
- **TradingView-label family chains:** C 54 (43 months; 24 / 30).
- **live:** C 106, W 79. C fires on 2026-09-04 (Semicap + Datacenter builders), 2026-10-08
  (neoclouds = the fragile link, rest median 0.53) and others. W fires for AI compute capex on
  2026-10-16.
- **Used in Results:** item 6 keeps only events with t + 20 inside the panel (last event date
  2026-07-27), which drops one C event, one W, one M, 4 B rows and 94 P rows. Results therefore use
  C 195 (86 months; 75 / 120), W 141, M 65, B 797 rows, P 4,035 rows (2,484 chain-dates once
  collapsed), and family chains C 53 (24 / 29).

## Assumptions (stated explicitly, PROFILE.md rule)

1. **Survivorship:** the book alive in 2026-08 (us11) and 2026-10 (live). Forward lines are
   relative to the same-day pond and to the chain's own control days for that reason.
2. **Today's labels applied backwards** (themes.csv, HAND, TradingView industry). The curated
   chains are flattered by hindsight, and the family-only chains are the check.
3. **The chain map is hand-made** on 2026-10-23, after seeing the Oct 2026 example. Only the AI
   chain was in view when it was written; the other 11 chains follow the standard economic sector
   links. The thresholds (0.10 / 0.40 / 0.50 / SPY EMA50) are round numbers chosen with the Oct 8
   example in mind (rest median 0.53 that day), which is why the sensitivities are reported.
4. This is a reading signal, not a trade. A short on the rest of a chain pays the 6 bp round trip
   and borrow (0–80 bp a night on hot names), unmodeled here.
5. Entry at the event close assumes the breadth is computed a few minutes before the close; the
   one-session-later entry is the conservative sensitivity.
6. Event counts were checked before any outcome existed (`events.py`, no forward returns read).
   **Superseded by Amendment 1:** under the rules as first written, us11 had C 302 (105 months,
   134 fragile-link; halves 138 / 164, first written here as 147 / 155 by mistake), W 185, M 119,
   P 6,003, and live had C 148, W 107, M 75, P 3,493. The counts the study uses are listed under
   Amendment 1. On the live layer the C rule fired on **2026-10-08** and **2026-09-04** under both
   versions.

## Scripts

- `chains.py`: the chain map and every threshold.
- `lib.py`: panel loaders, index EMAs, group breadth.
- `build_live.py`: the live panel from the daily store → `out/panel_live.npz` (gitignored).
- `events.py --layer us11|live [--key b21|b50]`: events and control masks →
  `out/events_<layer>.csv`, `.log`. Counts only.
- `analyze.py [--excl past|both]`: outcomes, matched baselines, the cluster bootstrap, gates and
  sensitivities, about 15 min. `past` (default, Amendment 2) writes `out/`; `both` (Amendment 1 as
  run) writes `out/asrun_amendment1/`. It refuses to run if `chains.py` / `events.py` differ from
  the hashes in Amendment 1.
- `replay.py`: the live layer, Aug–Oct 2026, day by day for the AI chain → `out/replay.log`.
- Verification scripts (independent rebuild, code tests, post-hoc audit) lived in the session
  scratchpad and are not kept. Their findings are Amendment 2 and the "as pre-registered" column.

## Amendment 2 (2026-10-23, after the results: corrections found in verification)

A three-agent verification ran after the first full run:
- an independent rebuild from this README alone, without reading analyze.py or events.py;
- a code review;
- a post-hoc audit.

The rebuild reproduced every headline number to floating-point precision, and the review found no
coding error that moves a gate. **They found one design flaw in Amendment 1 item 7.**

**The forward half of the overlap exclusion is a look-ahead.** Dropping a control day s because a
qualifying day falls in (s, s+k] keeps only control days whose own outcome window holds no break
(C) or no washout (W). That selection lifts the control mean:
- C holding days dropped by the forward half earned −169 bp over 10 sessions, against +33 for
  the days kept;
- W control days dropped by it earned −609 bp raw, against +261.

The events are not filtered the same way: 62% of C events and 70% of W events have another
qualifying day inside their own window. **The corrected rule excludes only [s − k, s]**
(`analyze.py --excl past`, the default, writing `out/`). The pre-registered version is kept as run
(`--excl both`, writing `out/asrun_amendment1/`).

Smaller fixes, none of which changes a gate:
- The BREAK / HOLD sensitivities now score their control days at the variant's thresholds. Before,
  they used 0.10 / 0.40.
- The log's P fallback rate was hard-coded to 0%; it now prints the true share.
- The B line is deduplicated: when two chains fired on one date, each bystander chain counted once
  per firing chain (797 rows, 750 after deduplication). The bootstrap's B (+5.3 bp) still counts
  every row; the Results table shows the deduplicated mean.
- C − P renormalizes its chain weights inside a draw. Before, 22% of draws were dropped.
- A "without thin baselines" line is added. Five 2026 Clean energy events had all-years cells of
  3 control days.
- Spec gaps, now recorded:
  - The cooldown is a gap > 20 sessions.
  - A merged event's broken set is every link broken that day.
  - A cell with < 60 control days gets no terciles and uses the unmatched baseline.
  - The rho and beta series use size-valid links only.
  - The W core is the valid links' names.
  - The H2 fallback sentence in the verdict rules ("washout shorts earn less…") pointed the wrong
    way. The raw short numbers below replace it.

## Results (2026-10-23)

Logs: `out/analyze.log` (corrected), `out/asrun_amendment1/analyze.log` (as pre-registered),
`out/replay.log` (live). Tables: `out/*.csv`. k = 10 unless noted, in bp. The z is the cluster
bootstrap with baselines re-estimated (months; W: SPY-under-50 episodes).

### H1 contagion: 195 events, 86 months, halves 75 / 120

| Line | Corrected (past-only) | As pre-registered |
|---|---|---|
| C d_ex matched (verdict line) | **−5.6 (z −0.15)**; halves −11.5 / −1.9 | −73.7 (z −1.83); halves −51.6 / −87.5 |
| k = 5 / k = 20 | +12.7 / −36.7 (z +0.56 / −0.64) | −23.1 / −207.6 (z −0.97 / −3.09) |
| family chains, TradingView labels (n 53) | +25.6 (z +0.93) | +5.9 (z +0.19) |
| C − P (chain-reweighted) | +12.9 (z +0.31) | −76.1 (z −1.73) |
| C − same-date bystanders | −11.8 (z −0.34) | −51.8 (z −1.36) |
| d_beta (beta-neutral) | +22.0 (halves −9.2 / +41.6) | −47.0 |
| P null center / B (deduplicated) | −10.6 / +9.2 | +2.9 / −21.0 |
| fragile (beta) / fragile − non-fragile within chain | −9.7 / +5.9 (z +0.08) | −59.4 / +31.4 |

- **The rest of the chain does not fall after a link breaks.**
  - Its median name rose a mean +51 bp and a median +50 bp over the next 10 sessions; it fell on
    only 43% of events (SPY +61 bp over the same windows).
  - Against its own matched holding days it is −6 bp. Every chain-specific contrast is within
    ±13 bp of zero.
- The pre-event balance is still uneven after matching: on event days the rest had trailed its
  matched control days by −99 bp over 20 sessions (−372 unmatched). The house one-month reversal
  would push d *up*, so it cannot hide a negative contagion effect.
- **The as-run lean was the look-ahead.** Its k = 20 line (−208 bp, z −3.09) also came from two
  curated chains: Oil & gas held 66% of the summed d and Speculative growth (n 7, 2025–26) 39%.
  The families were positive (+43) and C − B failed. Without both chains it was +20.
- By chain (corrected): Oil & gas −54 (n 69), Speculative growth −327 (n 7), Clean energy −290
  (n 5, thin baselines), AI compute capex **+64** (n 27), Banks +59, Software +138. The
  precision-weighted mean is +31.
- Sensitivities (corrected): every cell is within ±33 bp with |z| ≤ 1.31 (the static map −27,
  z −1.31; BREAK 0.00 −33). Nothing flips into a signal.
- **The broken link itself** goes on to do −1.7 bp vs the same control days: no contagion and no
  convergence.
- **M reverse propagation** (diagnostic): −74 (z −1.11), the wrong sign for lead-lag.

### H2 capitulation floor: 141 washouts, 28 SPY-under-50 episodes (t critical 2.68)

| Line | Corrected | As pre-registered |
|---|---|---|
| W d_raw vs weak-tape non-washout days | **+20.8 (z +0.17)**; halves −211 / +152 | −255.2 (z −1.96) |
| matched / survivorship windows dropped | −36.9 / +20.4 | −286.1 / −254.1 |
| k = 5 / k = 20 | −26.2 / +149.9 | −153.4 / −273.0 |
| vs "breaking, not washed" days | −3.7 | −203.7 |
| d_ex (chain vs market) | +6.6 | −82.0 |

- **Washouts are neither better nor worse than the chain's other weak-tape days.**
- The control set leans to early-recovery days: 39% fall after the episode's SPY low (hindsight),
  against 7% of washouts, and 87% of washouts print within 0.5% of SPY's running low.

### What a short would have earned

Raw short P&L = −y_raw − 6 bp, before borrow. Mean / median / hit rate.

| Short | 5 sessions | 10 sessions | 20 sessions |
|---|---|---|---|
| the rest of the chain after a link breaks (195) | −22 / −50 / 42% | −57 / −56 / 43% | −50 / −91 / 44% |
| … AI compute capex only (27; 19 in 2021–26) | −87 / −31 / 44% | −13 / +62 / 56% | +73 / +119 / 59% |
| … Oil & gas only (69) | +56 / +8 / 52% | +11 / −29 / 46% | −56 / −30 / 45% |
| a washed-out chain, SPY under its 50 (141) | −6 / −60 / 44% | −59 / −136 / 43% | −235 / −329 / 34% |

- The washout short at 10 sessions has a p10 of −989 bp and a p90 of +920. It was positive in
  7 of 28 episodes at 10 sessions and 2 of 28 at 20.
- Hedged with a SPY long, both shorts net about 0.
- The AI-chain line has a positive median on small n with fat tails, and its 20-session mean
  comes from the 8 first-half events. It does not survive the matched comparison (+64 bp the
  wrong way).

### The motivating case and the live replay (never voted)

| Neocloud break | Rest of the AI chain, raw 10 sessions | vs matched holding days | QQQ over its 50 | Crypto b21 |
|---|---|---|---|---|
| 2025-11-11 | −4.6% | −1.1% | yes | 0.00 |
| 2025-12-12 | −1.2% | +3.0% | yes | 0.15 |
| 2026-02-04 | +9.3% | +11.8% | no | 0.00 |
| **2026-10-08** (live) | **−20.6%** | **−22.0%** | yes | — |

- On the live layer the rule also fired for the AI chain on **2026-09-04** (Semicap + Datacenter
  builders broke): the rest went **+4.3%** over 10 sessions (+8.5% vs baseline).
- Of the four neocloud breaks, only the October one was followed by a large fall. The three in
  the 11-year layer ran −1.1%, +3.0% and +11.8% against matched holding days.
- Three of the 2025–26 breaks came with Crypto fully broken the same day (b21 0.00–0.15). The
  neocloud link was moving with bitcoin miners, not with AI demand.

## Verdict (2026-10-23)

**Kill H1, contagion.** "A link of a chain breaks while the rest and SPY hold, so lean against
the rest" fails every voted gate under the corrected baseline.
- The matched contrast is −6 bp (z −0.15). It is +13 against the cross-chain placebo, −12 against
  same-date chains and +26 on the family chains.
- The rest of the chain rose on 57% of events, so the short lost a mean 57 bp (median 56) per 10
  sessions.
- The pre-registered version failed too (z −1.83 against a −2.5 bar). Its lean was the
  forward-looking control selection plus two curated chains.
- **H1b, the high-beta link breaking first, is not distinguishable** (+6 bp vs other links). The
  CoreWeave reading gets no support: of the four neocloud breaks, only the live Oct 8 one was
  followed by a large AI-chain fall.

**Kill H2, the capitulation floor, in both directions.**
- A fully washed-out chain under a weak SPY does no better and no worse than the chain's other
  weak-tape days (+21 bp, z +0.17, halves disagree).
- So there is no washout-specific bounce rule. But the raw short loses (median −136 bp at 10
  sessions, −329 at 20, hit 34–43%), consistent with the house rule that high ADR in a weak tape
  is capitulation, never a short (`high_adr_short`).
- Read it as a reason not to short there, not as a reason to buy.

**What it means for the question.** The Oct 8 neocloud break followed by a −20% AI leg is
real, but it is one draw from a pattern that over 11 years is a coin flip. The same rule on the
same chain fired on Sep 4 and the chain rallied. Breadth inside a chain does not carry direction
(consistent with `industry_rotation`, `theme_leaders` and the 2026-10-13 leader-break read). Any
edge in reading "the AI trade is cracking" would have to come from something these prices don't
hold, if from anywhere. A testable next step would be the news layer, but only with
point-in-time data.

**Nothing is proposed for the workbook or the radar.**
