# How a long study hands off between AI sessions

A big study (here `movers_grid`: two data layers, eight stages, fifteen amendments, about thirty scripts) outlives
any single AI session's context. Every study of that size keeps one `HANDOFF.md` at its root. A fresh session is told
one thing, "start from HANDOFF.md", and must be able to continue correctly from that file alone. Its shape:

## 1. The title line is the status

One sentence that is always current: what is done, what printed, what waits on whom. Example (rewritten):

> movers_grid hand-off (2026-10-23): the study is done. Amendments A11–A15 entered; the search printed 20 of 20 KILL;
> the season line KILL; the +10% list is the earlier and wider net; the verdict is in the README; proposed LESSONS
> lines wait on the owner's go.

## 2. "Start here": the reading order

Which sections of the frozen plan to read, in which order (the decision box, the bar, the luck check, the
amendments), then which result files. A session that skips this tends to re-derive decisions that were already made.

## 3. The blind protocol: what has been read, and what has not

The single most important block. It lists, stage by stage, which outcome tables have been printed and read, and which
are on disk but still unread, plus every outcome number ever shown before its stage (disclosed in the plan). An AI
session must never open an unread table "to check something": that would let the outcome steer the design.

## 4. Status: a dated ledger

One bullet per stage or amendment: the date, the owner's approval, the scripts built, their logs, the independent
review's result (a count of mismatches, never an outcome value), and the reproducible counts a later session can check
against (for example "list days unchanged: 89,074"). Backups taken before a change are named with their location.

## 5. Waiting on the owner

Only decisions that are the owner's: commits, readings a reviewer flagged as ambiguous ("kept as built unless the owner
says otherwise"), optional follow-ups. Each one says what it would change and what it would not.

## 6. Known limits, and Next

Known limits point to the log lines that measure them. "Next" is a numbered plan; finished items stay listed with
their date, so the history of the study is readable top to bottom.

## 7. House rules that bind the study

Repeated in every hand-off because a fresh session has not seen them:

- Other repos and other studies' folders are read-only; never import another study's modules (copy the function).
- Every script takes `--limit N` and then writes only under `out/_smoke_limitN/`: smoke first, then the full run.
  Never smoke-run inside a study's real output folders.
- Any change to a frozen rule is a dated amendment, proposed in plain words and approved before that stage's
  statistics ([PLAN_AMENDMENTS.md](PLAN_AMENDMENTS.md)).
- Builds run as builder → independent reviewers → fixer → cross-check; the code and the logs on disk are the record.
- The owner commits. A session proposes the exact files and the exact patch; it never commits other sessions' edits
  along with its own.

## 8. When a later study builds on this one

A finished study leaves one reference page for the studies that reuse its outputs:

- **Which file answers which question**, and whether each file is known on the day or is an **outcome**. Outcome files
  are read only under the reading study's own blind protocol: its code may use them, but nothing it prints shows a
  value from them before its own luck check.
- **Which version was read.** Each output's md5 sits in its writer's log; the reading study writes the md5s it read
  into its own log and re-checks them before every print.
- **Copy, never import.** Functions are copied together with their tests, and a boundary test proves the copy runs
  without the original study's modules.
- **One seed convention.** Random draws come from a hash of a seed string that names the study, so two studies never
  share draws.

## Why it works

- The model's memory is the file system, not the chat: plan, hand-off, logs and LESSONS are written for the next
  reader, and the next reader is often a model.
- Disclosure is explicit, so "did we look?" always has an answer.
- Every claim in the status is checkable against a count in a log, so a new session verifies before it builds.
