# Degradation tracking & Remaining Useful Life (RUL)

## Step 1: turn a raw reading into a residual

Nothing is judged against a raw sensor value directly. Every monitored channel is first turned
into a **residual**: observed reading minus the healthy-engine prediction from the physics plant
model (see `02-engine-and-simulator-theory.md`), for the *current* flight condition. This is what
makes the system judge "is this abnormal for what the engine is doing right now" rather than "is
this number bigger than a fixed constant" — a healthy engine at full climb power runs hotter than
a healthy engine at idle, and a residual-based check already knows that.

## Step 2: don't trust one noisy sample — require persistence

A single noisy sample crossing a threshold isn't treated as a confirmed deviation. The system
requires **N-of-M confirmation** — a channel has to be outside its band in enough of the last M
samples before it counts as "confirmed" — before that deviation feeds into anything downstream
(an advisory, a degradation episode, a trend estimate). This is a real, separate mechanism from
the detection threshold itself, and it's what stops a single spike or a moment of sensor
quantisation from reading as a real event.

## Step 3: work out if it's actually *trending* — Theil-Sen regression

### Why not just take (now − then) / elapsed time?

This is the obvious first approach, and the project explicitly tried it and rejected it, with the
reasoning kept in the source rather than just the conclusion: on a typical thermal channel, raw
sensor noise alone produces an *apparent* rate of change that is roughly **twice the size** of a
real, developing fault's actual rate of change. A naive two-point difference is measuring noise,
not a trend.

### Why Theil-Sen instead of an ordinary least-squares fit

An ordinary regression line uses every point, which helps average out noise — but a single
dropout, spike, or sensor-quantisation artifact drags the whole fitted line with it, and this data
has all three by design (it's simulating real, imperfect instrumentation). Theil-Sen instead takes
the **median of every pairwise slope** between all the sample points in a window. That median is
mathematically tolerant of roughly **29% of the points being corrupted** before it breaks — and,
as a side effect that turns out to be just as useful, the *spread* of all those pairwise slopes
directly gives an honest uncertainty band on the rate itself, with no assumption that the noise is
smoothly bell-curve shaped (it isn't, once instrument quantisation is involved).

### Why one window length isn't enough — multi-scale windows

The catalogue of faults this system models spans an enormous range of how fast they develop — from
roughly two minutes for the fastest (an oil-pressure fault) to roughly ten hours for the slowest
(a lead-fouling/sludge fault). A single fixed window length cannot serve both: a window long
enough to see the slow fault clearly is *longer than the entire lifetime* of the fast one, so most
of that window is flat, pre-fault data and the computed rate comes out near zero — which was
measured, concretely, to produce a nonsensical "1,150 seconds to the limit" estimate on a fault
that was actually 88 seconds away.

The fix: run the Theil-Sen fit at **three window lengths at once** (60, 180, and 600 seconds), and
take whichever one currently shows the **fastest established trend**. "Established" has a precise
meaning here, not just "any nonzero slope":

- The full uncertainty band from that window's pairwise-slope spread has to sit **entirely on one
  side of zero** — if the low and high estimates disagree on even the *direction* of change, that
  window isn't trending, it's noise, and it's discarded rather than averaged in.
- The window also has to span at least **30 seconds of real elapsed time**, not just a minimum
  sample count. This was added after a real, measured false trigger: six samples spaced only a few
  seconds apart happened, by chance, to have pairwise slopes that all agreed with each other,
  which the original rule (six samples, no time requirement) accepted as an established trend
  within the first 5–9 seconds of a mission.

## Step 4: turn a rate into a time-to-limit projection (P10 / P50 / P90)

Once a channel has an established rate, projecting it forward to the point it would cross its
published limit gives a time estimate — but a single number would overstate how precisely that's
known. Instead the system reports a **band**:

- **P10** (earliest plausible crossing) comes from the *fastest* credible slope in the
  uncertainty range — the pessimistic case.
- **P90** (latest plausible crossing) comes from the *slowest* credible slope — the optimistic
  case.
- **P50** is the median-rate projection — the single best estimate.

The band is also widened, deliberately, the further out the projection reaches — because a
regression fit only really describes the behavior *within* the window it was fitted on; projecting
it further into the future is inherently less certain, and the band width grows to reflect that
rather than pretending a 600-second fit is equally confident about an hour from now.

## Sensor drift is judged *before* persistence, on purpose

Ordering matters here in a way that's easy to get backwards. If a whole instrument lane goes
missing, that produces *no residual at all* on the channels behind it — there's nothing to
confirm. If the instrumentation check ran *after* the persistence/confirmation step, a dead lane
would quietly read as "nominal" (nothing to confirm means nothing gets flagged) instead of being
correctly reported as a real, active instrumentation failure. So the check for "is a whole lane
missing" runs first, ahead of and independent of the normal residual-confirmation pipeline.

## One channel is allowed to override the confirmation requirement

A single rule exists for the highest-trust, fastest-moving safety channel (oil pressure): if it is
**far** past its action band, it raises an advisory even without a second, corroborating channel
confirming it. Nothing else in the physics model is causally coupled tightly enough to oil
pressure to reliably corroborate it in time, and oil pressure is, by the project's own reasoning,
the single fastest path to actually losing the engine — so requiring a second witness before
acting on it would mean sometimes not acting on it at all.

## The result: a per-component degradation log, not just a single alarm

Rather than a flat list of alarms, the system builds a **per-component episode log** — one entry
per continuous stretch of time a given subsystem spent outside its steady band, closed only once
that subsystem has been back inside its band for long enough that a persistence-clear rule + the
60-second minimum trend window both agree it's genuinely settled. An operator's own
acknowledge/resolve action on an episode is recorded **separately** from the machine's own
open/recovered verdict — the two are never merged into one status, so a human's judgment call
never silently overwrites what the detector actually measured.
