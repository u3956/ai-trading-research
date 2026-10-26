# PROFILE: the file every study agent reads first

The single source of truth about the environment a study runs in. **Every study links here instead of re-deriving
these facts, and no constraint applies to a study unless it is written in that study's own `## Assumptions` block.**
A premise one study needs (for example the rules of an overnight-hold test) lives in that study's folder and stays
there, so it cannot leak into the next study as a silent default.

It exists because an AI agent fills gaps with plausible defaults. One such default, a holding-period premise that
belonged to a single study, once traveled into later ones before anyone noticed. Since then every fact a study may
rely on is either here, dated and confirmed by the owner, or stated in the study itself.

_Shown here with the owner's own trading details made generic; the structure is the one in use._

## How studies treat the owner's trading

- **Studies are decision support** for selection and direction questions ("which names", "which way"), not trading
  systems. They do not assume mechanical stops, forced exits or automation unless a study's plan asks for that exact
  simulation.
- **When a stop is simulated on request**, the house finding applies: a fixed-percentage stop across stocks of
  different volatility is an invalid expectancy metric (it pays a placebo entry on both sides; see
  [LESSONS.md](../5_research/LESSONS.md), section 2).
- **The universe of interest** is the screener's lists plus liquid US stocks generally, read with their sector context.

## House numbers

- **Costs:** 6 bps round trip on every net line.
- **Short side:** borrow tiers of 0 / 5 / 20 / 80 bp per night (hot names are hard to borrow). Slippage on wide-range
  names is real and usually unmodeled; a study says so when it is.
- **Eligibility floors:** previous close at least $5, and a 20-day median dollar volume of at least $5M.

## The screener's column definitions

The screener's code is the authority for every column a study uses. A study replicates the formula **verbatim and
cites the line**; it never writes its own version of a column with the same name. When a definition changes in the
screener, this file records the date, because thresholds quoted from older studies refer to the old series.

## Data environment

- **Broker files** are the tape minus odd lots. The measured VWAP effect of that is at most 0.5 pp.
- **Consolidated minute history** comes from the downloaders in
  [../2_pipelines/market_data/](../2_pipelines/market_data/README.md), verified against the broker's files before use.
  Yahoo's free 1-minute window only reaches back about 30 days.
- **The screener's stores are read-only for studies.** They are grown by the screener's own nightly scripts; a study
  only reads them through the shared loader or a direct path. Two old scripts that once wrote there now refuse to run
  at those paths. Bulk pulls go to a git-ignored folder inside the research repo.

## For any new study

1. Copy the template and write the hypothesis and the rules **before** any result exists
   ([template](../5_research/template/README.md)).
2. State every constraint, cost, metric and data limitation in the study's README, explicitly, even the obvious ones.
   This is the owner's standing rule.
3. Link this file. If a fact here is wrong or stale, fix it here, not in a study README.
4. Build cohorts from the stores (bars, minute, benchmark, metadata, themes, earnings, the workbook's snapshot), never
   from retired outputs. Old hit lists stay on disk only so the finished studies that read them still run.
