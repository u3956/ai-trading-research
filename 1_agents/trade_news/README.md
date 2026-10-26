# Trade news: filtered news on a watch list, judged by an AI

A small agent that watches the news on the stocks in a watch list and sends only what could move them to the owner's
Telegram chat. It runs every minute during US trading hours (pre-market through after-hours) and every 15 minutes
otherwise. It also answers three questions asked through the radar bot: "why TICKER" (why is it moving today),
"TICKER analysts" and "TICKER holders".

This folder shows **how it is built and checked**. The program (`news.py`, about 430 lines of standard-library
Python, no packages) appears here only as the excerpts below. The owner's watch list is not published.

## What it does, every minute

1. **Asks every source at once**, in parallel threads:
   - public RSS feeds per ticker (Yahoo Finance and Nasdaq headlines; for a site with no feed of its own, a Google News
     search limited to that site), listed one per line in a plain text file;
   - the company's latest SEC filings on EDGAR (the ticker-to-CIK map is looked up once and kept);
   - the Nasdaq trading-halt feed;
   - analyst rating and target changes, and the next earnings date.
2. **Skips what it has already seen.** Every item gets a stable id, kept in a state file, so nothing is sent twice.
   The very first run records everything silently, and items more than two days old are recorded, never sent.
3. **Sorts each item by kind:**
   - **push**, sent at once: offerings and shelf registrations, 8-Ks with material items, activist stakes, halts,
     analyst changes;
   - **brief**, kept for a morning brief before the open: insider trades (Form 4), planned sales (Form 144), 5% holders
     (13G), routine 8-K items (vote results, bylaws, exhibits);
   - **headline**: it must name the company and pass a noise filter ("stocks to buy", "should you", roundups,
     previews), then it goes to the AI judge.
4. **The AI judge** keeps only headlines that report a new fact that could move that stock, writes one plain line for
   each, and drops anything that repeats a story already sent.
5. **Sends, logs, saves**, then prunes its memory (ids after 14 days, the sent log after 7).

## How it is built

- **Standard library only**, one file, run by the scheduler with "never two at once". The same file moves to a Linux
  server unchanged: one cron line with `flock`.
- **A broken source costs only that source.** Each fetch runs inside a wrapper that turns any error into "nothing from
  this source this minute"; the rest keeps working.
- **The AI is boxed in.** The same locked-down CLI call as the radar: read-only sandbox, no web search, no tools, a
  JSON schema for the answer. The judge only ever sees numbered headlines and answers with numbers, and an index
  outside the list is ignored.
- **A failed AI call loses nothing.** Headlines waiting for the judge are marked as seen only after the judge has
  answered, so if the call fails they are judged again the next minute.
- **Polite to the SEC.** A contact line in the User-Agent (from `.env`) and a pause between requests, under the SEC's
  ten-requests-a-second limit.
- **Atomic state.** The state file is written to a temporary file and swapped in.
- A run takes about 14 seconds, mostly waiting on the websites. The AI starts only when a new headline passes the
  filters.

## How it is checked

- `python news.py why TICKER` (and `analysts`, `holders`) prints the answer in the terminal and sends nothing.
- `python -m py_compile news.py` after every edit: a syntax error stops the minute job, while a wrong feed address only
  silences that feed.

## Excerpts

**1. The judge's instructions and its answer format** (`news.py`): plain English, with the stories already sent passed
in so repeats are dropped.

```python
JUDGE = """You filter stock news for a trader. Below are new headlines, each tagged with the stock it was found for.
Keep a headline only if it reports a fact from the last day or so that could move THAT stock: results or guidance, an
offering or other share sale, a merger, acquisition or investment, a major contract, customer or partnership, an analyst
rating or price-target change, a regulatory, legal or government action, a management change, an index change, a
short-seller report, a big holder buying or selling, a product or technology event the market will react to, or a sector
or market event that is moving that stock sharply today (say what the event is and the move).
Drop opinion and valuation pieces, old results retold, "should you buy", comparisons, previews, roundups, price-move
recaps that give no cause, and articles mainly about other companies. If several headlines tell the same story, keep
only the clearest one. Also drop anything that repeats one of these stories already sent:
{sent}
For each headline you keep, write one plain line (at most 15 words) with the key fact and number.
Headlines:
{items}"""
# ...
JUDGE_SCHEMA = {"type": "object", "additionalProperties": False, "required": ["keep"], "properties": {"keep": {
    "type": "array", "items": {"type": "object", "additionalProperties": False, "required": ["n", "line"],
                               "properties": {"n": {"type": "integer"}, "line": {"type": "string"}}}}}}
```

**2. "Why is it moving?"** (`news.py`): the answer may use only the items gathered for it, and must say so when they
do not explain the move.

```python
WHY = """A trader asks: why is {sym} moving? Today: {move}.
Using only the items below (newest first, New York times), answer in at most 4 short lines: what is new and when,
whether it is company news or a sector or market move, and the most likely driver. If nothing below explains the move,
say so plainly. No advice.
Items:
{items}"""
```

**3. The AI call** (`news.py`): the schema is written to a temporary folder, the CLI runs locked down there, and the
answer is read back as JSON.

```python
def codex(prompt, schema):
    exe = ENV.get("CODEX") or shutil.which("codex.cmd") or shutil.which("codex") or "codex"   # CODEX: see .env
    with tempfile.TemporaryDirectory() as d:
        Path(d, "schema.json").write_text(json.dumps(schema), encoding="utf-8")
        cmd = [exe, "exec", *LOCKDOWN, "--model", MODEL, "-c", f"model_reasoning_effort={EFFORT}", "--sandbox",
               "read-only", "--skip-git-repo-check", "--ephemeral", "--color", "never", "--output-schema",
               str(Path(d, "schema.json")), "--output-last-message", str(Path(d, "answer.json")), "-"]
        subprocess.run((["cmd", "/c"] if exe.lower().endswith(".cmd") else []) + cmd, input=prompt, text=True,
                       encoding="utf-8", capture_output=True, cwd=d, timeout=300, check=True,
                       creationflags=getattr(subprocess, "CREATE_NO_WINDOW", 0))
        return json.loads(Path(d, "answer.json").read_text(encoding="utf-8"))
```

**4. Every source at once, each one fenced off** (`news.py`):

```python
def safe(f, *a):
    try:
        return f(*a)
    except Exception:
        return []
# ...
def collect(st, syms, ratings):
    with ThreadPoolExecutor(8) as ex:
        jobs = [ex.submit(safe, feed, n, u, s) for n, u in feeds() for s in syms]
        jobs += [ex.submit(safe, sec_all, st, syms), ex.submit(safe, halts, set(syms))]
        jobs += [ex.submit(safe, yahoo, st, syms)] if ratings else []
        # ...
        return [i for j in jobs for i in j.result()]
```

**5. SEC filings sorted into "now" and "morning brief"** (`news.py`): an 8-K is pushed only when it carries an item
that matters; vote results, bylaws and exhibits wait for the brief.

```python
def sec(sym, cik):
    url = (f"https://www.sec.gov/cgi-bin/browse-edgar?action=getcompany&CIK={cik}&type=&dateb=&owner=include"
           "&count=10&output=atom")
    out = []
    for e in plain(get(url, SEC)).iter("entry"):
        form, nums = e.findtext("content/filing-type") or "", re.findall(r"\d\.\d\d", e.findtext("content/items-desc") or "")
        label = next((v for k, v in PUSH_FORMS.items() if form.startswith(k)), None)
        if label is not None:
            kind = "push" if not form.startswith("8-K") or set(nums) - QUIET_ITEMS else "brief"
            words = ", ".join(ITEMS[n] for n in nums if n in ITEMS) if form.startswith("8-K") else label
        elif form.split("/")[0] in ("4", "144", "10-Q", "10-K", "20-F") or form.startswith(("SC 13G", "SCHEDULE 13G")):
            kind, words = "brief", ""
        else:
            continue
        out.append(item(f"sec|{e.findtext('content/accession-number')}", sym, "sec", when(e.findtext("updated")),
                        f"{form} {words}".strip(), e.findtext("content/filing-href") or "", kind))
    return out
```

**6. The minute run's core** (`news.py`): new items only, fresh ones only; push items go out at once; headlines go to
the judge, and only the indexes it returns that exist in the list are used. The headlines' ids are recorded only after
the judge has answered.

```python
    new = list({i["id"]: i for i in collect(st, syms, first or ny.minute % 5 == 0) if i["id"] not in st["ids"]}.values())
    fresh = [] if first else [i for i in new if datetime.fromisoformat(i["pub"]) >= now - timedelta(days=2)]
    cands = [i for i in fresh if i["kind"] == "head" and named(st, i["sym"], i["title"]) and not NOISE.search(i["title"])]
    for i in fresh:
        if i["kind"] != "head":
            note(st, i, i["title"], i["kind"] == "push", stamp)
    send([show(i, i["title"]) for i in fresh if i["kind"] == "push"])
    st["ids"].update({i["id"]: stamp for i in new if i not in cands})
    st["brief"] = ny.date().isoformat() if first else st["brief"]
    save(st)
    if cands:
        sent = [f"{l['sym']}: {l['line']}" for l in st["log"][-60:] if l["push"]]
        lines = "\n".join(f"{n}. [{c['sym']}] {c['title']}" for n, c in enumerate(cands))
        kept = [(cands[k["n"]], k["line"]) for k in codex(JUDGE.format(sent="\n".join(sent) or "(none)", items=lines),
                                                          JUDGE_SCHEMA)["keep"] if 0 <= k["n"] < len(cands)]
        for i, line in kept:
            note(st, i, line, True, stamp)
        send([show(i, line) for i, line in kept])
        st["ids"].update({i["id"]: stamp for i in cands})
```

## Built with

Python's standard library only (urllib, xml, concurrent.futures, zoneinfo), public RSS and SEC EDGAR endpoints, the
Telegram Bot API, the Codex CLI with JSON-schema output as the judge, and Windows Task Scheduler.
