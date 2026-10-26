# SEC filings and the filing clock

Two fetchers pull public SEC EDGAR data for every company in the research panels:

| Script | What it pulls |
|---|---|
| `fetch_edgar.py` | each company's submissions history (every filing's form, acceptance time, 8-K items, industry code) and its cover-page share counts, per CIK (the SEC's company id), resumable |
| `fetch_facts.py` | the SEC's bulk XBRL "company facts" file, reduced to the reported revenue facts, with the first checks on them |

A third piece, the filing-clock rule, is a short block inside a 2,091-line study build. It is the reason this card
exists: it turned up a data error that would have quietly put filings on the wrong trading day.

This folder shows **how the data is fetched, fingerprinted and corrected**. The scripts appear here only as the
excerpts below.

## How it is built

- **Inside the SEC's fair-access rules.** Requests carry a user agent naming the requester (read from the
  environment, never stored in code) and stay under eight a second across six threads through one shared clock.
- **Resumable and atomic.** A file already on disk is the done marker; every write goes to a temp file and is renamed.
- **Every bulk file fingerprinted.** The company-facts zip is saved under its `Last-Modified` date, its md5 goes into
  a manifest with the download time, a short download is refused, and a zip already on disk for that date is reused.
  The same file name therefore always means the same bytes, and a study can record exactly which version it read.

## The filing-clock lesson

EDGAR stamps each filing with an acceptance time in UTC. Converted to New York time, the stored copies of about a
third of the companies read **4–5 hours late**: an earnings release accepted after the close appeared to arrive in
the middle of the night, or on the next day. An event study keyed on those times reacts on the wrong session.

- **First rule (replaced):** treat it as a property of each stored file; a file was "shifted" when at least 10% of its
  core filings read late. Checked against a list of companies with a known clock, it mislabeled them both ways.
- **The rule in use: a property of the company, decided by a vote.** EDGAR's own rules make some readings impossible:
  a time after the filing date, a time outside EDGAR's opening hours, a same-day time after the filing cutoff (insider
  Forms 3/4/5 excepted). For each company, count the impossible readings as stored (A) and again shifted back by the
  offset (B). The company is shifted when A > B; a tie is not shifted.
- **Checked before use:** 2,176 of 6,400 companies come out shifted, and a full run must reproduce those counts; a
  regression list of known companies must keep its labels (Apple and Nvidia shifted, Microsoft clean); a filing stored
  under two companies must get identical corrected times; and any page of filings with enough votes of its own must
  agree with its company's label.
- **The source moved.** A fresh EDGAR download in October 2026 read correctly for the same filings, so the error sits
  in the stored copies, not in today's EDGAR. That is why every file keeps its download date and fingerprint.

## Excerpts

**1. One clock for six threads** (`fetch_edgar.py`): each request books the next free slot under a lock, so the
threads together never exceed the SEC's rate.

```python
def get(url):
    for a in range(4):
        with _lock:
            t = max(time.time(), _last[0] + GAP)
            _last[0] = t
        if t > time.time():
            time.sleep(t - time.time())
        try:
            with urllib.request.urlopen(urllib.request.Request(url, headers=UA), timeout=60) as r:
                return r.read()
        except urllib.error.HTTPError as e:
            if e.code == 404:
                return None
            time.sleep(5 * (a + 1))
        except Exception:
            time.sleep(5 * (a + 1))
    return None
```

**2. A dated, fingerprinted bulk download** (`fetch_facts.py`): named by the server's `Last-Modified` date, hashed
while it streams, refused if short, and logged to a manifest.

```python
def download_zip(raw):
    """GET the bulk file once; reuse a zip already on disk for the same Last-Modified date."""
    raw.mkdir(parents=True, exist_ok=True)
    with urllib.request.urlopen(urllib.request.Request(URL, headers=UA), timeout=120) as r:
        lm = parsedate_to_datetime(r.headers["Last-Modified"]).astimezone(timezone.utc)
        size = int(r.headers["Content-Length"])
        path = raw / f"companyfacts_{lm:%Y-%m-%d}.zip"
        if path.exists() and path.stat().st_size == size:
            say(f"reusing {path.name} ({size:,} bytes, Last-Modified {lm:%Y-%m-%d %H:%M} UTC)")
            return path
        md5, part, t0 = hashlib.md5(), path.with_suffix(".part"), time.time()
        with open(part, "wb") as f:
            while chunk := r.read(1 << 20):
                md5.update(chunk)
                f.write(chunk)
    if part.stat().st_size != size:
        raise RuntimeError(f"short download: {part.stat().st_size:,} of {size:,} bytes")
    part.replace(path)
    man = raw / "manifest.csv"
    new = not man.exists()
    with open(man, "a", newline="") as f:
        w = csv.writer(f)
        if new:
            w.writerow(["file", "url", "bytes", "md5", "last_modified_utc", "downloaded_utc"])
        w.writerow([path.name, URL, size, md5.hexdigest(), f"{lm:%Y-%m-%d %H:%M:%S}",
                    f"{datetime.now(timezone.utc):%Y-%m-%d %H:%M:%S}"])
    say(f"downloaded {path.name}: {size:,} bytes in {time.time() - t0:.0f}s, md5 {md5.hexdigest()}, "
        f"Last-Modified {lm:%Y-%m-%d %H:%M} UTC")
    return path
```

**3. The filing-clock rule** (the study build, `acceptance_time_fix.py` here): what EDGAR's hours forbid, then one
vote per company between the stored reading and the shifted-back one.

```python
A7_EVID = {"10-Q", "10-K", "8-K", "6-K", "3", "4", "5", "FWP"}
A7_FROM = np.datetime64("2003-01-01", "D")
A7_INS = {"3", "4", "5"}
A7_CORE = {"10-Q", "10-K", "8-K"}
A7_SEC = {"EFFECT", "UPLOAD", "LETTER", "QUALIF"}
A7_OPEN, A7_CLOSE, A7_LATE = 6 * 3600, 22 * 3600 + 5 * 60, 18 * 3600
# ...
def et_local(sec):
    """UTC instants (int64 seconds since the epoch; NaT-free) -> (ET local seconds, the New York offset in seconds:
    -14400 EDT / -18000 EST), IANA America/New_York with its historical DST rules."""
    t = pd.DatetimeIndex(sec.astype("datetime64[s]"), tz="UTC").tz_convert(ET)
    loc = t.tz_localize(None).values.astype("datetime64[s]").astype(np.int64)
    return loc, loc - sec


def a7_impossible(loc, fd, ins):
    """A7 (4): an ET reading EDGAR's hours and filing-date rule forbid: (a) its date is after the filing date; (b)
    before 06:00:00 or after 22:05:00; (c) on the filing date after 18:00:00, unless a Form 3 / 4 / 5. loc: ET local
    seconds, fd: filing date (days since the epoch), ins: Forms 3 / 4 / 5."""
    day, tod = np.floor_divide(loc, 86400), np.mod(loc, 86400)
    return (day > fd) | (tod < A7_OPEN) | (tod > A7_CLOSE) | ((day == fd) & (tod > A7_LATE) & ~ins)
# ... (inside parse_edgar, over every filing of every company read)
    ok = ~np.isnat(st_a) & ~np.isnat(fd_a)
    sec = np.where(ok, st_a.astype(np.int64), 0)
    loc0, off0 = et_local(sec)                      # as stored
    loc1, off1 = et_local(sec + off0)               # shifted back: minus the offset in force at the stored instant
    base = form.str.replace(r"/A$", "", regex=True)
    ins = base.isin(A7_INS).values
    evid = (base.isin(A7_EVID) | base.str.startswith("424B")).values & ok & (fd_a >= A7_FROM)
    fdn = fd_a.astype(np.int64)
    imp0 = a7_impossible(loc0, fdn, ins) & evid
    imp1 = a7_impossible(loc1, fdn, ins) & evid
    cu, ci_ = np.unique(cik_a, return_inverse=True)
    A = np.bincount(ci_, imp0, len(cu)).astype(np.int64)
    B = np.bincount(ci_, imp1, len(cu)).astype(np.int64)
    shifted_c = A > B
    sh = shifted_c[ci_]
    loc = np.where(sh, loc1, loc0)
```

## Built with

Python (pandas, numpy, pyarrow), the SEC's public submissions, company-concept and bulk company-facts endpoints.
The clock error was caught by an independent review step of a study, written up as a plan
amendment ([../../6_working_with_ai/PLAN_AMENDMENTS.md](../../6_working_with_ai/PLAN_AMENDMENTS.md), A7) and is now a
rule every study applies ([../../5_research/LESSONS.md](../../5_research/LESSONS.md)).
