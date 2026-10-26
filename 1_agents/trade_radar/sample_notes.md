# Sample conversation with the radar

**Sample data.** Everything below is made up by hand to show the message formats: the ticker `DEMO`, its prices and
the notes are invented, and none of it comes from the owner's notebook. The formats follow the bot's code.

---

**Owner** (voice note):

> demo, nice base under 52, interesting if it closes above 52 with the 21 rising

**Radar:**

> 🎙 “DEMO, nice base under 52, interesting if it closes above 52 with the 21 rising”

> 📌 **DEMO** parked. I'll write when its chart changes.
>
> **Rule #1**, waiting for your ok:
> closes above $52.00 and the 21 is rising
>
> ❔ Rising 21 = the daily 21? Say 'weekly' if you meant the weekly line.
>
> now: close $50.84 (2.2% under $52.00) · the 21 is rising
>
> [ ✅ Watch it ] [ ✖ Just a note ]

**Owner:** ok

**Radar:**

> ✅ Watching **DEMO** rule #1

---

**A few nights later**, in the nightly message:

> 📡 **Radar** · Tue close · 1 of 4 moved
>
> **DEMO** $51.62 +1.1% today · +1.5% since Oct 5
> ⏳ getting close to your rule #1 (0.3 ADR to go): closes above $52.00 and the 21 is rising
> tomorrow's lines: 9 $51.02 · 21 $50.37
>
> quiet: AAAA BBBB CCCC

**The next night:**

> 📡 **Radar** · Wed close · 1 of 4 moved
>
> **DEMO** $52.48 +1.7% today · +3.2% since Oct 5
> 🔔 **your rule #1 happened**: closes above $52.00 and the 21 is rising
>    your note of Oct 5: “DEMO, nice base under 52, interesting if it closes above 52 with the 21 rising”
>    it's off now: “rearm DEMO” watches for it again
> tomorrow's lines: 9 $51.31 · 21 $50.56

---

"Tomorrow's lines" are exact, not estimates: by the EMA recursion, tomorrow's EMA equals tonight's EMA whenever
tomorrow's price equals tonight's EMA, so tonight's EMA value is the price at which the next bar touches the line.
