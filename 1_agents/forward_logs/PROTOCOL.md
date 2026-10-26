# Forward-test protocol: a no-orders shadow log

How one lean that survived a backtest is tested going forward without placing an order. Research only: neither the
lean nor any result is published here.

## Why not a paper-trading account

A simulated account fills a short at the displayed quote, with no real locate. For a thin stock at the open, that
hides the three things that decide the trade:

1. whether the shares can actually be borrowed that morning, and at what fee;
2. what the thin opening book really gives (the opening cross, the first minutes' spread);
3. what a squeeze or a volatility halt does to the position.

So the instrument is a **shadow log**: every qualifying event is scored from the tape the next morning, and a few
real-time readings of the book and of borrow availability are recorded on event days. It tests feasibility and
builds forward evidence without risking anything.

## What counts as an event

A fixed definition, written before the first event, in which timestamps decide and never the price move: when the
news was first public comes from the wire, the EDGAR acceptance time (corrected for the EDGAR clock shift, see
[LESSONS](../../5_research/LESSONS.md)) or the Nasdaq halt feed. The definition itself is not published here.

**Controls, always logged and scored, never traded:** a placebo family that should show nothing, an out-of-sample
family, and every skipped event with its reason.

## What is scored

From the tape, for every logged event: open to close and the 5-minute bar to close, each with and without a
catastrophe guard, at two cost levels; the largest move against the position before the close; the time of the day's
high; and the sector index over the same window. One line is named the primary line in advance; the others are
reported beside it. Rows are written within the hour and never edited afterwards (a correction is a new row).

## Decision rule, written before any event

- A **kill** line and a **continue** line at 10, 20, 30, 40, 50 and 60 events, simulated from the backtest's
  per-trade spread before the first event, with their error rates printed (the chance of killing a real edge, and of
  continuing a dead one).
- Kill takes precedence. "Continue" only means the log goes on; it is never a "keep": a keep still needs the house
  bar (placebo, both halves, the top events dropped).
- **Feasibility gauges for the first 10 events**: the share of events that could not be borrowed, the slippage of the
  would-be fill, guard hits, and the share of events missed for non-rule reasons. One red gauge pauses the test for
  a written review.

## How slow it is, said up front

Qualifying events are rare (single digits a year). The log can reject the trade and can prove or disprove
feasibility; it cannot prove an edge within a year or two. That is written in the protocol so that nobody reads an
early streak as evidence.

## What the independent review changed

- The first draft called for paper trades; the review showed paper fills cannot test the locate, the cross, halts or
  squeezes, so the instrument became the shadow log.
- A post-hoc subset that looked strong lost its significance after dropping two events: it is not quoted as evidence.
- Choosing the reaction session by its price move inflated the result slightly: the protocol uses timestamps only.

## Automation

The event calendar is built from SEC filings, a weekday job logs borrow availability from a broker's public
short-borrow file (`borrow_log.py`, about 110 lines, not shown), and a nightly scorer fills the tape columns. No part of
it places an order.

Each weekday before the US open the borrow job takes a snapshot of that file (shares available and fee for every
symbol), keeps the raw file compressed so a name that joins the calendar later still has its history, and logs one line
per name on the event calendar. A file version already logged is skipped, so a second run the same morning adds nothing.
