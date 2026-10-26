# Market-data downloaders

Intraday studies need years of 1-minute bars, more than the free rolling windows of the nightly pipeline keep.
Three downloaders fill that gap. Each writes **the same file schema and file names** (`{SYM}_1min_{rth|ext}_long.csv`,
the broker's columns, exchange-time stamps), so every study reads any source unchanged.

| Downloader | Source | What it is good for |
|---|---|---|
| `alpaca_history.py` | Alpaca market-data API, free plan | consolidated 1-minute history back to 2016, delisted stocks included, 200 requests a minute |
| `massive_history.py` | Massive (formerly Polygon.io), free plan | consolidated 1-minute bars, two years back, 5 calls a minute (an overnight run for a universe) |
| `broker_history.py` | the broker's API | walks backwards from the last completed session, inside the broker's request limits |

This folder shows **how the sources are pulled and checked against each other**. The downloaders appear here only as
the excerpts below.

## How they are built

- **One schema for every source.** Timestamps are converted to the exchange's clock and written in the broker's format, so a
  study switches source by changing a file path, not its code.
- **Keys never travel in a URL.** Each key is read from a git-ignored secrets file or the environment, sent as a
  request header, and never printed.
- **Paced to each plan's limit**, with backoff on rate-limit answers and a clear message on a refused plan
  ("the free plan covers 2 years and excludes today").
- **Ended at a finished session.** Every pull ends at the last completed session's close, so a half-formed bar is
  never written.

## How they are checked

- **Source against source, minute by minute.** `--check` pulls the sessions an existing broker file covers and joins
  them: the median and 95th-percentile close difference in basis points, and the volume ratio. A ratio near 1.0 means
  consolidated bars; near 0.03 means single-venue bars. In use, Alpaca's free consolidated bars matched Massive's.
- **Bad prints are reported, not silently fixed.** The consolidated tape keeps off-exchange prints the broker's feed
  filters. A wick is flagged only when it sticks out far beyond its own bar's body *and* both neighbors, and never in
  the first or last five minutes of the session, where wide bars are real. Calibrated on one stock it scored three
  right and one wrong: too few cases to call it validated, so clipping stays off by default.

## Excerpts

**1. A paced request with backoff** (`massive_history.py`): the key goes in a header; rate limits and server errors
retry with growing waits; a bad key or a refused plan stops with the reason.

```python
def get(url, key, pacer, tries=4):
    req = urllib.request.Request(url, headers={
        "Authorization": f"Bearer {key}", "Accept": "application/json"})
    for attempt in range(tries):
        pacer.wait()
        try:
            with urllib.request.urlopen(req, timeout=120) as resp:
                return json.loads(resp.read().decode("utf-8"))
        except urllib.error.HTTPError as e:
            body = e.read().decode("utf-8", "replace")[:300]
            if e.code == 429:                       # over the rate limit
                time.sleep(15 * (attempt + 1))
                continue
            if e.code == 401:
                sys.exit(f"HTTP 401 (bad key): {body}")
            if e.code == 403:
                sys.exit(f"HTTP 403 (plan refuses this range): {body}\n"
                         "-> the free plan covers 2 years and excludes today; "
                         "try a shorter --months or an explicit --end.")
            if e.code >= 500 and attempt < tries - 1:
                time.sleep(5 * (attempt + 1))
                continue
            raise
    raise RuntimeError(f"gave up after {tries} attempts: {url}")
```

**2. Flagging isolated bad prints** (`massive_history.py`): a low (or high) is suspect only when it lies far beyond the
bar's open/close body and beyond both neighbors, measured in the session's median 1-minute range.

```python
def find_bad_prints(rows, clip=False):
    # ... (docstring: the rule, the example that motivated it, its calibration)
    import pandas as pd
    if len(rows) < 3:
        return rows, []
    # ... rows into a frame; each bar's day and minute of the session from its timestamp
    core = (mso >= SPIKE_EXEMPT_OPEN) & (mso <= 389 - SPIKE_EXEMPT_CLOSE)
    rng = (df.high - df.low).groupby(day).transform("median").replace(0, float("nan"))
    body_lo = df[["open", "close"]].min(axis=1)
    body_hi = df[["open", "close"]].max(axis=1)
    nb_lo = pd.concat([df.low.shift(1), df.low.shift(-1)], axis=1).min(axis=1)
    nb_hi = pd.concat([df.high.shift(1), df.high.shift(-1)], axis=1).max(axis=1)
    bad_lo = core & ((body_lo - df.low) > SPIKE_BODY * rng) & ((nb_lo - df.low) > SPIKE_NEIGHBOUR * rng)
    bad_hi = core & ((df.high - body_hi) > SPIKE_BODY * rng) & ((df.high - nb_hi) > SPIKE_NEIGHBOUR * rng)
    rep_lo = pd.concat([nb_lo, body_lo], axis=1).min(axis=1)
    rep_hi = pd.concat([nb_hi, body_hi], axis=1).max(axis=1)
    found = [(df.datetime[i], "low", float(df.low[i]), float(rep_lo[i])) for i in df.index[bad_lo]]
    found += [(df.datetime[i], "high", float(df.high[i]), float(rep_hi[i])) for i in df.index[bad_hi]]
    if clip and found:
        df.loc[bad_lo, "low"] = rep_lo[bad_lo]
        df.loc[bad_hi, "high"] = rep_hi[bad_hi]
        return df.values.tolist(), found
    return rows, found
```

## Built with

Python's standard library for the HTTP clients, pandas for the checks, and the broker's own API client. The plan
research (rate limits, history depth, which feed each free plan really serves) was checked against the live APIs
before any study relied on a source.
