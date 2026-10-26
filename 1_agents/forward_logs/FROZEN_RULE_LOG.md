# Forward log: a frozen rule, scored every week, no orders

A backtest lead that was found after looking at the data cannot be confirmed on the same data. This log freezes the
rule and lets the future decide. Research only: neither the rule nor any result is published here.

The lead came out of the movers studies in the [study index](../../5_research/INDEX.md). It was found after looking,
it is absent in every wider population and it is concentrated in a few seasons, so it is a lead, not a result. Any
change to the rule starts a new log.

## Why version 1 was archived

Version 1 read the filings at run time, so its "known on the day" filing class included filings up to five sessions
after D, which nobody knows on the day, and one rule promised a timing clause the code never applied. Version 2 reads
only filings accepted by D's close, states the rule exactly as the code applies it, and rebuilds its backfill. Version
1's log is archived, not deleted. The log also keeps the backtest's hindsight version of the filing class in its own
column, so both versions can be scored side by side.

## The run

- `rule_log.py`, a weekly scheduled job: it tops up its own data store, processes list days whose 5-session window
  is complete, appends the log, re-scores every row and writes a summary.
- One Telegram message per run on the radar bot: the week's new entries and the scoreboard. A failed run sends one
  warning line, with the traceback logged.
- Columns: identification, the day's inputs and both filing-class versions, then outcomes updated weekly (best close
  since, 6- and 12-month results, whether the ticker stopped trading).

## Decision rule (frozen before the first forward entry)

Judged once at least 40 names in the test group have a 12-month outcome. Keep only if the test group beats the
same-week comparison group on the study's bar (doubling more often by a set ratio, halving no more often, a higher
median), on both filing-class versions. Quarterly six-month reads are progress reports, not decisions; backfill rows
never count toward the decision.

## Excerpt

From `rule_log.py` (the weekly job, about 470 lines; shown here only in part). The rule's parameters and every result
stay out.

**1. Filings with their acceptance time**: EDGAR's submissions file gives each filing's acceptance time in UTC; it is
converted to New York market time so it can be compared with the session close.

```python
def filings(cik):
    b = get(f"https://data.sec.gov/submissions/CIK{int(cik):010d}.json", SEC)
    if not b:
        return None
    r = json.loads(b).get("filings", {}).get("recent", {})
    recs = [x for x in zip(r.get("form", []), r.get("filingDate", []), r.get("acceptanceDateTime", []),
                           r.get("items", [])) if x[0] in CM.KEEP_FORMS]
    if not recs:
        return None
    d = pd.DataFrame(recs, columns=["form", "fdate", "accept", "items"])
    d["fdate"] = pd.to_datetime(d["fdate"])
    d["accept_et"] = (pd.to_datetime(d["accept"], utc=True, errors="coerce").dt.tz_convert("America/New_York")
                      .dt.tz_localize(None))
    d["items"] = d["items"].fillna("")
    return d.drop(columns="accept").sort_values("accept_et").reset_index(drop=True)
```

**2. Only what was public on the day** (inside the candidate loop): the filing class uses filings accepted by the close
of day D (the version-2 fix), and the market cap uses the last share count filed before D.

```python
            f = filings(cik) if cik else None
            f_day = f[f["accept_et"] <= close_et[i]] if f is not None else None   # only what was public by D 16:00 ET
            cls, _ = CM.classify(f_day, cal, i, close_et)
            sh = shares(cik) if cik else None
            known = sh[sh.index <= cal[i - 1]] if sh is not None else None
            cap = c[i - 1, j] * known.iloc[-1] if known is not None and len(known) else np.nan
```

**3. Every logged row re-scored each week**: outcomes are measured from the list-day close, a name that stopped trading
is flagged rather than dropped, and the filing class is re-read until its 60-session window closes, so the hindsight
version sits beside the on-the-day one.

```python
    if len(L):
        col = {s: k for k, s in enumerate(cols)}
        for n, row in L.iterrows():
            j, i = col.get(row["sym"]), int(cal.searchsorted(pd.Timestamp(row["date"])))
            if j is None or i >= T or cal[i] != pd.Timestamp(row["date"]) or not a[i, j] == a[i, j]:
                continue
            fut = a[i + 1:, j]
            ok_f = np.isfinite(fut)
            last = np.flatnonzero(np.isfinite(a[:, j]))[-1]
            L.loc[n, "sessions"] = T - 1 - i
            L.loc[n, "max_x"] = round(np.nanmax(fut) / a[i, j], 3) if ok_f.any() else np.nan
            L.loc[n, "now_ret"] = round(a[last, j] / a[i, j] - 1, 3)
            L.loc[n, "stopped_trading"] = bool(last < T - 6)
            for h, tag in ((126, "6m"), (252, "12m")):
                if i + h < T:
                    w = a[i + 1:i + h + 1, j]
                    end = a[i + h, j] if np.isfinite(a[i + h, j]) else a[min(last, i + h), j]
                    L.loc[n, f"max_{tag}"] = round(np.nanmax(w) / a[i, j], 3)
                    L.loc[n, f"ret_{tag}"] = round(end / a[i, j] - 1, 3)
            if T - 1 - i <= 66:                              # re-read filings until the 60-session window closes
                cik = keep.get(row["sym"], {}).get("cik")
                f = filings(cik) if cik else None
                L.loc[n, "filing_class_now"] = CM.classify(f, cal, i, close_et)[0]
                time.sleep(0.15)
        L.to_csv(LOG_CSV, index=False)
```
