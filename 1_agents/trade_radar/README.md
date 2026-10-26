# Trade radar: notes in plain words, rules the code can check

A Telegram bot that remembers the stocks the owner parks. The owner sends notes in their own words, typed or as a
voice note ("interesting if it closes back above the 50 on light volume"). An AI turns any condition in a note into a
rule the code can check, the bot repeats the rule back in plain words for a yes or no, and every night after the close
it re-reads each parked stock's daily chart and writes only when something changes. It is an attention tool, not a
signal: it says when to look at a chart that would otherwise be forgotten.

This folder shows **how it is built and checked**. The code (four Python modules, about 3,900 lines, plus the router
prompt and its answer schema) appears here only as the excerpts below. The owner's notes and rules are not published.

## What it does

- **Park and note.** A bare ticker starts tracking a stock; any other message is a note, kept verbatim with the day's
  close. A chart screenshot sent with or just before a note is kept with it.
- **Rules from notes.** When a note holds something measurable (a line crossed, a pullback to a moving average, a move
  in % or in ADR, volume drying up, a new high, a market leg such as "while SPY is above its 50"), the bot answers with
  the rule exactly as the code will check it, the one guess it made written as a question, and two buttons:
  **✅ Watch it** / **✖ Just a note**. "No, the 50" rewrites the rule before it is armed.
- **Every night after the close**, right after the data pipeline has downloaded the day's bars
  ([../../2_pipelines/nightly/](../../2_pipelines/nightly/)): each rule says "getting close" once and "it happened"
  once, then rests until re-armed. The message also gives **tomorrow's lines**, the exact prices where each stock
  would touch its 9- and 21-day EMA in the next session. By the EMA recursion, the price at which tomorrow's bar
  touches the line is tonight's EMA, so these are exact, not estimates.
- **Groups and the indexes.** "watch <group>" follows one of the workbook's groups name by name, and SPY and QQQ can be
  watched against their own averages. A group's tests count names (the median name, or a share of the names) instead
  of drawing one index line, so a runaway stock or a bad print cannot fake a move. Each watched group gets a nightly
  log of what its names did against their 9, 21 and 50.

## How it is built

- **No AI for plain commands.** A ticker, "TICKER?", "list", "drop …", "ok" are matched by regular expressions and
  answered at once. Everything else goes to the router.
- **The router is a prompt plus a fixed answer format.** The prompt (`prompts/route.md`) describes the 11 actions and
  23 condition kinds and warns about speech-to-text noise. Each call adds a context block:
  the tracked stocks with their current lines, the rule waiting for an answer, the message being replied to. The AI
  runs as a locked-down CLI (read-only sandbox, no web search, no tools, nothing kept between calls), and its answer
  is forced into a strict JSON schema (`schemas/route.json`).
- **The code, not the AI, decides what fires.** Every condition the AI returns is checked again by code before it
  becomes a rule; anything it cannot express stays in the note, and the bot says what it could not watch.
- **Voice notes** are downloaded from Telegram, transcribed by a local Whisper install, echoed back word for word, and
  then routed like typed text.
- **State that can be rebuilt.** One notebook file, written atomically under a lock shared by the listener and the
  nightly job, with a copy kept each day; a journal of every message verbatim, from which the notebook can be rebuilt;
  every AI call's prompt, answer and log kept on disk.
- **It keeps running unattended.** The scheduler restarts the listener within a minute if it dies, and only one
  listener can run at a time. Telegram rate limits and network errors are retried. A nightly message Telegram refused
  is queued and resent by the listener. When the AI is unavailable, a message that starts with a ticker is still saved
  as a note. Messages and buttons from any chat other than the owner's are ignored.
- **One set of numbers.** The chart arithmetic loads bars through the screener's own functions, so every EMA and ADR
  the bot quotes equals the workbook's ([../../3_workbook/](../../3_workbook/)).

## How it is checked

- `selftest` checks the arithmetic, the rules and the notebook against known cases replayed from history, with no
  Telegram and no AI.
- `replay TICKER --park DATE` runs the nightly check night by night on a throw-away notebook and prints what the bot
  would have sent.
- `try-route "words"` shows what the AI makes of a message; nothing is saved or sent.

## Excerpts

**1. The router prompt** (`prompts/route.md`, opening, three of its actions and two of its rules; `…` marks a cut):
the prompt is the whole contract with the AI, written as plain instructions plus worked examples.

```text
You are the note-taker of one trader's stock radar, a Telegram bot. The trader parks US stocks they want to remember, adds notes in their own words, and asks to be told when something happens on a stock's daily chart. They can also watch a whole GROUP of stocks, and SPY or QQQ. Most messages are spoken and transcribed, so expect transcription noise: …, company names instead of tickers, filler words, repeated words. Each message comes to you: choose ONE action and reply with JSON matching the schema. Fields an action does not use are "", [] or 0.

## Actions
...
- **note**: something to keep about ONE stock (or one group, or SPY/QQQ): what they see, a plan, why it is interesting, a condition to watch for. symbols = [its ticker]; note = their words, verbatim, only filler and repetitions removed (keep their wording and every detail; do not summarize, do not add); conditions = the rule when the note asks to watch for something ("if it ever...", "when it...", "tell me if...", "interesting if...", "it would be a setup if..."), else []. since: as for park, when they say where tracking should count from. A stock that is not tracked yet starts being tracked with its first note.
- **confirm**: they accept the rule waiting for an answer (PENDING RULE): "ok", "yes", "good", "correct", "that's it", "perfect".
- **correct**: they change the PENDING RULE ("make it the 50 instead", "only on a weekly close", "also with dry volume", "drop the volume part", …): symbols = [the pending rule's ticker, group or SPY/QQQ]; conditions (and more_rules) = the WHOLE corrected rule, not just the change; note = "".
...
Anything a condition cannot express (news, fundamentals, a time of day, another stock, intraday or pre-market prices, "a good setup" with nothing measurable) is left out of conditions: stays in the note, and left_out says in a few words what you could not watch ("the news part").

assumption: when a word in the rule could mean two things, the ONE guess you made, written as a short question they can answer in a word or two. … Also "below the 9/21" without "close": basis close is the default; only ask when it matters. No doubt: "".
...
```

**2. The answer schema** (`schemas/route.json`, shortened): strict, every field required, every choice an enum, so
the reply always parses into one of a fixed set of actions.

```jsonc
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "action": {
      "type": "string",
      "enum": [
        "park",
        "note",
        "confirm",
        "correct",
        "card",
        "list",
        "drop",
        "rearm",
        "rule_off",
        "help",
        "reply"
      ]
    },
    // ... symbols, since, note
    "conditions": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "kind": {
            "type": "string",
            "enum": [
              "price",
              "line",
              "slope",
              // ... 23 kinds in all
            ]
          },
          // ... line, line2, side, basis, value, unit, days, anchor, ref, share
        },
        "required": [
          "kind",
          // ... every field
        ]
      }
    },
    // ... more_rules, assumption, left_out, ids, reply
  },
  "required": [
    "action",
    // ... every field
  ]
}
```

**3. The locked-down AI call** (`trade_radar.py`): every capability the task does not need is switched off, the answer
must match the schema, and each call leaves its prompt, answer and log in its own folder. A usage-limit refusal is its
own exception, so the bot can fall back instead of failing.

```python
LOCKDOWN = ["--ignore-user-config", "-c", 'web_search="disabled"'] + [
    arg for feature in ("apps", "plugins", "remote_plugin", "browser_use", "browser_use_external", "computer_use",
                        "in_app_browser", "multi_agent", "image_generation", "skill_search", "tool_suggest",
                        "shell_tool", "unified_exec")
    for arg in ("--disable", feature)]


class CodexLimit(Exception):
    """Codex refused: its usage limit is reached."""


def codex_run(prompt, name, schema, model, effort, timeout=CODEX_TIMEOUT):
    work = WORK / f"{datetime.now():%Y%m%d_%H%M%S_%f}_{name}"
    work.mkdir(parents=True, exist_ok=True)
    answer = work / "answer.json"
    exe = codex_exe()
    cmd = [exe, "exec", *LOCKDOWN, "--model", model, "-c", f"model_reasoning_effort={effort}",
           "--sandbox", "read-only", "--skip-git-repo-check", "--ephemeral", "--color", "never",
           "--output-schema", str(schema), "--output-last-message", str(answer), "-"]
    if exe.lower().endswith((".cmd", ".bat")):
        cmd = ["cmd", "/c"] + cmd
    (work / "prompt.txt").write_text(prompt, encoding="utf-8")
    t0 = time.monotonic()
    r = subprocess.run(cmd, input=prompt, capture_output=True, text=True, encoding="utf-8", errors="replace",
                       timeout=timeout, cwd=str(work), creationflags=NO_WINDOW)
    tail = ((r.stderr or "") + "\n" + (r.stdout or "")).strip()
    (work / "codex.log").write_text(tail, encoding="utf-8")
    log(f"  codex {name} ({model}, {effort}): exit {r.returncode} in {time.monotonic() - t0:.0f} s")
    if (r.returncode != 0 or not answer.exists()) and LIMIT_RX.search(tail[-3000:]):
        raise CodexLimit(tail[-300:])
    if r.returncode != 0 or not answer.exists():
        raise RuntimeError(f"codex exit {r.returncode}: {tail[-400:]}")
    return answer.read_text(encoding="utf-8")
```

**4. One retry when the reply is not JSON** (`trade_radar.py`):

```python
def route(nb, text, reply_ref=None, voice=False, latest=None):
    prompt = (PROMPTS / "route.md").read_text(encoding="utf-8").strip() + "\n\n" + \
        route_context(nb, text, reply_ref, voice, latest)
    raw = codex_run(prompt, "route", SCHEMAS / "route.json", ROUTE_MODEL, ROUTE_EFFORT)
    try:
        return parse_json_reply(raw)
    except ValueError as e:
        log(f"  route reply was not JSON ({e}); retrying once")
        return parse_json_reply(codex_run(prompt + "\n\nYour previous reply was not valid JSON. Reply again "
                                          "with ONLY the JSON object.", "route_retry", SCHEMAS / "route.json",
                                          ROUTE_MODEL, ROUTE_EFFORT))
```

**5. The AI's conditions checked again by code** (`trade_radar.py`, then `chart.py`): a condition that cannot be
watched is refused with a plain reason, which the bot shows to the owner, instead of being guessed at.

```python
def normalize_all(raw, df=None):
    """The router's conditions, checked; an anchor before the stock's first stored bar is refused."""
    conds, bad = [], []
    for c in raw or []:
        try:
            k = chart.normalize(c)
            if df is not None and k.get("anchor", "") not in ("", "high", "low") and \
                    pd.Timestamp(k["anchor"]) < df["date"].iloc[0]:
                raise ValueError(f"my bars start {chart.day_y(df['date'].iloc[0])}: {chart.day_y(k['anchor'])} "
                                 f"is before that")
            conds.append(k)
        except (ValueError, TypeError) as e:
            bad.append(str(e))
    return conds, bad
```

```python
def normalize(cond):
    """A condition from the router, checked and with its defaults filled in. Raises ValueError
    with a plain reason when it cannot be watched."""
    k = {f: cond.get(f) for f in FIELDS}
    k["value"] = float(k["value"] or 0)
    k["days"] = int(k["days"] or 0)
    k["share"] = float(k["share"] or 0)
    for f in ("line", "line2", "side", "basis", "unit", "anchor", "ref"):
        k[f] = str(k[f] or "")
    kind, side, line = k["kind"], k["side"], k["line"]
    if kind not in KINDS:
        raise ValueError(f"unknown condition '{kind}'")
    if kind in ("price", "line", "level"):
        k["basis"] = k["basis"] if k["basis"] in ("close", "touch") else "close"
    if kind in ("move", "off_high", "off_low", "gap") or (kind == "line" and side in ("near", "far_above", "far_below")):
        k["unit"] = k["unit"] if k["unit"] in ("pct", "adr") else ("adr" if kind == "line" else "pct")
    if kind == "price":
        if side not in ("above", "below") or k["value"] <= 0:
            raise ValueError("a price condition needs above/below and a price")
    # ... one branch per kind
    return k
```

**6. The rule repeated back before it is armed** (`trade_radar.py`): the words come from the code's own description of
the stored conditions, not from the AI, so what the owner approves is what will be checked.

```python
def echo(sym, st, rules, df, i, assumption="", left_out="", bad=()):
    """The rule(s) in plain words (what the code will check), where they stand, and the two buttons. An “X or Y”
    note makes several rules: either one writes."""
    ids = [r["id"] for r in rules]
    if len(rules) == 1:
        lines = [f"<b>Rule #{ids[0]}</b>, waiting for your ok:", f"{esc(rules[0]['words'])}"]
    else:
        lines = [f"<b>Rules {' or '.join(f'#{x}' for x in ids)}</b> (either one writes), waiting for your ok:"]
        lines += [f"#{r['id']} {esc(r['words'])}" for r in rules]
    if any(k["kind"] in MEASURED for r in rules for k in r["conds"]):
        lines.append("(I'll also tell you once when it gets within ½ ADR)")
    if assumption:
        lines.append(f"❔ {esc(assumption)}")
    if left_out:
        lines.append(f"✂️ not watched: {esc(left_out)}")
    for b in bad:
        lines.append(f"✂️ couldn't use: {esc(b)}")
    conds = list({json.dumps(k, sort_keys=True): k for r in rules for k in r["conds"]}.values())
    lines.append("now: " + "; ".join(esc(cond_now(df, i, k)) for k in conds))
    for r in rules:
        if chart.rule_state(df, i, r["conds"], chart.row_of(df, st["since"]))[0]:
            lines.append(f"({'#' + str(r['id']) + ' ' if len(rules) > 1 else 'it '}holds already today: I'd tell you "
                         f"the next time it happens)")
    tag = ",".join(map(str, ids))
    kb = keyboard([[("✅ Watch it", f"arm:{sym}:{tag}"), ("✖ Just a note", f"norule:{sym}:{tag}")]])
    return lines, kb
```

**7. A voice note, start to finish** (`trade_radar.py`): transcribed locally, echoed so the owner sees what was heard,
then routed like text. AI work runs one job at a time on a worker thread, so the listener never blocks.

```python
    def do_job(self, kind, job):
        """One message for the AI: a voice note is transcribed first; the router decides, act() does it."""
        text = job.get("text", "")
        try:
            self.tg.chat_action("typing")
            if kind == "voice":
                audio = self.tg.download(job["file_id"], VOICE / f"voice_{datetime.now():%Y%m%d_%H%M%S}.oga")
                text = transcribe(audio)
                if not text:
                    self.tg.send("🎙 I couldn't hear any words in that voice note.")
                    return
                self.tg.send(f"🎙 “{esc(text)}”")
            latest = chart.latest_session()
            res = route(nb_read(), text, job.get("ref"), voice=kind == "voice", latest=latest)
            log(f"“{short(text, 120)}” -> {res.get('action')} {' '.join(res.get('symbols') or [])}")
            self.carry_out(res, text, kind, latest, job.get("image", ""))
        except CodexLimit:
            self.fallback(text, job, "the AI usage limit is reached")
        except Exception as e:
            log(f"job {kind} failed: {type(e).__name__}: {e}\n{traceback.format_exc()}")
            self.fallback(text, job, f"{type(e).__name__}: {str(e)[:150]}")
```

**8. When the AI is out, nothing is lost** (`trade_radar.py`):

```python
    def fallback(self, text, job, why):
        """The AI is out: a message starting with a ticker is still kept as a note, in your words."""
        m = re.match(r"^\s*\$?([A-Za-z][A-Za-z.\-]{0,6})\b[\s,:-]*(.*)$", text or "", re.S)
        if m and chart.in_store(m.group(1)) and m.group(2).strip():
            try:
                res = blank("note", symbols=[m.group(1)], note=m.group(2).strip())
                self.carry_out(res, text, "fallback", chart.latest_session(), job.get("image", ""))
                self.tg.send(f"⚠️ {esc(why)}: I kept your words as a note but made no rule. Send it again "
                             f"later if it holds a condition.")
                return
            except Exception as e:
                log(f"fallback note failed: {e}")
        journal(via="lost", text=text, why=why)
        try:
            self.tg.send(f"⚠️ I couldn't read that ({esc(why)}). Your words are in the journal; send it again "
                         f"later.")
        except Exception:
            pass
```

**9. The notebook: atomic writes, a daily copy, a journal to rebuild from** (`trade_radar.py`):

```python
def nb_write(nb):
    """Atomic; the first write of each day keeps yesterday's file in backups/."""
    DATA.mkdir(parents=True, exist_ok=True)
    if NOTEBOOK.exists():
        BACKUPS.mkdir(exist_ok=True)
        b = BACKUPS / f"notebook_{datetime.now():%Y%m%d}.json"
        if not b.exists():
            shutil.copy2(NOTEBOOK, b)
    tmp = NOTEBOOK.with_name("notebook.json.tmp")
    tmp.write_text(json.dumps(nb, indent=1, ensure_ascii=False, default=jsonable), encoding="utf-8")
    os.replace(tmp, NOTEBOOK)


@contextmanager
def notebook(timeout=180):
    with locked(timeout):
        nb = nb_read()
        yield nb
        nb_write(nb)


def journal(**kw):
    """Every message, verbatim, with what the bot made of it: the notebook can be rebuilt from this."""
    try:
        DATA.mkdir(parents=True, exist_ok=True)
        with open(JOURNAL, "a", encoding="utf-8") as f:
            f.write(json.dumps({"at": now_s(), **kw}, ensure_ascii=False, default=jsonable) + "\n")
    except OSError as e:
        log(f"journal: {e}")
```

**10. The Telegram client's retry loop** (`tg.py`, shared with the other bots): network errors are retried with a
growing pause, and a rate-limit answer waits exactly as long as Telegram asks.

```python
    def api(self, method, files=None, **params):
        clean = {k: v for k, v in params.items() if v is not None}
        for k, v in list(clean.items()):
            if isinstance(v, (dict, list)):
                clean[k] = json.dumps(v)
        for attempt in range(3):
            try:
                r = requests.post(f"https://api.telegram.org/bot{self.token}/{method}", data=clean, files=files,
                                  timeout=90)
                j = r.json()
            except (requests.RequestException, ValueError) as e:
                if attempt == 2:
                    raise RuntimeError(f"telegram {method}: {e}")
                time.sleep(3 * (attempt + 1))
                continue
            if j.get("ok"):
                return j["result"]
            if j.get("error_code") == 429:
                time.sleep(int(j.get("parameters", {}).get("retry_after", 3)) + 1)
                continue
            raise RuntimeError(f"telegram {method}: {j.get('description')}")
        raise RuntimeError(f"telegram {method}: gave up")
```

## Limits

- Rules are checked once a day, on the close. For an alert during the session, the nightly message gives the exact
  price to set as a chart alert.
- Only stocks in the screener's bar store can be tracked (US names above roughly $500M market cap).

## Built with

Python (pandas, numpy, requests), the Telegram Bot API, the Codex CLI with JSON-schema output as the router, a local
faster-whisper install for voice notes, and Windows Task Scheduler to keep the listener alive.
