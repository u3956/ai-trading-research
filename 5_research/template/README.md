# <strategy name>

**Started:** YYYY-MM-DD
**Symbols:**
**Data used:** (files in `../data/`)

## Hypothesis

One or two sentences: what edge is being tested, and why it might exist.

## Rules

- Entry:
- Exit:
- Costs / sizing: (house default: 3 bps per side)
- Session window: (e.g. 09:30–15:55 ET)

## Scripts

- `explore.py` — starter, loads bars via `common.data`

## Results

(numbers, tables, pointers to output files in this folder)

## Verdict

(dated: **keep** / **kill** / **park** — and why; then update the study's row in the study index)

---

Tip: before writing a new backtest engine, reuse the existing one from the `ema_4cross` study (a trade
simulator over several entry and exit rules, plus a fast numpy twin checked against it). A fresh engine
brings fresh bugs.

## Excerpts

The template's code is not published in full. **`explore.py`**: every study starts from the same shared loader and
EMA columns, so two studies never compute the same number two ways.

```python
# repo root on sys.path so every strategy folder can import the shared common/ package
sys.path.insert(0, str(Path(__file__).resolve().parents[1]))
from common.data import DATA, load, add_emas
# ...
bars = add_emas(load(symbol, tag))
print(f"{symbol} {tag}: {len(bars)} bars  {bars.index[0]} -> {bars.index[-1]}")
```

**`common/data.py`, `add_emas()`**: the recursive EMA, with the slope test made exact rather than approximate.

```python
def add_emas(df):
    # adjust=False gives the recursive form:  EMA[t] = a*P[t] + (1-a)*EMA[t-1]
    df["ema9"] = df["close"].ewm(span=9, adjust=False).mean()
    df["ema21"] = df["close"].ewm(span=21, adjust=False).mean()
    df["ema9_prev"] = df["ema9"].shift(1)
    df["ema21_prev"] = df["ema21"].shift(1)
    # slope > 0  <=>  close > previous EMA   (exact, since EMA[t]-EMA[t-1] = a*(P[t]-EMA[t-1]))
    df["ema9_rising"] = df["close"] > df["ema9_prev"]
    df["ema21_rising"] = df["close"] > df["ema21_prev"]
    df["above"] = df["ema9"] > df["ema21"]
    df["cross_up"] = df["above"] & ~df["above"].shift(1, fill_value=False)
    df["cross_dn"] = ~df["above"] & df["above"].shift(1, fill_value=False)
    return df
```

**`common/data.py`, `load_minute()`**: the one place that knows the minute store's quirks; studies never read those
files directly.

```python
    """...
    Differences from the broker files that this function absorbs so strategy
    code never has to know them:
      * naming/schema -- {SYM}.csv, no "US/Eastern" suffix, no wap/barCount
      # ...
      * session shape -- stored WITH pre/post (a superset, sliced here);
        rth=True keeps 09:30-15:59 ET
      * close basis   -- the 15:59 bar closes BEFORE the 16:00 auction, so
        it can differ from the official daily close (median 0.03%, >0.5% on
        ~2% of sessions). Use the daily files for official closes.
    """
```
