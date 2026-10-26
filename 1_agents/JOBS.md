# The scheduled jobs

Six jobs run unattended on one Windows PC through Task Scheduler. Each one is a plain Python script with a
`.env` file for its keys; none needs a server or a database. Each reports to the owner's Telegram chat in one message,
and quiet runs stay quiet.

| Job | When | What it does | Page |
|---|---|---|---|
| **Nightly pipeline** | every weekday night, after the close | tops up the daily bars for ~3,000 stocks, runs the radar's nightly check, rebuilds the screener workbook, copies it, captures the rolling minute bars; weekly legs for 30-minute and hourly bars and earnings history | [nightly](../2_pipelines/nightly/README.md) |
| **Trade radar** | always on (a listener, restarted by the scheduler within a minute if it stops) | turns typed or spoken notes into checkable rules through the AI; the nightly check itself runs inside the pipeline above | [trade_radar](trade_radar/README.md) |
| **Trade news** | every minute in US trading hours, every 15 minutes otherwise | public feeds, SEC filings and halts on a watch list; an AI judge keeps only what could move the stock | [trade_news](trade_news/README.md) |
| **Borrow log** | weekdays, before the US open | snapshots a broker's public short-borrow file for the names on an event calendar; part of a no-orders shadow log | [PROTOCOL](forward_logs/PROTOCOL.md) |
| **Frozen-rule log** | weekly | re-scores a frozen forward-test rule on the newest data, using only what was public on each day | [FROZEN_RULE_LOG](forward_logs/FROZEN_RULE_LOG.md) |
| **Weekly backup** | weekly | copies the data stores and the code that has no git remote to cloud storage: copy only, never delete, keys never copied | [nightly](../2_pipelines/nightly/README.md#the-weekly-backup) |

## Lessons from running unattended

- **The scheduler's environment is not the terminal's.** A program installed for the terminal user (the AI CLI, for
  one) may not be on the scheduler's path, so each job reads the full path of the programs it calls from its `.env`.
- **A windowless Python has no console.** Every job writes its own log file and sends its own summary; nothing
  depends on printed output being seen.
- **A new job is proven with a probe task first:** a throw-away scheduled task that runs the same command once and
  writes what it saw, before the real schedule is set.
- **Never two at once.** Each task is set not to start a second copy, and jobs that share a file (the radar's
  notebook, the GPU for transcription) also take a lock.
- **Exit codes are not reports.** Several scripts return 0 even when some stocks fail, so the summaries quote each
  step's own counts (updated / repaired / flagged / failed).
- **Schedules follow the US market.** Jobs that must land before the open or after the close check New York market
  time themselves.
