# ML & detection — the real, recorded results

DARPAN ships **two** detectors, deliberately, not one replacing the other:

1. A **physics-only detector** — pure residual-vs-threshold logic against the forward plant
   model described in `02-engine-and-simulator-theory.md`. Fully explainable: every decision
   traces to a specific channel crossing a specific, cited band.
2. A **gradient-boosted machine-learning model**, trained to catch what the physics thresholds
   miss.

The project's own build rule for whether the ML model was even allowed to ship: *"beat the
physics baseline on lead time at matched false-alarm rate, or ship the physics detector alone and
say so."* It beat it — the numbers below are how by how much, measured honestly, on data the
model never trained on.

## Detection bands are measured in noise-floor multiples, not raw units

Every threshold in the physics detector is expressed as a multiple of that channel's own known
sensor noise floor, never as a bare engineering number like "5 degrees." A threshold below the
noise floor is rejected automatically at build time — it would just be alarming on sensor
quantisation, not on anything real.

## The physics-only baseline — recorded result

Measured over roughly 4 hours of simulated healthy flight (14,404 samples) plus a set of
injected-fault test missions:

| Metric | Result |
|---|---|
| False advisories during healthy flight | **0** |
| Healthy samples correctly reported "nominal" | **99.45%** |
| Injected faults detected before reaching full severity | **5 of 5** tested |
| Sensor drift misread as a mechanical engine fault | **0.037%** of samples (non-consecutive, ramp-in transients only) |
| A dropped instrument lane misread as a mechanical fault | **0** |
| A real mechanical fault misread as instrumentation | **0** |

The last three rows are the direct test of the project's stated differentiator — telling a
failing sensor apart from a failing engine — and they're measured as their own explicit metric,
not folded into a generic accuracy number.

## RUL (remaining-useful-life) band coverage — recorded result

The acceptance bar: *"the true crossing time falls inside the predicted 10th–90th percentile band
in at least 80% of runs."* Measured over 11 held-out test runs (varied seed, mission profile, and
fault onset time):

| Metric | Result | Bar it had to clear |
|---|---|---|
| True crossing time landed inside the predicted band | **11 / 11 = 100%** | ≥ 80% |
| Median band width (90th ÷ 10th percentile time) | **4.5×** | < 25× |
| A healthy engine falsely projected toward a limit | **0%** of samples | < 15% |

100% is *over*-coverage relative to the 80% bar — stated directly in the project's own notes as
the right side to err on for anything safety-adjacent, not something to be proud of as
"precision." Narrowing that band further honestly needs real engine data, not more tuning against
the same synthetic missions.

## The machine-learning result

Measured on a **held-out operating condition** — every high-altitude mission profile was
excluded from training entirely, so the test isn't just held-out samples from a distribution the
model already saw, it's a whole flight regime the model never trained on.

- 234 missions → 75,114 five-minute feature windows → 49,764 for training, 25,350 for the
  held-out test
- All three detectors compared at the **same false-alarm rate** (0.186%), so the comparison is
  apples-to-apples

| Detector | Detected (of 54 held-out fault runs) | Severity at the moment of detection |
|---|---|---|
| Physics baseline | 16 / 54 | 0.537 |
| Isolation Forest (unsupervised anomaly detection) | 6 / 54 | 0.590 |
| **Gradient boosting** | **47 / 54** | **0.110** |

Read together: the gradient-boosting model catches roughly **three times** as many fault runs as
the physics baseline, and it catches them at **roughly a fifth of the severity** — meaning, on
average, much earlier in the fault's development, while it's still a small problem rather than an
advanced one. This is the actual case for why the ML layer exists at all.

## Proving the model isn't just re-reading the physics detector's own verdict

Six of the model's input features are quantities the physics detector already computes as part of
its own reasoning. Training on those and then claiming a win would be close to circular — so the
project ran a feature-ablation study, stripping different feature groups out and re-training from
scratch each time:

| Variant | Detected | Severity at detection |
|---|---|---|
| Physics baseline (for reference) | 16 / 54 | 0.537 |
| GBM — raw residual statistics only (49 features) | 45 / 54 | 0.110 |
| GBM — physics-derived features only (10 features) | 37 / 54 | 0.249 |
| GBM — everything (59 features) | 48 / 54 | 0.109 |
| GBM — everything + component-coupling features (61 total) | 47 / 54 | 0.110 |

The result that matters: the model trained on **raw residual statistics alone** — nothing derived
from the physics detector's own reasoning — still catches 45 of 54, almost matching the
full-feature model. The model is learning something genuinely present in the data, not
repackaging the physics detector's own output.

One honestly-reported negative finding from the same study: adding hand-engineered
"component-coupling" features (encoding which subsystems are physically linked) very mildly
*hurts* detection count (48 → 47) for essentially no change in severity. The project's own
transparency page reports this plainly as "no measurable help," rather than quietly dropping the
losing experiment from the writeup.

## Why the ML layer still doesn't replace the physics one

The physics detector stays in the shipped system as the **explainable fallback and safety gate**
— it never refuses to produce a reason, and its reasoning is checkable against a real formula
line by line. The ML model is faster and more sensitive, but its individual verdicts are harder
to hand-audit. Running both, and treating the physics path as the one that's allowed to override
in a safety-relevant direction, is the project's actual answer to "explainable AI" — not a
separate feature bolted on, but a structural choice about which model gets the final say.

See `data/ml-results.json` for these same numbers in machine-readable form, and
`07-honest-limits-and-backlog.md` for what this result explicitly does *not* claim (real engine
validation, coverage of the full fault catalogue).
