# The nightly pipeline

Every weekday night after the close, one unattended Python runner refreshes the data stores for the whole US
universe (~3,000 stocks): it tops up the daily bars, lets the trade radar run its nightly check on the fresh data,
rebuilds the screener workbook, copies it, and captures the rolling 1-minute window. Then it sends **one Telegram
message** saying what each step did. Once a week, longer legs follow with their own message: the 30-minute and
hourly bar stores and the earnings history. Windows Task Scheduler starts it; a full night takes about an hour.

This folder shows **how the chain is built and how it fails safely**. The eight scripts behind it (`nightly.py`,
`yahoo_update.py`, `yahoo_bars.py`, `yahoo_earnings.py`, `broker_conn.py`, `broker_daily.py`, `broker_universe.py`,
`backup_drive.py`, about 2,450 lines) appear here only as the short excerpts below.

## What runs, in order

| Step | Script | What it does |
|---|---|---|
| bars | `yahoo_update.py` | tops up every stock's daily bars, the consolidated-volume files and the SPY/QQQ benchmark to the last settled session |
| radar | the trade radar ([../../1_agents/trade_radar/](../../1_agents/trade_radar/)) | reads the store this run just topped up and sends its own alerts |
| workbook | `toggle_workbook.py` ([../../3_workbook/](../../3_workbook/)) | rebuilds the workbook from the fresh store |
| copy | `nightly.py` | copies the workbook to a synced cloud folder (a main copy and a dated one), compared byte for byte |
| minute | `yahoo_bars.py --interval 1m` | captures the rolling 1-minute window |
| weekly | `yahoo_bars.py --interval 30m / 60m`, `yahoo_earnings.py` | the 30-minute and hourly windows, and every earnings report per stock (consensus, reported, surprise) |

Two data sources, each used for what it does best. The broker's API gives long daily histories but is
rate-limited, so a pass over the whole universe takes hours: `broker_daily.py` is a resumable crawler for full
rebuilds and repairs. Yahoo does the nightly top-up of the whole universe in ~30 minutes;
on settled bars the two sources agreed to the digit in a full-history audit. Prices come from the broker, volume from
Yahoo's consolidated tape.

## How a bad night is caught

- **One message, built from what each script printed.** The scripts return 0 even when some stocks fail, so the
  message quotes each step's own summary line (updated / appended / repaired / flagged / failed), not exit codes.
- **The chain stops at its first failing step**, and the message names the step with the last lines of its log.
  The workbook copy is the exception: a locked or missing folder is a warning line and the chain carries on, because
  a copy can be redone any time while a missed night costs intraday history that cannot be fetched later.
- **Waiting for the close to settle.** For some hours after the close Yahoo serves the just-closed session with a
  blank close, and the top-up rightly drops such bars. The runner compares the session it reached with the one the
  calendar expects and re-runs the bars step every half hour until it lands, or until a deadline, and then says
  in the message which session everything is for. A market holiday looks the same and costs one late, labeled run.
- **A lock check before a long build.** If the workbook is open in Excel, the run fails in a second with "close it"
  instead of after a several-minute rebuild.
- **Rolling windows cannot lapse.** Yahoo serves 30-minute bars only ~60 days back and 1-minute bars ~30 days, so a
  missed stretch loses sessions for good. A marker file holds the date of the last complete weekly run; a missed
  week is caught up by the next run.

## How the store stays clean

- **Never append across an adjustment change.** Before anything is written, a stock's last stored closes must match
  Yahoo's. A split or spin-off moves every close in the overlap; that stock is flagged for a broker re-crawl and
  nothing is touched. The median error is used, not the max, so one odd print cannot re-flag a stock forever.
- **Repair half-formed bars.** A crawl during market hours writes a partial bar dated like a settled day. The last
  two stored sessions are re-checked against Yahoo's settled bar and replaced when they differ.
- **SPY's own dates are the trading calendar.** Yahoo once served a few thin stocks a bar on a market holiday; every
  fetch is now filtered against the sessions SPY printed, a calendar that needs no maintenance.
- **Temp-and-rename for every file**, so a run killed mid-write never leaves a half file that looks finished, and
  every fetcher is resumable: a file whose last bar is the target session is its own done marker.

## The weekly backup

The data stores, and the study and bot code that has no git remote, are copied once a week to cloud storage with
`robocopy` (`backup_drive.py`): copy only, never delete, so a file deleted on the PC survives in the backup; folder
links are not followed; `.env` files and keys are never copied; and Windows device names (a stock whose ticker is
`CON`) are renamed on the way, because the cloud folder refuses them.

## Excerpts

**1. One step of the chain** (`nightly.py`): each script runs as a child process with its output appended live to the
night's log; a failure carries the last lines of that log into the Telegram message.

```python
def run_step(name, args, log_path):
    """Run one project script with its output appended live to the log.
    Returns the lines it printed; raises StepFailed on a non-zero exit or
    a timeout (the child is killed)."""
    t0 = time.monotonic()
    header = f"===== {name}   {datetime.now():%Y-%m-%d %H:%M:%S}"
    with open(log_path, "a", encoding="utf-8") as log:
        log.write(f"\n{header}\n$ python {' '.join(args)}\n")
        log.flush()
        try:
            rc = subprocess.run(
                [sys.executable, *args], cwd=C.ROOT, stdout=log,
                stderr=subprocess.STDOUT, timeout=STEP_TIMEOUT,
                env={**os.environ, "PYTHONIOENCODING": "utf-8"}).returncode
        except subprocess.TimeoutExpired:
            rc = f"timeout after {STEP_TIMEOUT // 3600} h"
        minutes = (time.monotonic() - t0) / 60
        log.write(f"===== {name} exit {rc}   {minutes:.0f} min\n")

    lines = log_path.read_text(encoding="utf-8", errors="replace").splitlines()
    start = max(i for i, ln in enumerate(lines) if ln == header)
    out = lines[start + 2:-1]
    if rc != 0:
        tail = [ln for ln in out if ln.strip()][-8:]
        raise StepFailed(f"{name} (exit {rc}, {minutes:.0f} min)\n"
                         + "\n".join(tail)[-700:])
    return out
```

**2. The settle wait** (`nightly.py`): re-run the top-up until the expected session is there, or give up at a
deadline and say so instead of failing.

```python
def update_bars(log_path, t0):
    """Step 1 with the settle wait. Returns (output lines, target session,
    note or None)."""
    expected = expected_session()
    waits = 0
    while True:
        out = run_step("bars", ["yahoo_update.py"], log_path)
        target = target_session(out)
        if target == "?" or target >= expected:
            note = (f"settled after {waits} wait(s) at {datetime.now():%H:%M}"
                    if waits else None)
            return out, target, note
        if time.monotonic() - t0 > SETTLE_DEADLINE:
            return out, target, (f"⚠️ no settled {expected} bar on Yahoo by "
                         f"{datetime.now():%H:%M} (holiday, or Yahoo late) -- "
                         f"everything below is for {target}")
        waits += 1
        with open(log_path, "a", encoding="utf-8") as log:
            log.write(f"\n===== waiting {SETTLE_WAIT // 60} min: Yahoo target "
                      f"{target} < expected {expected}   (wait {waits})\n")
        time.sleep(SETTLE_WAIT)
```

**3. Fail fast on a locked workbook, and prove the copy** (`nightly.py`): a copy counts only when it matches the
local book byte for byte.

```python
def check_workbook_writable():
    """Fail in a second, not after a 4-minute build, if Excel holds the file."""
    if WORKBOOK.exists():
        try:
            open(WORKBOOK, "r+b").close()
        except PermissionError:
            raise StepFailed(f"workbook: {WORKBOOK.name} is open in Excel on "
                             f"the desktop -- close it") from None


def copy_checked(dst):
    """Copy the local book to dst and compare the two byte for byte. None
    when they match, else why not."""
    try:
        shutil.copyfile(WORKBOOK, dst)
    except PermissionError:
        return "NOT copied -- locked, open in Excel on the desktop? close it"
    except OSError as e:
        return f"NOT copied -- {e.strerror or e}"
    if not filecmp.cmp(WORKBOOK, dst, shallow=False):
        return "copy differs from the local book"
    return None
```

**4. A missed week is caught up** (`nightly.py`): the weekly legs are due on their day, or when the last complete
run is too old, or when there is no record at all, and never twice on one day.

```python
def weekly_due(today):
    # ...
    last = None
    if WEEKLY_MARKER.exists():
        try:
            last = datetime.strptime(
                WEEKLY_MARKER.read_text(encoding="utf-8").strip(), "%Y%m%d").date()
        except ValueError:
            pass
    if last == today:
        return False
    return (today.weekday() == WEEKLY_DAY or last is None
            or (today - last).days >= WEEKLY_OVERDUE)
```

**5. SPY as the trading calendar** (`yahoo_update.py`): SPY prints a bar on every session and never on a closed day,
so its dates filter out phantom bars; a failed SPY fetch stops the run rather than append unverified dates.

```python
def calendar():
    """US sessions as a frozenset of yyyymmdd: SPY's Yahoo dates, fetched
    once per process (the module docstring says why SPY). Holds the live
    bar of a session still in progress too; last_settled_session()'s
    time-of-day cut is what keeps that one out of the target. A session
    whose SPY bar is served blank is left out and goes to _UNSETTLED
    instead."""
    global _CAL, _SPY_ROWS, _UNSETTLED
    if _CAL is None:
        blanks = []
        rows, err = _fetch_raw("SPY", blanks=blanks)
        if err:
            sys.exit(f"SPY calendar fetch failed ({err[0]}: {err[1]}) -- "
                     f"aborting rather than appending unverified dates")
        _SPY_ROWS = rows
        _CAL = frozenset(r[0] for r in rows)
        _UNSETTLED = frozenset(blanks)
        note = (f"   blank on Yahoo: {' '.join(sorted(_UNSETTLED))}"
                if _UNSETTLED else "")
        print(f"calendar: {len(_CAL)} SPY sessions, {min(_CAL)} .. {max(_CAL)}{note}")
    return _CAL
```

**6. The overlap check before appending** (`yahoo_update.py`): too little overlap or a shifted median close flags the
stock for a broker re-crawl and writes nothing.

```python
OVERLAP = 10                                     # stored sessions to verify
REPAIR_TAIL = 2                                  # trailing sessions repairable
TOL = 0.5                                        # % close mismatch -> basis flag
REPAIR_TOL = 0.1                                 # % c/h/l mismatch -> repair
MIN_MATCH = 3                                    # fewer matched dates -> thin
# ...
            # Basis check on the sessions BEFORE the repairable tail -- the
            # tail may be half-formed, which is a repair, not a basis change.
            tail = stored[-REPAIR_TAIL:]
            base = stored[:-REPAIR_TAIL][-(OVERLAP - REPAIR_TAIL):]
            matched = [(c, ymap[d][4]) for _i, d, _h, _l, c in base
                       if d in ymap and c > 0]
            if len(matched) < MIN_MATCH:
                flags.append({"symbol": stem, "reason": "thin_overlap",
                              "detail": f"{len(matched)} of last {len(base)} "
                                        f"stored sessions found on Yahoo"})
            else:
                # A basis change (split ratio) shifts EVERY overlap close;
                # ...
                # Median, not max, so one odd day can't re-flag forever.
                errs = sorted(abs(y - c) / c * 100 for c, y in matched)
                med = errs[len(errs) // 2]
                if med > TOL:
                    flags.append({"symbol": stem, "reason": "basis_change",
                                  "detail": f"median close mismatch "
                                            f"{med:.1f}% in last "
                                            f"{len(matched)} sessions"})
                else:
                    # ... repair the tail, then append the new sessions
```

**7. The weekly backup copy** (`backup_drive.py`): copy-only flags, no folder links, and keys never sent.

```python
    for src, dst, xd, xf in JOBS:
        # /E with no /PURGE keeps what the PC deleted; /XJ never follows a folder
        # link. Never copied: keys and .env files, Excel lock stubs, bytecode,
        # this run's own open log, the job's own skips -- and device names, done
        # below.
        worst = max(worst, subprocess.run(
            ["robocopy", str(src), str(dst), "/E", "/XJ", "/FFT", "/R:2", "/W:5",
             "/XD", "__pycache__", *xd,
             "/XF", ".env", "*.env", "*.key", "~$*", LOG.name, *xf,
             *(f"{n}.*" for n in sorted(RESERVED)),
             "/NP", "/NDL", "/NFL", f"/UNILOG+:{LOG}"],
            creationflags=subprocess.CREATE_NO_WINDOW).returncode)
        copy_reserved(src, dst, xd)
    return 1 if worst >= 8 else 0          # robocopy: 0-7 done, 8+ failed
```

## Built with

Python (the fetchers use the standard library only; the broker crawler uses the broker's own API client), Windows Task Scheduler,
`robocopy` and the Telegram Bot API for the one-message report. Each failure
mode above was met in a real run first and is named, with its date, in the scripts' own docstrings.
