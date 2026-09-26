# Glossary

**CHT** — Cylinder Head Temperature. On this engine family, the same physical probe also serves
as the coolant-temperature reading (this specific engine has no separate per-cylinder head
sensor) — so CHT and coolant temperature are treated as one measured quantity, not two
independently cross-checkable ones.

**EGT** — Exhaust Gas Temperature, measured per cylinder (four probes on this engine). The
*spread* between cylinders — how far apart the four readings are — is often a more useful signal
than any single cylinder's absolute value, because it's a strong tell for a single-cylinder
problem (injector, ignition) versus something affecting the whole engine evenly.

**MAP** — Manifold Absolute Pressure. The turbo boost pressure the engine is actually running,
measured in hectopascals (hPa).

**RPM** — engine rotational speed.

**ΔT_ISA ("delta T ISA")** — how far the actual outside air temperature differs from the
International Standard Atmosphere's predicted temperature at the current altitude. This is the
second axis every published engine power table is indexed by (altitude is the first).

**FADEC** — Full Authority Digital Engine Control. A computerized engine-management system (as
opposed to purely mechanical carburetion and ignition timing).

**RUL** — Remaining Useful Life. An estimate of how much longer a component can run before it's
projected to cross a safety- or performance-relevant limit.

**P10 / P50 / P90** — the 10th, 50th and 90th percentile of a projection's uncertainty. In an RUL
context: P10 is the pessimistic (soonest) estimate of when a limit will be crossed, P50 is the
median best estimate, P90 is the optimistic (latest) estimate.

**BSFC / SFC** — (Brake) Specific Fuel Consumption. Fuel burned per unit of power produced
(kg of fuel per kWh). Notably *non-monotonic* with load on this engine — it doesn't just get
worse as load increases, it actually dips at a mid-range power setting before rising again near
maximum power.

**MCP** — Maximum Continuous Power. The highest power setting an engine is rated to run at
indefinitely (as opposed to Take-off power, which is time-limited).

**TBO** — Time Between Overhaul. The published service interval before a full engine overhaul is
required.

**Residual** — observed sensor reading minus the physics model's predicted "healthy engine"
reading for the current flight condition. The core signal essentially every detection and
degradation calculation in this project is built from.

**Persistence / N-of-M confirmation** — a rule requiring a deviation to appear in enough of the
last M samples (not just once) before it counts as a confirmed event, so a single noisy sample
can't trigger an advisory on its own.

**Theil-Sen (regression)** — a way of estimating a trend's rate of change by taking the *median*
of every pairwise slope between sample points in a window, rather than a single best-fit line.
Far more resistant to outliers/noise than an ordinary regression, at the cost of being a bit more
computationally expensive.

**Digital thread** — the recorded evidence chain behind a specific fault advisory: which sensor
channels were actually read, what their values and residuals were, and what prior internal state
the decision depended on — recorded in enough detail that the exact same advisory can be
independently re-derived from the record alone, with no reference back to the live system.

**Digital twin** — a live (or replayable) virtual model of a physical system, continuously
compared against real or simulated sensor data.

**Incident class** — a coarse category a specific fault is grouped under (for example:
lubrication, cooling, combustion, electrical) — used to keep the machine-learning classification
task tractable, since there are more individual named faults than there are cleanly distinguishable
sensor signatures.

**Root factor** — the single physical parameter a fault actually moves (for example, "cooling
effectiveness" dropping from 1.0). Every sensor reading coupled to that factor then responds
through the same physics equations a healthy engine uses — see
`02-engine-and-simulator-theory.md`.

**Severity trajectory / tau (τ)** — how a fault's magnitude develops over time once triggered.
"Tau" is the fault's own characteristic time constant — roughly, how fast it develops from
onset to full severity.

**Noise floor** — the smallest change in a sensor's reading that's distinguishable from its own
inherent measurement noise. Every detection threshold in this project is expressed as a multiple
of a channel's own noise floor, not as a bare engineering number.

**Provenance tag** — a label on every number in the system stating where it came from: a
published manufacturer figure, a value derived from published figures, or DARPAN's own
calibrated estimate. See `07-honest-limits-and-backlog.md`.

**SOP phase** — one step in the standard operating sequence a flight is modelled as walking
through: engine-off, pre-start, cranking, idle, warm-up, mag-check, taxi, takeoff, climb, cruise,
descent, cool-down.

**GBM (Gradient Boosting Machine)** — the machine-learning model used for fault detection; an
ensemble of decision trees trained sequentially, each one correcting the errors of the ones
before it.

**FAR (False Alarm Rate)** — how often the detector raises an advisory when nothing is actually
wrong. Every model comparison in this project is made at a *matched* FAR, so a "catches more
faults" claim can't secretly be explained by "and also alarms more often."

**Ablation (study)** — an experiment that deliberately removes specific input features (or
components) and re-measures performance, to check whether a result depends on something it
shouldn't (for example, whether an ML model is secretly just re-reading another system's own
output rather than learning something new from the raw data).
