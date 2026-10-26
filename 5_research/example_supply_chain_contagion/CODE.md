# Supply-chain contagion: the code, in excerpts

The study's code (about 1,600 lines across six scripts) is not published. Its working name was `chain_canary`, which
is why that name appears in the excerpts. These excerpts show how the plan in
[README.md](README.md) is enforced in code: thresholds frozen in one file and checked by hash, windows that only look
back, controls built so they can say no, standard errors clustered by month or episode, and the verdict gates written
as code before any outcome was read. Each excerpt is verbatim; `# ...` marks lines left out.

**1. Every threshold in one pre-registered file** (`chains.py`): written down before any forward return existed;
the sensitivities in the README move these numbers one at a time.

```python
# breadth = share of a group's eligible names closing over their EMA
EMA_FAST, EMA_SLOW = 21, 50
LINK_MIN = 6          # a link needs >= 6 eligible names on t
LINK_RHO = 0.30       # ... and a residual correlation >= 0.30 with the rest of its chain
RHO_WIN, RHO_MIN = 250, 120   # over the 250 sessions before t, >= 120 paired days
CHAIN_MIN = 3         # an event needs >= 3 valid links (the rest of the chain >= 2)
BREAK = 0.10          # broken link: <= 10% of its names over their 21
FRESH = 0.50          # ... after >= 50% over their 21 on some session in the prior FRESH_WIN
FRESH_WIN, FRESH_MINP = 20, 15    # ... with >= 15 size-valid sessions in that window
HOLD = 0.40           # the rest of the chain: median link share over the 21 >= 40%
COOL = 20             # one event per (chain, link) per 20 sessions
WASH = 0.15           # washout: every valid link <= 15% over its 21
# ...
MATCH_WIN, N_TILE = 20, 3         # baseline matched on the core's trailing 20-session excess tercile
CELL_MIN = 20         # a baseline cell needs >= 20 control days, else all years (counted)
HORIZONS = (5, 10, 20)
PRIMARY_K = 10
```

**2. The analysis refuses to run on changed definitions** (`analyze.py`): the README records the SHA-256 of the two
definition files, so a quiet edit after seeing results stops the run instead of moving the answer.

```python
def check_hashes():
    """Refuse to run if chains.py / events.py differ from the README's recorded SHA-256."""
    txt = (HERE / "README.md").read_text(encoding="utf-8")
    for f in ("chains.py", "events.py"):
        h = hashlib.sha256((HERE / f).read_bytes()).hexdigest()
        if h not in txt:
            raise SystemExit(f"{f} sha256 {h} is not the one recorded in README.md -- the "
                             "pre-registered definitions changed")
```

**3. Windows that end before the event day** (`events.py`, `lib.py`): every trailing statistic is shifted one
session, so the day being scored never feeds its own definition.

```python
def prior_max(x, valid, w=K.FRESH_WIN, minp=K.FRESH_MINP):
    s = pd.Series(np.where(valid, x, np.nan))
    return s.shift(1).rolling(w, min_periods=minp).max().to_numpy()
# ...
def roll_beta(y, x, win, minp):
    """beta of y on x over the win sessions before t (t excluded), jointly masked."""
    m = np.isfinite(y) & np.isfinite(x)
    ys = pd.Series(np.where(m, y, np.nan)).shift(1)
    xs = pd.Series(np.where(m, x, np.nan)).shift(1)
    cov = ys.rolling(win, min_periods=minp).cov(xs)
    var = xs.rolling(win, min_periods=minp).var()
    return (cov / var).to_numpy()
```

```python
def trailing_std(R, win, minp):
    """Rolling stdev over the win sessions BEFORE t (t excluded), column-wise."""
    return pd.DataFrame(R).shift(1).rolling(win, min_periods=minp).std().to_numpy()
```

**4. The event and its control days, from the same definitions** (`events.py`): a control day is a day when the
chain holds and the excluded link is valid but not broken, so event and control differ only in the break itself.

```python
    def canary(self, brk=K.BREAK, hold=K.HOLD, per_link=True):
        """(events [(t, broken link names)], qualifying mask)."""
        B, V = self.B, self.V
        broken = V & (B <= brk) & self.fresh_dn
        stale = V & (B <= brk) & ~self.fresh_dn
        rest = V & (B > brk)
        q = (self.up & (self.nv >= K.CHAIN_MIN) & broken.any(1) & ~stale.any(1)
             & (rest.sum(1) >= 2) & (masked_median(B, rest) >= hold))
        # ...
    def control(self, excl, brk=K.BREAK, hold=K.HOLD, floor=None):
        """C control days for the excluded link set."""
        B, V = self.B, self.V
        ex = self.ix(excl)
        others = V & ~ex[None, :]
        m = (self.up & V[:, ex].all(1) & (others.sum(1) >= 2) & (self.nv >= K.CHAIN_MIN)
             & ~(V & (B <= brk)).any(1) & (masked_median(B, others) >= hold))
        if floor is not None:
            m &= ~(V & (B < floor)).any(1)
        return m
```

**5. One event set for every horizon** (`analyze.py`): events too late for the longest horizon are dropped for all
horizons, so the 5-, 10- and 20-session answers describe the same events.

```python
    last_t = T - 1 - max(K.HORIZONS)          # one event set for every horizon
    # ...
    CH, ev, qual = E.build(d, br)
    ev = ev[ev["t"] <= last_t].reset_index(drop=True)
```

**6. The control-day exclusion, corrected** (`analyze.py`): the pre-registered rule dropped control days with an
event in [s−k, s+k]. Its forward half keeps only control days whose own outcome window holds no break, which is
look-ahead, and it alone faked a −74 to −255 bp effect. The verification caught it; the corrected rule looks back only.

```python
EXCL = "past"     # Amendment 2: "past" = [s-k, s] (corrected); "both" = [s-k, s+k] (as run)


def dilate(q, k):
    """True on control day s if a qualifying day lies in the exclusion window. "both" was the
    pre-registered [s-k, s+k]; its forward half keeps only control days whose own outcome window
    holds no break, a look-ahead selection (verification, 2026-10-23). "past" = [s-k, s]."""
    q = q.astype(float)
    if EXCL == "both":
        return np.convolve(q, np.ones(2 * k + 1), mode="same") > 0
    return np.convolve(q, np.ones(k + 1))[:len(q)] > 0
# ...
    days_ok = ctrl & ~dilate(qual, k) & np.isfinite(ser["raw"]) & np.isfinite(ser["ex"])
    days_ok[T - k:] = False
```

**7. A matched baseline** (`analyze.py`): each event is compared with control days from the same year and the same
tercile of the chain's trailing 20-session excess return; thin cells fall back to all years, and the fallback rate is
reported.

```python
        # matched cell: same year and same tercile of the trailing excess, else all years
        if match and edges is not None and np.isfinite(pre[t]):
            tile = int(np.digitize(pre[t], edges))
            st = cdays[(yrs[cdays] == yr) & (tile_c == tile)]
            if len(st) >= K.CELL_MIN:
                cm, fb_m = bank.cell((arm, chain, extra_key, yr, tile), st, ser), False
            else:
                at = cdays[tile_c == tile]
                cm, fb_m = bank.cell((arm, chain, extra_key, "all", tile), at, ser), True
        else:
            cm, fb_m, tile = cu, True, -1
```

**8. Standard errors clustered by month or episode** (`analyze.py`): events on nearby days share one market, so the
error is computed over clusters (calendar months for the contagion test, weak-market episodes for the washouts), and the
bootstrap resamples whole clusters and re-estimates the baselines in every draw.

```python
def cr1(x, cl):
    x, cl = np.asarray(x, float), np.asarray(cl)
    good = np.isfinite(x)
    x, cl = x[good], cl[good]
    n = len(x)
    if n < 2:
        return np.nan, np.nan, np.nan, 0
    m = x.mean()
    s = pd.Series(x - m).groupby(cl).sum().to_numpy()
    G = len(s)
    if G < 2:
        return m, np.nan, np.nan, G
    se = math.sqrt(G / (G - 1) * (s ** 2).sum()) / n
    return m, se, m / se if se > 0 else np.nan, G
```

```python
def boot(bank, stats, draws=K.BOOT, seed=K.SEED):
    """{name: (observed, se, z)} with clusters resampled and baselines re-estimated per draw."""
    rng = np.random.default_rng(seed)
    obs = {n: f(None) for n, f in stats.items()}
    acc = {n: [] for n in stats}
    for _ in range(draws):
        W = np.bincount(rng.integers(0, bank.G, bank.G), minlength=bank.G).astype(float)
        for n, f in stats.items():
            acc[n].append(f(W))
    # ...
```

**9. The verdict as code** (`analyze.py`): every gate from the README's verdict rules is a boolean, printed PASS or
fail, and most of them ask both halves of the data to agree.

```python
    C_o, _, C_z = bs["C"]
    g = {
        "H1(i) C d_ex < 0, boot z <= -2.5, both halves < 0":
            C_o < 0 and C_z <= -2.5 and bs["C_h1"][0] < 0 and bs["C_h2"][0] < 0,
        "H1(ii) family TV-label d_ex < 0, boot z <= -2.0, both halves < 0":
            bt["F"][0] < 0 and bt["F"][2] <= -2.0 and bt["F_h1"][0] < 0 and bt["F_h2"][0] < 0,
        "H1(iii) C - P (chain-reweighted) boot z <= -2.0": bs["C-P"][2] <= -2.0,
        "H1(iv) C - same-date bystanders boot z <= -2.0": bs["C-B"][2] <= -2.0,
        "H1(v) k5 and k20 same sign as k10": np.sign(hz[5]) == np.sign(hz[10]) == np.sign(hz[20]),
        "H1(vi) leave-one-year-out keeps the sign": all(np.sign(v) == np.sign(C_o) for v in loy.values()),
        # ...
    }
    for n, v in g.items():
        say(f"  [{'PASS' if v else 'fail'}] {n}")
        gates.append({"gate": n, "pass": bool(v)})
```

**10. The window that inspired the idea is never voted** (`analyze.py`, `replay.py`): the recent weeks that
suggested the hypothesis are replayed for description only, on a separate data layer.

```python
"""chain_canary analysis -- outcomes, matched baselines, placebo / bystander contrasts, the
bootstrap, gates and sensitivities on the us11 layer (README "Rules", "Amendment 1"). The live
layer is replay.py's job and is never voted.
```

```python
"""chain_canary live replay -- the AI compute capex chain day by day, Aug -> Oct 2026, on the live
layer, and every C / W event the rules fire there since 2026-08-03 with whatever forward window
exists. Descriptive only: this is the window that generated the hypothesis (README "Rules").
```

The independent check is not code in this folder: a separate AI agent rebuilt the study from the README alone,
without reading `analyze.py` or `events.py`, and reproduced every headline number. A code review and a post-hoc audit
ran alongside it; the exclusion flaw in excerpt 6 is what they found (README, "Amendment 2").
