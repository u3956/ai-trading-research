# Memory notes: how lessons outlive a session

The AI coding agent keeps a small file-based memory between sessions. Each note is one fact in one file with a short
header, and an index file lists every note in a single line that is loaded at the start of each session. A note
records a correction or a lesson together with **why** it matters and **how to apply** it, so the next session acts on
it instead of repeating the mistake.

Notes come in four kinds: the owner's preferences, feedback on how to work, project state, and pointers to external
sources. Notes link to each other by name. A note that turns out wrong is deleted or corrected, not left to drift.

Below are five real notes, rewritten for outsiders: file paths, dates of private work and the owner's words are taken
out, and the shape is kept.

---

```markdown
---
name: propose-before-changing-code
description: Findings and options come first; files change only on an explicit go-ahead, scoped exactly.
type: feedback
---

A question about how something behaves is a request for analysis, not permission to change it. An agent once
began editing a study script while the owner had only asked whether a problem could be mitigated; the owner
stopped it and asked for an explanation first. After the findings and options, the owner approved the study
edit and explicitly kept the live workbook out of it.

**Why:** study and analysis scripts are editable once approved; live product files (the workbook builder, the
screener) need their own, separate go-ahead.

**How to apply:** investigate read-only, report findings plus concrete options, wait for a pick. Honor the
scope of an approval exactly: approval to extend a study is not approval to touch the workbook.
```

```markdown
---
name: never-smoke-run-studies-in-place
description: Study scripts write to fixed output folders that back the written results; a quick test run overwrites them.
type: feedback
---

A study script writes its whole table set to a fixed output folder, and the study's write-up was produced from
those files. A "--limit 3" run to check a one-line edit wrote to the same folder and replaced them. The folder
was not in git and had no other copy; the outputs had to be rebuilt as of the original cut-off date, and the
rebuild matches the write-up to within rounding but is not the original.

**Why:** a smoke test that silently destroys the evidence behind a finished verdict.

**How to apply:** every script that takes --limit now writes a limited run to <folder>/_smoke_limitN/. For a
script without that guard, test by importing the edited function, or redirect its output constant to a scratch
folder. Before any full run, list the output folder's dates; if it must be regenerated, say so first.
```

```markdown
---
name: future-window-control-exclusion-trap
description: Excluding control days that have an event in their future window is look-ahead selection.
type: feedback
---

Never drop a control day s because an event falls in (s, s+k]; drop it only for events in [s-k, s]. The
forward half keeps only control days whose own outcome window holds no event, which inflates the baseline
whenever the event and the outcome share a driver.

**Why:** in one study this rule manufactured a -74 bp contagion effect and a -255 bp washout effect; both were
about zero with the past-only window. Three pre-registration reviewers approved the rule; an independent rebuild
that did not read the code caught it.

**How to apply:** write every exclusion window for a matched-control contrast as past-only. Before writing up,
run a verifier that re-implements the study from its README alone and varies the exclusion direction (past /
future / none). Logged in LESSONS, section 2.
```

```markdown
---
name: edgar-acceptance-time-shift
description: A third of the companies' stored EDGAR acceptance times read 4-5 h late after UTC->ET; fix per company.
type: reference
---

Stored EDGAR submissions files give, for about a third of companies, acceptance times that read 4-5 hours late
once converted from UTC to New York time (an earnings release accepted at 16:21 ET reads 20:21 ET; Microsoft
reads correctly, Apple and Nvidia do not). The errors are always late, so there is no look-ahead, but they move
"accepted before the close" gates and reaction days.

The shift is a property of the company, not of the file: label each company by a vote of its filings that are
impossible as stored (outside EDGAR's hours, after the filing date) against those impossible once shifted back.
2,176 of 6,400 companies come out shifted. A fresh download later read correctly, so files already on disk still
need the rule and every download keeps its date and md5.

**How to apply:** any study that reads acceptance times from stored files applies the per-company rule; copy its
functions, never import them. A rule change in a running study is a dated plan amendment.
```

```markdown
---
name: workbook-change-diff-before-install
description: A workbook change must leave everything else identical: build to a test path, diff, then install.
type: feedback
---

The owner's standing instruction for the workbook builder: make sure the rest of the workbook stays the same.
The procedure that satisfies it: build a baseline with the old code to a scratch path and diff it against the
installed book (it must be identical), make the change, build again, then diff cell by cell, formulas
included, against a whitelist of the regions the change is meant to touch. Prove any row shifts are pure
offsets, check that Excel does not hold the file, and only then copy the new book over the live one.

**Why:** the workbook is a daily instrument; unrequested layout drift, even one documentation row moving, is a
defect.

**How to apply:** never install a rebuilt workbook without the diff. Diff a baseline build, not yesterday's
install (the data rolls forward daily and would hide the real change). Array formulas compare by identity in
openpyxl, so compare their range and text instead. New columns go at the end so letters do not move. An
independent reviewer is worth it: one caught a helper block that broke when a session row was deleted.
```
