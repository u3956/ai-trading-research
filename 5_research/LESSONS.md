# LESSONS: the house method, and the traps already paid for

Every new study applies this file. Each rule names the study that paid for it, and a study that breaks a rule here has
a bug, not a judgment call. AI agents working on a study read this file before writing any code, so the same mistake
is never paid for twice.

---

## 1. The bar a claim must clear

- **z ≥ 2.5 and the same sign in both halves of the data**, before any "keep". One condition alone is not enough: a
  cell that crossed z 2.5 on a covariance fix still died on its placebo and a year flip (`index_low_pullback`); a
  semis cell at z 3.0 dissolved into 7 episodes (`breakdown_short`).
- **Cluster by session.** Signals shared across stocks on the same day are one bet, not thousands (24k events ≈ 400
  effective bets). For regime claims the unit is the episode: require episode-out sign agreement (`regime_gate`).
- **Week-cluster any overlapping 5-day+ outcome.** Date clustering inflated those z's ~1.4× (`breakdown_short`).
- **A placebo line ±0.75 ATR on every level claim.** Every EMA/VWAP result so far was shared by its placebo except the
  daily EMA21's +1.4 pp (`level_reactivity`, `level_regime`, `weekly_ema_touch`).
- **Per-event nulls, never 50%.** A ±ATR race null sits at 0.29–0.47 depending on where the level is; a quiet-stock
  control for gap days sits at 45.8% (`level_reactivity`, `gap_up`).
- **No per-cell eligibility.** Choosing members inside a cell selects the bouncers into it (`level_regime`).
- **Count the clusters, not just the z.** With 10 year-blocks the 2.5 bar is really ~3.1; shifting block boundaries
  moved honest z's by up to 30%. Quote the cluster count beside any long-horizon z (`analyst_upside`).
- **A permutation placebo is valid only if its null is centered at zero.** A within-date permutation gave p < 0.001,
  but its own null mean was +80 bp; within-symbol pairing flipped the "edge" to −175 bp (`analyst_upside`).
- **Size a multi-row stop check on null worlds of the final design.** A centering check sized on a draft stopped 23 of
  480 pure-noise worlds by chance; re-cut on the final design's worlds, before any real read, it stopped 4 of 480
  while still catching the fault it exists for in 480 of 480 (`movers_grid`).
- **Run the whole pipeline on synthetic random walks before reading a real outcome, and set the family-wise bar
  there.** On prices where nothing can predict anything, a first draft still printed the expected sign: three design
  choices built it in, and three reviewers had missed two of them. On 118 cells, 82% of noise worlds had at least one
  |z| ≥ 2.5 that passed the halves rule, so the bar became |z| > 4.02 (`first_mover`).
- **Measure power with planted effects before calling a kill informative.** A planted 3% effect came back at about a
  third in one comparison design and at 81–89% in another (`first_mover`).

## 2. Metric traps: ways a backtest manufactures an edge

- **Fixed-%-stop expectancy across names of different volatility is invalid.** Capping losses at −1R while wins scale
  with range pays ~+0.3–0.4R to a placebo entry, on both sides of the trade (`breakdown_short`). Rank on raw or
  ATR-normalized outcomes. **The sign-flip test** is the cheap check for any new metric: if the mirrored trade also
  "wins", the metric is broken.
- **A hit-rate lift is not an expectancy lift when the misses are the crashes.** A 59.6% hit cell had a +6 bp mean.
  Quote hit rate, mean and median together (`open_flush_bounce`).
- **Daily bars cannot simulate a tight stop.** Close-basis stops ran +0.21R optimistic against minute truth; report
  stop results as bands and calibrate coarse layers on a minute subsample (`breakdown_short`). Overnight stop legs
  fill at max(stop, next open): 25% of next-day stop hits gap through.
- **An hour-level flag for a minute-level condition is mostly noise** (75% contamination, contrasts at ~0.25× truth).
  Coarse layers bound, minute layers measure (`index_low_pullback`).
- **Time-of-day composition.** If a condition cannot fire in part of the session, that window floods the control arm
  and fakes a result. Stratify by time or do not compare (`index_low_pullback`).
- **A fill earns the stock's own drift.** Demean per symbol against a matched day-type baseline. **Hand-curated hot
  theme lists are survivorship machines**: today's theme membership crawled back 5 years prints ~+100 bp / 20 days raw
  that each stock's own mean removes entirely (`weekly_ema_touch`, `theme_rs_divergence`).
- **But own-demeaned flatters crashed names.** A crashed name's own mean contains its later decay, so any crashed
  name-day reads positive against it; judge against the same-day pond (`crashed_gap_basket`).
- **Measure gaps from the open.** The gap itself is not buyable (`gap_up`). Likewise **fill at min(line, open)**: a
  line the stock gapped through is unfillable; filling at the line faked +17 bp at z 3.8 (`adr_exhaustion_short`).
- **Hour-grain running-extreme lines must use the extreme known at the start of the bar.** Using the current hour's
  high faked a −110 bp (z −13) cell that minute bars put at −17 bp (`adr_exhaustion_short`).
- **MFE is not capturable R.** Quote what a defined exit collects (`breakdown_short`).
- **Attention is not information.** Against quiet controls, upgrades and downgrades both "won"; direction lives in the
  up-minus-down contrast, which was zero (`analyst_upside`). The same on +10% movers: a +10% day and a −10% day reach
  5x equally often against quiet look-alikes; both tails are fatter (`movers_grid`).
- **Breakeven arithmetic before building anything.** With stop rate p and cost c in R, survivors must average
  (p + c)/(1 − p) (`breakdown_short`).
- **Winsorize equal-weight means (±50%).** One bad +4,739% print moved a 2,500-name mean ~190 bp a day
  (`industry_rotation`).
- **Stops reshape, they never create.** If the no-stop mean is ≤ 0 net, no stop width fixes it (`breakdown_short`,
  `early_low_entry`).
- **A label normalized by pre-event ADR fires mechanically on earnings sessions.** Set reaction sessions aside as
  their own row (`first_hour_high_fade`).
- **Define event days by timestamps, never by the larger price move.** Picking the reaction session by its move
  builds the result into the selection (`forward_test_design`).
- **A trade that hinges on a locate, an opening cross or a squeeze fill cannot be tested on a paper account.** Use a
  no-orders shadow log scored from the tape (`forward_test_design`).
- **Never drop control days for what happens after them.** Removing control day s when an event falls in (s, s+k]
  keeps only controls whose own outcome window is clean, and when events and outcomes share a driver that inflates
  the baseline by construction. It faked a −74 bp contagion effect and a −255 bp washout effect; both were ≈ 0 with a
  past-only window [s−k, s]. It passed a three-reviewer pre-registration; an independent rebuild that did not read
  the code caught it (`supply_chain_contagion`).
- **One runner counts many times**: in a rate, in fake labels and across a time split. Quote base rates per entry
  day or one day per name per quarter; move a name's overlapping days as one block in fakes; drop earlier winners'
  names before confirming on a later period (`movers_grid`).
- **A null that rotates each name by its own offset manufactures leads.** For a lead-lag null, move every candidate of
  a unit by one offset; for the luck level of a best-of-N score on fat-tailed returns, use a sign-flip null
  (`bellwether_lead`).

## 3. Data rules

- **The screener repo is read-only for studies.** Its column formulas are the authority: replicate them verbatim and
  cite the lines.
- **One volume basis per calculation.** Broker volume ≈ 0.7× the consolidated tape (off-exchange prints filtered);
  never mix bases in one VWAP. A store that changed source mid-history carries two bases in one rolling window:
  splice it before computing any volume statistic.
- **Suspect low-wicks exist in every 1-minute pull**: define intraday extremes on close basis, keep wick basis as a
  sensitivity.
- **Guard adjustment bases at every join**: check the close ratio at seams; splits leave silent seams in stores
  pulled on different dates.
- **Never take a 20-session pre-event state from a short minute store**: take it from the daily file and join by date.
- **Bulk pulls go to a git-ignored folder**, never into tracked data.
- **Free daily history drops dead tickers.** Count the holes by outcome before trusting a mean; recycled tickers
  return another security, so check the first bar's date.
- **EDGAR text traps**: "complete response" in oncology is tumor response, not a rejection letter; risk-factor
  boilerplate carries dates that are not events.
- **Stored EDGAR acceptance times read as UTC are 4–5 h late for about a third of companies.** Their copies carry
  the New York offset added to the UTC time (a fresh download in October 2026 read correctly); the shift belongs to
  the CIK, not the file. Label each CIK by a
  vote of its filings that are impossible as stored against those impossible when shifted back (`movers_grid`).
- **Downstream code reads a decision's value, never the sentence printed beside it** (`movers_grid`).

---
_Add a line the day a new trap bites or a new fact survives review, with the study that paid for it._
