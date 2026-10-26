# How a frozen plan changes: dated amendments

A study's plan is frozen before any outcome is computed: the hypotheses (two-sided, with the expected direction),
the data layers, the comparison that decides, the statistics, the bar, the luck check and the order in which results
are printed. AI agents build against it. When building reveals that something in the plan is wrong, the plan is not
quietly edited: an **amendment** is written, approved by the owner, and entered before the statistics it affects.

## The form every amendment takes

1. **A number, a title, a date and the approval**, e.g. "A7: the EDGAR clock (before any stage-2 statistic)".
2. **What had been computed when it was written.** The exact state of the work, so a reader can see that no outcome
   could have steered the change ("no outcome of any line, entry, exit or comparison with look-alikes").
3. **The change itself**, written as a rule a builder can implement without asking.
4. **Why it is legitimate**: the error it fixes and how it was found (usually from counts known on the day, never
   from outcomes).
5. **Its effect, measured on inputs only**: how many rows it moves, printed in the build log.
6. **The checks printed at every build** that would show if the fix itself is wrong.

## Three real amendments, retold

### A7: the EDGAR clock

- **Found by:** an independent review of the company features, and separately by another study's session.
- **The problem:** in the stored copies of the SEC's submissions files, about a third of companies carry acceptance
  times with the New York offset added. Read as UTC, those filings land 4–5 hours late, so an 8-K filed after the
  close appears to have been filed in the evening, or the next day. (A fresh download in October 2026 read
  correctly; the stored copies still need the rule.)
- **The rule:** label each company (CIK), not each file. Read every filing two ways, "as stored" and "shifted back".
  Count the filings that are impossible under each reading (after their filing date, at night, or after 18:00 on the
  filing date for forms that are never filed then). The company is shifted when "as stored" has more impossible
  filings. No threshold to tune.
- **Rejected on the way:** a per-file 10% rule (it mislabeled named companies both ways) and two checks that could
  not pass even with a correct clock.
- **Checks at every build:** label counts and ties, filings stored under two CIKs agreeing after correction, a list
  of confirmed companies on each side, and every feature value the amendment changes.

### A14: an age bound on "latest revenue growth"

- **Found by:** the review of the quarterly-features script, from counts known on the day.
- **The problem:** the plan set no age limit, so a company whose recent quarters could not be computed was read
  through a quarter that ended years earlier. That is not its revenue growth on the day.
- **The rule:** the value is "not defined" when the quarter ended more than 200 days before the day.
- **Effect, printed before and after:** the list days it moves, on each data layer. The script and its review were
  re-run before any search result.

### A15: a stop cut sized on the final design

- **Found by:** re-reading the calibration's null worlds row by row.
- **The problem:** the search's centering check stops a run when one value's fake lift is off by more than 5% and by
  more than c standard errors. c = 3.5 had been sized on a smaller draft of the check; at the final scope it stopped
  23 of 480 pure-noise worlds by chance, and with fixed seeds a chance stop cannot be cleared by any fix.
- **The rule:** c = 4.0, the smallest cut stopping at most 1% of the worlds (4 of 480; 0.8% and 1.3% on split
  halves). The broken design the check exists to catch still stops in 480 of 480.
- **Legitimacy:** computed from synthetic worlds already run, entered before the print it governs, and reading no
  outcome.

## The full list, for scale

A1 data cleaning · A2 data follow-ups · A3 splits just before the second layer · A4 share counts for market cap ·
A5–A6 fixes to the cap build · A7 the EDGAR clock · A8 how look-alikes are drawn · A9 freezing every scoring reading ·
A10 price ties · A11 re-pointing the study at one question (which day-of traits make a +10% mover likelier to reach
5x) · A12 one feature's third value · A13 printing in two parts · A14 the age bound · A15 the stop cut.

Fifteen amendments, all entered before the statistics they touched. The verdict (no keep in 8,821 rows) is believable
because the plan could not have been bent towards it.
