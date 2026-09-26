# Honest limits — what's real, what's estimated, and what's a declared boundary

This section is not an apology or a "future work" list dressed down. In DARPAN's own project
culture, stating a limit clearly is treated as *part of the deliverable*, not a gap in it. This
file explains why, and lists the actual limits as the project itself states them.

## Why: a real, documented failure this project was forked away from

DARPAN was built starting from an earlier, related project (a monitoring system for a large
marine diesel engine). That earlier project has a documented postmortem worth knowing, because
it's the direct reason DARPAN's provenance discipline exists at all:

On that earlier system, alarm thresholds were repeatedly nudged, one small change at a time, each
one locally reasonable in isolation — "this alarm fires a bit too eagerly, loosen it slightly." No
single change looked wrong. But over enough of these small, individually-defensible adjustments,
the *cumulative* effect walked the alarm envelope far enough away from the manufacturer's actual
published limits that a **fully healthy engine** ended up violating five separate manufacturer
limits — three of which were supposed to be automatic-shutdown conditions. And because those
thresholds fed everything downstream — baselines, degradation weights, risk scores, even labels
stored for machine learning — every one of those downstream systems silently inherited the same
error, with nothing to catch it.

DARPAN's response to that history is a hard rule, stated directly in the project's own
documentation: **"do not launder an estimate into a fact by restating it without its tag."**
Every limit, threshold, baseline value, and coefficient in the codebase has to cite where it came
from — a real manufacturer document, or an explicit `ESTIMATE` / `GUESS` tag if it doesn't. And a
threshold is never adjusted just to make an alarm stop firing — either the underlying physics
model is fixed, or the discrepancy is written down as a known issue.

## Declared, permanent limits — not chased, stated instead

**The sensor suite is intentionally frozen.** The system monitors exactly the set of channels a
real published research dataset for this engine class actually contains — no channels were added
beyond what that dataset provides, even where a richer sensor would obviously help. This was a
deliberate decision, guided by the project's own rule: *"if a number is not in the data files, it
does not exist yet."* Concretely, this freeze means:

- **Only one cylinder-head-temperature-equivalent probe exists** (shared with the coolant
  reading) — there is no per-cylinder head sensor, so per-cylinder thermal imbalance can't be
  diagnosed the way exhaust-gas-temperature imbalance can.
- **Vibration is a single scalar reading from one mounting point**, not a multi-axis or
  frequency-banded signal — so faults that would show up as a specific *vibration signature*
  (rather than just "more vibration than usual") are explicitly unrepresentable in this model, not
  merely undetected.
- **There is no knock sensor and no propeller-shaft-speed sensor** in the modeled instrument
  suite, even though the real, larger engine variant this project references *does* have a knock
  sensor — the published research dataset this project is built from doesn't carry that channel,
  so it isn't modeled.

**Only 11 of the full 24-fault catalogue are actually trainable for machine learning** — and this
is explicitly the ceiling the frozen sensor suite imposes, confirmed by direct measurement, not
just an assumption. The other faults each require a channel the current sensor suite doesn't
carry (spectral vibration analysis, a knock signal, propeller-shaft rpm) to even be
*representable*, let alone detectable — no amount of relabeling existing data fixes that. One
specific, understood exception exists: one more fault (a generator/alternator failure mode)
already has everything else it needs and could become the 12th trainable fault — but promoting it
requires an explicit retrain-and-remeasure decision, deliberately not taken automatically as a
side effect of an unrelated change.

**No real engine data exists anywhere in this project, and none is claimed.** Every detection
threshold, every thermal slope, every noise-floor figure is either a published manufacturer number
used as printed, or a DARPAN-authored calibrated estimate — clearly tagged either way. This is
stated as the project's single biggest open limitation on any false-alarm-rate claim: without a
real bench or flight dataset to validate against, no number in this system can honestly claim to
be validated against reality, only against the physics the project itself encoded. The project's
own instruction to itself: say this plainly in every deliverable, rather than letting it be
inferred.

**Take-off power is capped at the engine's own published critical altitude** (roughly 8,000 feet
for the performance-baseline engine variant) — this is not a bug or an arbitrary modeling
shortcut, it's the actual published rating of the real engine being modeled. Attempting a
full-power take-off above that altitude is refused by the model rather than extrapolated, because
extrapolating past a published rating would be inventing a number nobody published.

**The evidence-chain integrity check is a corruption detector, not a cryptographic signature.**
Every recorded advisory carries a checksum (FNV-1a) that reliably catches accidental corruption —
truncation, reordering, a record re-saved by a tool that altered it. It provides **no protection
against deliberate tampering** — anyone who edits a record can simply recompute a matching
checksum. The project is explicit that this log must never be described as "tamper-evident,"
only as corruption-checked.

## Model gaps that are honestly distinguished from sensor gaps

A few readings exist as real telemetry channels but currently have a *thin* model behind them —
this is called out as a different, more fixable category than the permanent sensor-suite limits
above:

- **Alternator/bus current** is currently modeled as a flat constant rather than a real function
  of electrical load — the sensor channel genuinely exists, the physics behind it just hasn't been
  built out yet.
- **Fuel-injection timing** as a monitored parameter was dropped entirely at one point in the
  project's evolution — not left thin, removed outright — once it became clear the
  performance-baseline engine variant (carburetted, mechanically fuel-injected) has no FADEC
  injector-pulse-width signal to read in the first place. The project's own framing: keeping a
  placeholder "estimated" version of a channel the real baseline engine cannot physically produce
  would have been a worse kind of dishonesty than dropping it and saying so.

## The takeaway

None of the above is hidden inside the codebase for someone to eventually discover. It's written
down, in the project's own primary documentation, as plainly as its successes are. That's the
actual point of this file existing at all — an interactive explainer that quietly dropped these
limits while keeping the impressive numbers from `03-ml-and-detection.md` would be misrepresenting
the project it's trying to teach.
