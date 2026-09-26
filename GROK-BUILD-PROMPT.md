# Build instructions for Grok Build

Paste this whole file as the build prompt. It says what to build, what to build it FROM, and the
rules to build it BY.

## What to build

An interactive, single-page (or lightly multi-page) web app that teaches a newcomer — a
teammate, a judge, an interviewer — what DARPAN is, why it exists, and how each of its pieces
works. Think "onboarding explainer" crossed with "interactive architecture diagram," not a
static slide deck. Someone using it should come away able to answer: what problem is this
solving, what is the core idea, how does the simulator work, how does the ML layer work, how
does degradation/RUL prediction work, how does the 3D digital twin work, and what does each real
page of the actual app do.

## Source material

Everything needed is in `content/*.md` (prose, organized by topic) and `data/*.json` (structured
data for interactive widgets). Read all of it before building navigation — the sections below map
directly to `content/` filenames.

## Required sections / nav

1. **The problem** (`content/00-problem-and-solution.md`) — render the verbatim SIH26054 problem
   statement (it is marked verbatim in the source on purpose — do not paraphrase it further),
   then DARPAN's solution approach and core idea.
2. **Architecture tour** (`content/01-architecture-tour.md` + `data/component-tree.json`) — an
   **expandable/collapsible tree component** driven by `component-tree.json`, not a static code
   block. Clicking a node should show its description. Render the data-flow pipeline
   (also in that JSON) as a horizontal step diagram.
3. **Engine & simulator theory** (`content/02-engine-and-simulator-theory.md`) — the physics:
   ISA atmosphere, the forward plant model, root-factor fault injection, SOP phases. This is the
   densest section — use collapsible subsections rather than one long scroll.
4. **ML & detection** (`content/03-ml-and-detection.md` + `data/ml-results.json`) — build a real
   **comparison chart** (bar chart or similar) of physics baseline vs Isolation Forest vs
   Gradient Boosting from `ml-results.json`'s actual numbers. Render the ablation table as a real
   table, not prose.
5. **Degradation & RUL** (`content/04-degradation-and-rul.md`) — explain Theil-Sen, multi-scale
   windows, and the P10/P50/P90 band concept. A small interactive diagram helps here (even just a
   slider showing how the uncertainty band widens with horizon) but isn't required.
6. **The 3D digital twin** (`content/05-digital-twin-3d.md`) — explain the two-renderer
   architecture and the live-telemetry-to-color mechanism.
7. **Fault catalogue** (`data/faults-catalogue.json`) — a **filterable/sortable table**: which
   faults are trainable vs not, and why. This is one of the most concrete "how honest is this
   project" artifacts in the whole corpus — make it genuinely browsable, don't bury it in prose.
8. **Glossary** (`content/06-glossary.md`) — a searchable term list (CHT, EGT, RUL, FADEC, etc).
9. **Honest limits** (`content/07-honest-limits-and-backlog.md`) — what's real vs estimated vs a
   declared, permanent limit. This section is a DELIBERATE FEATURE of the real project (see the
   rules below) — do not soften it into generic "future work" language.

## Rules — read before writing a word of copy

- **Never invent a number, a test result, or a capability that is not in `content/` or
  `data/`.** If a figure that isn't there would be nice to have, leave a citation-needed style
  note instead of guessing. This mirrors the real project's own "Provenance Rule," explained in
  `content/07-honest-limits-and-backlog.md`, and it is the single most important instruction in
  this file.
- **Don't upgrade an estimate into a fact.** The source material tags several numbers
  `DARPAN-ESTIMATE` or "transferred from a related engine, not the one actually modelled." Keep
  those qualifications visible in the UI — a small badge or tag is enough — rather than smoothing
  them into confident, unqualified prose.
- **Don't claim more coverage than the source claims.** Example: the ML model is trainable on 11
  of 24 catalogued faults. Say "11 of 24, and here's why," not "detects engine faults" generically.
- **Tone**: technically confident, plain language, no hype. The real project's own writing style
  (visible throughout `content/`) states its limits as plainly as it states its results — match
  that register, don't polish it into marketing copy.
- **Depth over breadth on request**: it's fine — good, even — to let a user expand into real
  depth (an actual formula, an actual test count) rather than staying at a one-line summary
  everywhere. The audience is technical.

## Suggested (not mandatory) shape

A single-page app with a persistent nav across the 9 sections above, each a scrollable panel. No
login, no backend needed — static content plus client-side interactivity (expand/collapse,
filter/sort, a chart library for the ML comparison). A dark theme suits an engine-instrumentation
subject but isn't required.
