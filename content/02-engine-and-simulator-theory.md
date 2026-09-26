# Engine & simulator theory

## Which real engine this models, and why two engines

DARPAN's twin is built on a real, published aero piston engine family — a horizontally-opposed,
4-cylinder, turbocharged design used in real light aircraft and UAV applications. Two variants of
that family are blended, deliberately and explicitly:

- **Performance baseline** (power ratings, published power-vs-altitude tables, operating limits):
  taken from the smaller, carburetted, non-turbo-intercooled variant of the family — used *as
  printed*, with no scaling factor applied.
- **Instrument suite** (dual-lane engine management, fuel-rail-pressure sensing, some derived
  temperature figures): taken from the larger, FADEC-injected, turbo-intercooled variant, and
  explicitly labelled "transferred" wherever it's used, rather than presented as if it belonged
  to the performance baseline.

Why blend rather than pick one: the smaller engine is the one whose complete, printed power
tables and limits the project actually has good source data for; the larger one is the one whose
richer instrumentation (dual ignition lanes, fuel-pressure sensing) is closer to what a real
modern health-monitoring system would actually have to read. The project is explicit that this
is a stand-in choice for a prototype, not a claim about any specific real aircraft — no aircraft
or engine manufacturer is named in any user-facing text, by design.

## ISA — the atmosphere model

Altitude and ambient temperature deviation are **first-class inputs** to the whole model — engine
power is not a single number, it's a *surface* over (altitude, temperature-deviation-from-ISA).
The International Standard Atmosphere gives the reference:

- Sea-level reference temperature: 288.15 K
- Sea-level reference pressure: 101,325 Pa
- Standard lapse rate: 0.0065 K per metre of altitude (the "6.5°C per 1,000 m" figure used
  throughout aviation)

From those three constants alone, the model derives: the ISA temperature at any altitude, the
ISA pressure at any altitude (via the standard barometric formula), how far the *actual* outside
air temperature deviates from what ISA predicts at that altitude (this deviation, "ΔT_ISA," is
exactly the second axis every published engine power table is indexed by), and air density —
which is what everything on the intake/turbo side of the model actually scales with.

None of this needs a citation to the engine manufacturer — ISA is a public international
standard, not proprietary engine data.

## The forward plant model — predicting a *healthy* engine

The single most important function in the whole codebase computes: given (rpm, throttle,
altitude, outside-air-temperature), what should every sensor channel read on a **healthy**
engine, right now? This is used by *two* different consumers for two different reasons — the
simulator calls it to generate the "ground truth" a mission flies against, and the detector calls
it to generate the "expected" value every residual is measured against. Using the exact same
function for both is deliberate: it's what makes "a fault injected at zero severity is
bit-for-bit identical to a healthy engine" true automatically, rather than something that has to
be carefully kept true by hand.

Roughly, in order:

1. **Power available** comes off a real, published lookup table indexed by (altitude, ΔT_ISA),
   sampled with bilinear interpolation — not a formula, an actual printed table, used as printed.
   Outside the table's published range, the model returns "not valid" rather than extrapolating a
   number nobody published.
2. **Power delivered** is that ceiling scaled by how far rpm and throttle are from their rated
   values.
3. **Manifold pressure** tracks published boost targets at the two rated engine speeds, scaled by
   throttle, and reduced by anything degrading the turbo or leaking pressure out of the intake
   plumbing.
4. **Thermal channels** (coolant/cylinder-head temperature, exhaust gas temperature, oil
   temperature) are anchored to one real reference flight condition and then move linearly with
   load and with outside temperature. The *slope* of that line is one of the model's few honestly
   labelled estimates — no manufacturer publishes a thermal-vs-power slope for this engine — but
   the slope is constrained so that a fully healthy engine, run at its own rated conditions, sits
   safely *inside* every published limit. That constraint is enforced by an automated test, not
   just a design intention.
5. **Fuel flow** comes off the baseline engine's own published propeller-curve fuel chart,
   re-indexed by power so it can be sampled at any condition, not just along one fixed load line.
6. **Oil pressure** is deliberately a *curve*, not a single threshold — the published safe band is
   itself rpm-dependent (a much lower minimum is acceptable below a certain rpm than above it), so
   a single flat number would be wrong somewhere in the operating range.

## Faults move a *root factor*, not a sensor reading — this is the important design decision

A fault in this model is never "add some amount to this one sensor's reading." Instead, a fault
moves one **root physical factor** — for example, cooling effectiveness dropping from 1.0
(healthy) to some lower fraction — and every sensor reading that's physically coupled to that
factor is left to respond *through the same physics equations a healthy engine uses*. So a
cooling fault raises coolant temperature directly, and oil temperature rises too — not because
anyone wrote "also raise oil temp," but because the oil-temperature equation already contains a
term for shared cooling-circuit heat, and that term now sees a worse cooling effectiveness. Two
simultaneous faults compose correctly through the same shared physics, rather than by adding two
independently-authored deltas that might double-count an effect or miss a real interaction.

This is also exactly how a genuine sensor problem is told apart from a genuine engine problem: a
fault applied to *instrumentation* (a biased sensor, or a whole dropped instrument lane) is
applied **after** the physics runs, never through a root factor — so its neighbouring, physically
coupled channels stay completely normal. A mechanical fault always drags its coupled channels
with it; an instrumentation fault never does. That gap is the detection signal.

## A tick has memory — the lag model

Nothing in a real engine changes instantly. A metal cylinder head doesn't jump to a new
temperature the instant load changes — it approaches it, over time, following (approximately) a
first-order exponential lag with its own time constant per channel. The model applies exactly
that: every tick, the *lagged, observed* state moves part-way from where it currently is toward
the newly computed *steady-state target*, at a rate set by that channel's own time constant.
Faults inherit this for free — a slow-developing problem like coking shows up over many minutes,
not instantly, purely because its root factor feeds into the same lagged channels everything else
does.

Two seeding modes exist for a reason: a mission that starts mid-flight seeds every lagged channel
already at its settled value (so it doesn't fire a spurious "warming up" alarm the instant it
starts); a mission that starts from engine-off seeds every thermal channel at ambient temperature,
so a genuine cold-start and warm-up sequence can actually be exercised and tested against.

## Flying a mission — the SOP phase sequence

The simulator's procedure state machine walks through the same phase sequence a real
before-flight checklist would:

```
OFF → PRE_START → CRANKING → IDLE → WARM_UP → MAG_CHECK → TAXI → TAKEOFF → CLIMB → CRUISE
  → DESCENT → COOL_DOWN
```

Some transitions are gated on real, checkable conditions rather than just "operator clicked
next" — for example, take-off is refused until the mag-check has actually passed and oil
temperature has actually reached its published minimum. Take-off power itself is time-limited (a
published 5-minute cap), and the model auto-transitions out of it rather than letting an operator
hold full power indefinitely against physics that says otherwise.

## Three ways to fly, one of them deliberately locked

- **Sim**: an operator directly commands a live, in-browser instance of the plant model above —
  sets environment, moves the throttle, injects a catalogue fault, steps through the phase
  sequence.
- **Logger**: pure playback of a previously recorded flight — either a real recording, or an
  export from a Sim session. Never commandable.
- **Live**: watches a persistent engine process running as its own service, reachable the same
  way a real telemetry link would be — but every control is disabled. The project's own reasoning
  for that lock, verbatim: a monitoring dashboard that *can* start or shut down a real engine over
  a network link "does not become safe just because nothing real is plugged in yet" — the same
  reason a genuine dual-lane engine management system doesn't take throttle commands from a
  laptop.
