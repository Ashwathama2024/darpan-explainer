# The problem, and DARPAN's answer to it

## The problem statement (SIH26054), verbatim

> Recorded exactly as issued, typos included. A requirement paraphrased is a requirement
> quietly renegotiated — so this section is copied, not summarized.

### Background

Medium Altitude Long Endurance (MALE) UAV are increasingly being deployed for Long-duration
intelligence, surveillance, reconnaissance (ISR).

Communication relay maritime surveillance and strategic defence missions Reliability and
availability of propulsion systems are critical for mission success because piston-engine
failures during flight may lead to mission abort, asset loss, or unsafe recovery conditions.

Conventional engine monitoring systems used in UAVs are primarily threshold-based and reactive in
nature. These systems generally indicate failures only after abnormality has already occurred.
Present approaches also have limited capability to estimate remaining useful life (RUL) predict
degradation trends, or simulate mission-wise engine behavior under varying environmental and
operating conditions.

A Digital Twin (DT) framework for aero piston engines can significantly improve predictive
maintenance, operational reliability, mission planning, and life cycle management by creating a
continuously synchronized virtual representation of the physical engine using real-time sensor
data, physics-based models and AI/ML techniques.

The proposed problem aims to develop an indigenous Digital Twin framework suitable for
deployment in MALE UAV ground control and health monitoring architecture. The solution should
support real-time engine state estimation, anomaly detection, degradation tracking, fault
prediction, and mission replay capability.

### Description

Develop a scalable and modular digital Twin System for an aero piston engine used in MALE UAV
applications. The system shall create a real-time virtual representation of the engine by
integrating:

- Engine sensor data
- Thermodynamic behavior models
- Engine performance maps
- Failure/degradation logic
- AI/ML based predictive analytics

The system should be capable of:

- Real-time engine parameter visualization
- Monitoring of engine health indicators
- Detection of abnormal operating conditions
- Predicting probable failures before occurrence
- Estimating degradation trends and Remaining Useful Life (RUL)
- Simulating engine behavior under different mission profiles and environmental conditions
- Supporting post-flight analysis and mission replay

### Expected Solution — six subsystems (A–F)

**A. Digital Twin Core Framework** — virtual engine model synchronized with live engine data,
modular architecture for future scalability, real-time data ingestion capability.

**B. Health Monitoring System** — continuous condition assessment and health indices for: RPM,
Cylinder Head Temperature (CHT), Exhaust Gas Temperature (EGT), Oil Pressure & Temperature, Fuel
flow, Vibration signatures, Battery/Alternator health, Injection timing parameters.

**C. Fault Detection & Predictive Analytics** — move from threshold-based to intelligent
predictive diagnostics for: misfire conditions, injector abnormalities, coking degradation,
lubrication issues, sensor drift/failure, combustion instability, overheating trends, abnormal
vibration patterns.

**D. AI/ML Layer** — adaptive learning for predictive diagnostics: anomaly detection algorithms,
RUL estimation, trend analysis, predictive maintenance recommendations.

**E. Simulation & Replay Capability** — replay of historical mission data, environmental
condition simulation, engine behavior simulation during high altitude, endurance missions,
hot-weather operation, rapid throttle transitions.

**F. Visualization Dashboard** — real-time engine health status, fault alerts, engine efficiency
trends, maintenance advisory, mission-wise health reports.

### Deliverables expected from teams

Functional prototype/software demonstrator · digital twin architecture design · engine
simulation model · AI/ML-based anomaly detection module · visualization dashboard ·
demonstration using simulated or real engine datasets · technical documentation and deployment
roadmap.

### Desired innovation areas

Physics-informed AI · Edge AI for UAV applications · lightweight onboard analytics · hybrid
thermodynamic + data-driven models · federated learning · explainable AI for fault diagnosis ·
secure telemetry architecture · autonomous maintenance advisory systems.

---

## DARPAN's core idea

Build a **physics-informed digital twin**: a forward simulation of a real, published aero
piston-engine's behavior, fed by real OEM engineering data wherever it exists, that runs
**alongside** a live or replayed telemetry stream. The twin predicts what a *healthy* engine
should be reading right now, given the current conditions (altitude, temperature, throttle,
rpm). The gap between that prediction and the actual reading — the **residual** — is the raw
signal every downstream layer works from: fault detection, degradation tracking, and RUL
estimation all start from "how far off is reality from the healthy-engine prediction," not from
a fixed threshold on a raw sensor value.

On top of that physics baseline sits a machine-learning layer (gradient boosting) trained to
catch what pure threshold logic misses — and it's shipped *alongside* the physics detector, not
instead of it. The physics path stays as the explainable fallback and safety gate; the ML path
is only kept because it's been proven, by a real held-out test and a feature-ablation study, to
catch substantially more faults earlier at the same false-alarm rate. See
`03-ml-and-detection.md` for the actual numbers.

### The stated differentiator: sensor drift is not an engine fault

A failing sensor and a failing engine look similar in raw data — both produce an abnormal
reading. DARPAN treats "is this instrumentation drift or an actual mechanical problem"
as a first-class question, judged *before* any specific fault is even considered. Concretely:
readings are split across two independent instrument lanes, and if one whole lane goes missing,
that is flagged as an **instrumentation anomaly**, never as a mechanical fault — because a
missing lane produces no corroborating signal on any other channel, which is exactly the
signature of an instrument problem rather than a physical one.

### Explainable, not a black box

Every fault advisory the system raises carries its own **evidence chain**: which channels were
read, what their residuals were, what prior state the decision depended on. That evidence chain
can be **replayed** later — re-derived entirely from its own recorded contents, with zero
reference to live state — and independently verified to produce the exact same advisory. This is
checked by an automated test suite, not just asserted in a document.

### Honesty about scope is treated as an engineering requirement, not a caveat

The project has one hard, self-imposed rule that shapes almost every other decision in it: never
state an estimated or assumed number as if it were a measured, published fact. A value with no
citation has to say so explicitly. This rule exists because of a real, documented failure mode
from an earlier, related project this one was forked from — see
`07-honest-limits-and-backlog.md` for that story, because it explains *why* DARPAN is built the
way it is, not just *what* it does.
