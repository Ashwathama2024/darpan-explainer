# The 3D digital twin

DARPAN's 3D engine view actually contains **two separate renderers**, built for two different
jobs. Understanding why there are two, and which one a user actually sees, is the interesting
part.

## Renderer 1 — exact CAD wireframe (a contract-testing tool, not a visible page)

A research corpus the project was built from ships real CAD-derived engine geometry: roughly 20
named parts, described as ~193 exact edge polylines (not triangulated mesh surfaces — literal
line-segment edges taken straight from CAD solids). DARPAN has a renderer that draws this data
exactly as given, plus an invisible clickable "hit box" per part (since a one-dimensional line is
almost impossible to click precisely in a 3D scene).

What this renderer is actually *for*: the research corpus also shipped a reference HTML page
demonstrating this same geometry with its own fault-to-part and sensor-to-part bindings — except
that reference page is missing a data file it depends on, so it can't actually run standalone.
Rather than either (a) trying to reverse-engineer and rebuild that exact missing file, or (b)
ignoring the reference page entirely, DARPAN treats the reference page as a **pinned oracle**: an
automated test parses the reference page's own source directly, extracts its fault-binding and
sensor-binding tables, and asserts DARPAN's own bindings match it exactly. If either one ever
drifts from the other, the test fails and says so immediately — the contract is enforced by code,
not by a document nobody re-reads.

This exact-geometry renderer is **not currently mounted on any page a user sees** — its only job
today is anchoring that contract test.

## Renderer 2 — the stylized "holographic" twin (what's actually on screen)

The engine page users actually see is a hand-built, hand-positioned stylized twin: ~20 parts
assembled from primitive 3D shapes (cylinders, boxes, a few custom extruded profiles for things
like valve covers and the gearbox housing), each one positioned using real measurements where
they exist. Every part's placement carries an explicit tag: **published** (taken directly from a
real engineering drawing or manual), **derived** (calculated from two or more published numbers),
or **ESTIMATE** (DARPAN's own best guess where nothing is published) — visible on request, not
hidden.

A shared custom shader (used by every part) gives the "holographic" look: a fresnel rim-light
effect, a slow scanning band, and — the functionally important part — color mixing driven
directly by live engine health data.

## How live telemetry becomes a part's color

Each part reads the specific telemetry channels physically relevant to it — cylinders and heads
read cylinder-head-temperature and their own exhaust-gas-temperature probe; the oil tank/cooler
reads oil pressure and temperature; the turbo/intercooler/intake read manifold pressure and
manifold air temperature; and so on. Each of those readings is converted to a 0–1 "how close to
its limit is this" proximity figure using the exact same banding logic the 2D gauges elsewhere in
the app use — one shared function, not a separate copy for the 3D view. That proximity figure then
drives a green → amber → red color ramp on the part.

**Critically: a part with no fitted or relevant live reading is left a neutral, untinted grey —
never colored green.** This is the same "don't invent a healthy reading" discipline that runs
through the whole project: green is a specific claim ("this is measured, and it's fine"), and a
part that has nothing measuring it doesn't get to make that claim by default.

## Fault-to-part highlighting — and refusing to guess

When the detector's current leading fault candidate has a known part binding, that part
highlights. When it doesn't — which is true for roughly two-thirds of the full fault catalogue —
**nothing highlights**, and that's the deliberate default branch, not a missing feature. The
project's own stated reasoning: a twin that lights up a plausible-looking part for a fault it
doesn't actually have a real mapping for would send a maintainer to the wrong physical component,
which is worse than admitting it doesn't know.

## Interaction model

- **Single click** selects a part and opens an info panel (its fitted channels, its current
  degradation state, its RUL projection if one applies) — the camera does **not** move.
- **Double-click**, or an explicit "Fly in" button, animates the camera toward that part.
- Three view modes — holographic / x-ray / solid — change the shader's opacity balance.
- The whole view respects the operating system's "reduce motion" accessibility setting: every
  purely decorative animation (a spinning propeller, a scanning light band) turns off, while
  anything that's actually live data keeps updating.
