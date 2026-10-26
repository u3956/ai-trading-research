# Earnings research pack: one company, one quarter, start to finish

The plan: take one company's last four quarters and work them end to end with AI agents under checks. Reader agents
fill fixed tables from the filings, the press release and the call, with every number cited back to its source. A
script and a second agent check those numbers, and a person checks a sample by hand. A driver model is rolled forward
one quarter, and a one-page note comes out at the end.

The company is **Zscaler (ZS)**, whose fiscal year ends on July 31. It was chosen because it has four quarters of
10-Q/10-K filings on EDGAR and its next report is due on Nov 24, 2026, after the close. That means a forecast can be
frozen in git before the report and scored after it. This is research practice, not investment advice.

**Status:** steps 1 (the sources) and 2 (the readers) are done, and the script part of step 3 has run. The hand
check, the checker agent and steps 4–5 are next, and this page grows with them.

## Steps

| # | Step | Status |
|---|---|---|
| 1 | **Sources.** Every filing, release and call for the four quarters, fetched once and kept as fetched | done, Oct 25, 2026 |
| 2 | **Readers.** One agent each for the filings, the release and the call. Fixed tables; every number carries its file, line and exact quote | done, Oct 26, 2026 |
| 3 | **Checks.** Filing numbers against the filing's own XBRL tags (a script, so the check is deterministic), a checker agent on the text, 30 numbers by hand, and an errors page | script done, Oct 26, 2026; hand check and checker agent next |
| 4 | **Model.** Built on FY26 Q3 and rolled forward to Q4 as an honest test, then a dated FY27 Q1 forecast committed before the report | planned |
| 5 | **Note, one page.** What changed, the model against the street, and what would change the view | planned |

## The sources (described here, not published)

None of these files is in this repo. They belong to the company, the SEC or a data vendor, and the official call
transcripts are licensed for local use and short quotes only. The repo shows how they were collected and what was
found in them.

| Source | What was kept |
|---|---|
| SEC EDGAR | every 10-Q, 10-K and 8-K since Nov 2025: the index page, the main document, the press-release exhibit, and the XBRL instance with its labels; plus dated snapshots of the company's filing list and of all its XBRL facts |
| The company's investor-relations page | the official call transcripts for all four quarters, the shareholder letters, the supplemental financials, the "modeling considerations" sheet and the investor-day deck |
| The Q4 call and the investor day | the audio of each, transcribed locally with Whisper (large-v3, on the GPU) and timestamped; a check on the official text, not a replacement |
| Analyst consensus | a dated snapshot of Yahoo's estimates (revenue and EPS by quarter and year, estimate history, ratings) |
| The screener's own stores | the ticker's daily bars and earnings history, copied with a date |

| Quarter | Ends | Release filed (8-K, EX-99.1) | Report |
|---|---|---|---|
| FY26 Q1 | 2025-10-31 | 2025-11-25 16:07 ET | 10-Q |
| FY26 Q2 | 2026-01-31 | 2026-02-26 16:07 ET | 10-Q |
| FY26 Q3 | 2026-04-30 | 2026-05-26 16:08 ET | 10-Q |
| FY26 Q4 | 2026-07-31 | 2026-09-03 16:08 ET | 10-K |

**How the sources are kept honest.** Every file goes into a manifest with its URL, its fetch time in UTC, its size
and its md5. A filing folder that already exists is never fetched again, so a source stays exactly as it was first
read. The filing list and the XBRL facts are new dated files on each run, so a change on the SEC's side shows up as a
difference between two dated files instead of silently replacing the old one.

## Found during step 1

These are the first entries for the errors page, all found before any model existed:

1. **Two downloads of the same SEC data disagreed.** A fresh EDGAR download gives the correct acceptance time for
   ZS's Sep 3 release (20:08 UTC = 16:08 ET). Copies of the same filing list taken on Oct 12 read four hours later. The
   same held for PLTR, NVDA and AAPL, while MSFT was right in both copies. A source can change after you read it,
   which is why every file here carries its fetch time and md5. The pipelines card has the full story and the
   per-company fix ([../2_pipelines/edgar/](../2_pipelines/edgar/README.md)).
2. **A web search summary called the ARR guidance "revenue".** It gave FY27 revenue guidance as $4.396–4.426B. The
   release says that range is ARR; revenue guidance is $3.908–3.938B. Every number in this pack is cited to the
   primary document for exactly this reason.
3. **Headline growth includes an acquisition.** Red Canary closed in FY26 Q1. Revenue and ARR grew 25–26%, but
   without Red Canary ARR grew 21%, 21% and 20% in Q2, Q3 and Q4, and FY27 is guided to about 17%. An earlier
   "steady 25% grower" reading was wrong, and only the filings showed it.
4. **The street sits at the top of guidance.** Yahoo, Oct 25: FY27 Q1 revenue $938.2M (42 analysts) against
   $935–939M guided; FY27 EPS $4.91 against $4.86–4.90.
5. **The company publishes its own transcripts.** A web search found only patchy third-party copies. The investor-
   relations page has the official transcript for every quarter, plus shareholder letters and supplementals that are
   not on EDGAR. The local Whisper transcript of the Q4 call lines up with about 80% of the official words before
   the PDF's page headers and disclaimers are removed. Step 3 measures that properly.

## Step 2: the readers

- **Every source becomes line-numbered text first.** A script turns each filing, release, transcript and deck into
  plain text with one line per table row or paragraph. A citation is then a file, a line number and an exact quote,
  which a script can find again.
- **One fixed schema for every table.** The columns, the allowed values and the citation rules are written once
  (excerpt 5), and every reader works to them. A number that has to be computed may only appear in a table marked
  *derived*, which points to the rows it uses.
- **One agent per source, each with its own brief.** The filings, the release and the call each get a reader. Each
  hands back its CSV tables, a notes file (what it could not find, its judgment calls, anything in the sources that
  looked wrong) and a report. The briefs are kept exactly as sent, as the record of what each agent was told.
- **Nothing is corrected before the check.** The tables pass a shape check (headers, ids, allowed values,
  citations filled in), but their truth is step 3's job, so step 3's error count is the readers' own. Where a reader
  departed from its brief, the departure is written down and left in place.

| Reader | Tables (rows) |
|---|---|
| filings | facts 532, changes 60 |
| release | results 127, guidance 69, bridge 164, consensus 36, compare 26 (derived) |
| call | statements 720, Q&A 81 (43 answered directly, 33 in part, 5 dodged), wording 43 |

**An A/B on the release reader.** Two other models redid the release reader from the same brief, and a script scored
all three. All three were clean on every citation and on the GAAP figures against XBRL, and none disagreed beyond
rounding. One of the two was the most complete on the financials (231 result rows against 127), but both missed
the 12 product metrics in the letters' prose that the original reader picked up without being asked. So the brief
for the next quarter names every product and customer metric, the year-to-date columns, one vocabulary for scale,
and the known traps in this company's documents.

## Found during step 2

1. **The local transcription misheard a number.** The investor-day transcript gives the market size as "$20
   billion" where the deck shows $220B. The call reader caught it against the slides.
2. **The official transcript can be wrong too.** The Q4 text credits the CFO with an answer to a question put to the
   CEO, and has "legacy fiber-based SaaS" for what the deck calls a firewall-based SASE displacement.
3. **The company's own documents have errors.** EPS guidance is printed as "$3.78 to $3.82 million" (Q2 release)
   and "$3.99 to $4.02 million" (Q3). The Q2 supplemental's year-to-date columns sum the wrong quarters, with `#REF!`
   cells and two different share counts for one quarter.
4. **A metric changed its definition without a bridge.** AI Security ARR "passed $400M" on the Q1 call; the investor
   day reports Security for AI at about $70M under new categories, and nothing reconciles the two.
5. **The screener's own store mixes rounded and unrounded estimates.** EPS estimates are stored to two decimals, but
   the surprise percentage comes from Yahoo's unrounded figures (12.56% shown, 12.2% from the stored ones).

## Step 3 so far: the citation check

A script re-reads every citation, confirms each value is inside its quote, checks every filed figure against the
XBRL of the 10-Qs and the 10-K, and re-derives the derived table. For the three readers:

| Check | Result |
|---|---|
| Quote found exactly on its cited line | 1,983 of 1,983 |
| Value inside its quote | 1,452 of 1,452 |
| Filed figures matching XBRL / contradicted by it | 562 / 0 |

The A/B's release reader quoted 540 of 540 lines exactly but had 25 values outside their quotes: dash cells written
as 0. Three filings rows date an acquisition's adjusted goodwill to the deal, where the filing tags it at the quarter
end, and twelve rows have no XBRL tag for their period.

A script cannot see the right column, item or basis inside a line that holds several numbers, nor completeness, nor
the judgment calls. The next two checks cover those: 30 numbers the script could not confirm, drawn with a fixed seed
and checked by hand against the original pages, and a checker agent on the text.

## Excerpts

**1. Every file is recorded the moment it is written** (`fetch.py`): one function writes, and the same call adds the
manifest row, so no file can enter the pack without its URL, fetch time and md5.

```python
def add_to_manifest(base, path, url, data=None):
    """Record one file of the pack; data is its bytes (read from disk when None)."""
    data = path.read_bytes() if data is None else data
    m = base / "manifest.csv"
    new = not m.exists()
    with open(m, "a", newline="", encoding="utf-8") as f:
        w = csv.writer(f)
        if new:
            w.writerow(MANIFEST_COLS)
        w.writerow([path.relative_to(base).as_posix(), url, now_utc(), len(data),
                    hashlib.md5(data).hexdigest()])


def save(base, rel, url, data):
    p = base / rel
    p.parent.mkdir(parents=True, exist_ok=True)
    p.write_bytes(data)
    add_to_manifest(base, p, url, data)
    return p
```

**2. Fiscal quarters, and which quarter an earnings release belongs to** (`fetch.py`): a July fiscal year puts
calendar April in Q3. An earnings 8-K reports the quarter of the 10-Q or 10-K that follows it within ten days.

```python
def fiscal_q(period, fy_end):
    """'FY26Q3' for a period ending 2026-04-30 when the fiscal year ends in July."""
    d = date.fromisoformat(period)
    fy = d.year + (d.month > fy_end)
    return f"FY{fy % 100:02d}Q{(d.month - fy_end - 1) % 12 // 3 + 1}"

# ...
    for x in rows:                           # an earnings 8-K reports the quarter of the next 10-Q/10-K
        if x["form"] == "8-K" and "2.02" in x["items"]:
            d = date.fromisoformat(x["filed"])
            nxt = [y for y in rows if y["form"] != "8-K"
                   and d <= date.fromisoformat(y["filed"]) <= d + timedelta(days=10)]
            x["fiscal_q"] = min(nxt, key=lambda y: y["filed"])["fiscal_q"] if nxt else ""
```

**3. Sources stay as first read** (`fetch.py`): a filing already on disk is skipped, never overwritten.

```python
    for x in sorted(rows, key=lambda y: y["accepted_utc"]):
        x["folder"] = f"sec/{x['filed']}_{x['form']}_{x['accession']}"
        if (base / x["folder"]).exists():
            print("kept   ", x["folder"])
            continue
        page, page_url, docs = filing_docs(cik, x["accession"])
        save(base, f"{x['folder']}/index.htm", page_url, page)
        for typ, name, doc_url in docs:
            if typ == x["form"] or typ.startswith(KEEP):
                save(base, f"{x['folder']}/{name}", doc_url, get(doc_url))
```

**4. The call through a local Whisper** (`calls.py`, the command shortened): the company's own spellings are passed
in as a names list, so product and acquisition names come out right. A lock file keeps two transcriptions off the
GPU at once, and the transcript keeps a timestamp on every line, which the official text lacks.

```python
NAMES = {"ZS": "Zscaler, ..., Red Canary, SPLX, Zero Trust Exchange, "
               "Zero Trust Everywhere, ZDX, ARR"}       # as spelled in the company's own releases
# ...
    if LOCK.exists():
        sys.exit(f"GPU lock held ({LOCK}): another transcription is running; try again later")
    names.write_text(NAMES.get(sym, ""), encoding="utf-8")
    LOCK.parent.mkdir(parents=True, exist_ok=True)
    LOCK.write_text(str(os.getpid()))
    try:
        subprocess.run(WSL + [...transcribe_wsl.py, audio, tsv, names, "en", "0.5"], check=True)
    finally:
        LOCK.unlink(missing_ok=True)
    add_to_manifest(base, tsv, "whisper large-v3 of " + audio.relative_to(base).as_posix())

    lines = tsv.read_text(encoding="utf-8").splitlines()
    head = dict(x.split("\t", 1) for x in lines[:2])
    segs = [x.split("\t", 2) for x in lines[2:] if x.count("\t") >= 2]
    txt.write_text(f"{sym} {quarter} | {head.get('label', '')} | language {head.get('language', '')}\n\n"
                   + "\n".join(f"{stamp(a)} {t.strip()}" for a, _, t in segs) + "\n", encoding="utf-8")
```

**5. The citation rules every reader works to** (`readers/SCHEMA.md`, `…` marks a cut): written so that a script
can check every number without judgment.

```markdown
- **Read only `text/ZS/`**, the line-for-line text of the sources (`extract.py` built it from
  `sources/ZS/`). Do not open the XBRL files: they are the checker's.
- **Every value is cited.** Three columns say where it came from:
  - `text`: the file's path under `text/ZS/`, …
  - `line`: the 1-based line number in that file
  - `quote`: an exact substring of that line, copied character for character (no edits, no "..."), at most
    200 characters, containing the value as printed there.
…
- **Never cite a quote that does not contain the value.** If a number has to be computed, it belongs only in
  a table marked *derived*, which points to the rows it uses by `id`.
- **Missing is fine; guessing is not.** List anything not found in `notes.md`.
```

**6. Checking a citation** (`checks/check.py`): the quote must sit on its cited line exactly as written, and the value
must be one of the numbers inside the quote. A miss says why: no file, no such line, a spacing difference, the
quote found on another line, or nowhere at all.

```python
def in_quote(v, quote):
    return any(abs(abs(v) - float(m.replace(",", ""))) < 1e-9 * max(1.0, abs(v)) for m in NUM.findall(quote))

# ...
def cite(L, ln, q):
    """(exact, where) for one citation."""
    line = L[ln - 1] if L and 1 <= ln <= len(L) else None
    if line is not None and q and q in line:
        return True, ""
    if L is None:
        return False, "no file"
    if line is None:
        return False, "no line"
    squash = lambda s: re.sub(r"\s+", " ", s).strip()
    if squash(q) and squash(q) in squash(line):
        return False, "spacing"
    return False, "other line" if any(q in x for x in L) else "nowhere"
```

## Built with

Python's standard library only (urllib, csv, hashlib, zoneinfo), the SEC's EDGAR and XBRL APIs, Whisper large-v3 in
WSL on the local GPU, AI agents as the readers, and an AI coding agent running the steps one at a time, each on the
owner's go-ahead.
