# Architecture tour

DARPAN is a pnpm monorepo — one package's job per folder, a strict one-way dependency graph, no
package importing "sideways" or "up" into something that depends on it.

## The repo tree

```
darpan/  (package.json name: "aero-pdm")
├── shared/                    — types, constants, and the physics plant model
│   └── src/plant/             — isa.ts, rotax916-plant.ts, envelope.ts, lag.ts, climb.ts,
│                                 interp.ts, engine-baseline.ts — the forward model everything
│                                 else is built on (see 02-engine-and-simulator-theory.md)
├── packages/
│   ├── simulator/             — mission generator, fault injection, the SOP procedure
│   │                             state machine, sensor-realism (noise, quantisation, drift)
│   ├── features/              — turns raw snapshots into residuals + persistence-confirmed
│   │                             deviations + the sensor-vs-engine instrumentation split
│   ├── decision/               — the physics-only fault detector, RUL/degradation-rate
│   │                             estimation, the "digital thread" evidence-chain logger
│   ├── knowledge/               — baseline curves, sensor registry
│   ├── ingestion/               — pull-based replay bus (stands in for a real telemetry link)
│   ├── engine-host/             — a standalone process that runs ONE live engine instance and
│   │                             streams it over Server-Sent Events, so every browser tab /
│   │                             machine watching "LIVE" sees the exact same engine
│   └── ui/                      — the React/Vite dashboard — gauges, the 3D twin, analysis
│                                 pages, the degradation log, mission reports
├── data/
│   ├── corpus/                 — vendored OEM/research source files, checksummed so they can
│   │                             never be silently hand-edited
│   └── authored/                — DARPAN's own calibrated values, each carrying a provenance
│                                 tag explaining where the number came from
├── tools/                       — gen-corpus.ts (the only code allowed to read data/ directly
│                                 and turn it into typed constants), export-dataset.ts (builds
│                                 the ML training set), and similar generators
├── ml/                          — train.py / ablate.py — the gradient-boosting model and the
│                                 experiment that checks it isn't cheating
└── docs/                        — problem statement, deliverable tracker, deployment roadmap,
                                  per-page design plans
```

## The engineering rule behind that structure

The project's own build mandate, quoted directly: *"Load every constant from the JSON data
files. Zero hardcoded engine numbers... If a number is not in the data files, it does not exist
yet — add it to the data file with a provenance tag, then read it."* In practice this means: no
`.ts` file anywhere in the codebase is allowed to contain a typed-in engine number. Every limit,
curve point, and threshold traces back to a JSON file under `data/`, and an automated test
re-runs the generator and fails the build if generated output and source data ever drift apart.

## The one-way package graph

```
shared  ←────────────── simulator ─────────→ engine-host
  ↑                          ↑                    ↑
  └──── features ──────── decision              (its own process — nothing imports it,
           ↑                                     the UI reaches it over HTTP so the
        knowledge ← ingestion                    engine could physically live on a
           ↑                                      different machine)
          UI ─────────────────────────→ (HTTP) engine-host
```

Nothing depends on `ui`. `simulator` has no UI dependency at all. `decision` depends only on
`shared`. This matters because it's what makes the physics model and the detector independently
testable, and independently *trustable* — the same forward model that generates a simulated
mission's "ground truth" is the same one the detector predicts against, so a detection result
can't quietly depend on some UI-layer fudge factor.

## The data-flow pipeline (what happens every tick)

```
SimulatorEngine (or the replay bus, reading a recorded/logged mission)
  → EngineSnapshot                    — one tick's worth of every sensor channel
  → FeatureEngine  → FeatureSnapshot  — residuals (observed − physics-predicted), plus both a
                                        lagged baseline and an un-lagged "instant" baseline
  → DecisionEngine → DecisionOutput   — system risk score, active alerts, and each alert's own
                                        evidence chain (which channels, what values, why)
  → the React UI, subscribed live
       ├→ forecast / RUL module → mission-ready duration, P10/P50/P90 time-to-limit
       └→ a labelled training-data recorder → one row per running tick, feeding the ML pipeline
```

Two independent instances of the *same* physics plant model run side by side inside the
simulator: one is the injected "truth" (with any fault's effect actually applied), the other is
the "predictor" the detector compares against (always healthy). This separation is deliberate —
collapsing them into one instance would make the residual always read zero, which would make
every downstream test pass for the wrong reason.

## Real pages, and what each one is for

| Route | What it's for |
|---|---|
| `/` | Home — engine-instrumentation-system-style dashboard: condition band, gauges, an AI-computed degradation score, SOP phase rail, trend sparklines |
| `/simulator` | The FADEC-style "lab console" — fly a mission by hand (throttle, altitude, OAT), inject a catalogue fault, step through the SOP phases, watch the engine react |
| `/engine3d` | The 3D digital twin — see `05-digital-twin-3d.md` |
| `/analysis` | Residual plots, per-cylinder view, an efficiency (fuel-consumption) panel, advisory history |
| `/degradation` | A per-component episode log — every stretch of time a subsystem spent outside its steady band, with an operator's own acknowledge/resolve actions kept separate from the machine's own verdict |
| `/mission-report` | A single mission's health report — exportable, and archivable to a capped local history |
| `/baselines` | The engine's own reference numbers, browsable per SOP phase, with a narrow set of editable anchor points that every derived curve then heals around |
| `/ml` | ML transparency — the real, recorded model comparison numbers, the ablation study, and every declared limit, rendered directly from a results file rather than recomputed live |
| `/training` | A library of recorded/exported flight CSVs, with fault labeling |
| `/help`, `/about` | Generated reference docs — an operator's manual and a system dossier, both rendered from the same source constants as the rest of the app, so they can't drift out of sync with it |

## Two operating "intakes," and a third that's read-only by design

- **Sim** — a live, in-browser simulated engine the operator actually commands: sets throttle,
  altitude, injects faults, steps through startup/taxi/takeoff/climb/cruise/descent. This is
  where a mission is *authored*.
- **Logger** — pure playback of a previously recorded file (either a real flight recording, or a
  Sim session exported to CSV). Never commandable — it only replays what already happened.
- **Live** — watches a real engine process running elsewhere (the `engine-host` service) over a
  network link. Deliberately **read-only, always** — even though nothing physical is plugged
  into it yet. The reasoning, stated directly in the project's own log: *"this software does not
  command whatever answers at the link, it only watches... a dashboard that can start or shut
  down a real engine over a link does not become safe just because nothing real is plugged in
  yet."*
