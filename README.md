# DARPAN Explainer — teaching corpus

This repo is **not the DARPAN project itself**. DARPAN (SIH26054, team Shadows) is a private
repository: an AI-enabled digital twin for health monitoring, fault prediction and
mission-reliability of aero piston engines in MALE UAVs. This repo is a **curated, accurate
mirror of DARPAN's logic and theory**, written so an AI app-builder (Grok Build) can construct
an interactive learning/onboarding panel from it — without needing access to the private source
repository itself.

## What's here

| Path | Contents |
|---|---|
| [`GROK-BUILD-PROMPT.md`](./GROK-BUILD-PROMPT.md) | The literal build spec. Paste this into Grok Build as the instruction. |
| `content/` | Eight markdown files: problem → architecture → engine/simulator theory → ML → degradation/RUL → 3D twin → glossary → honest limits |
| `data/` | Structured JSON (architecture tree, ML results, fault catalogue) — for building real interactive widgets, not just rendering prose |

## Where this content came from

Every fact, number and formula in this repo was pulled directly from the real DARPAN source —
its project log, its physics plant model source code, its recorded ML pipeline results, and its
verbatim SIH26054 problem statement — not summarized from memory or invented to fill a section.
Where the source project itself flags a number as an estimate rather than a published fact, that
flag is carried over here rather than smoothed away.

This matters more than it might look like it does: DARPAN's own engineering discipline is built
around never restating an estimate as if it were an established fact — a rule it calls out
directly in its own project documentation: *"do not launder an estimate into a fact by restating
it without its tag."* A teaching corpus about that discipline which quietly broke it while
explaining it would be its own kind of joke. So this corpus keeps every provenance flag it found,
and says plainly, in `content/07-honest-limits-and-backlog.md`, exactly which numbers are
published engineering data and which are DARPAN's own calibrated estimates.

## Using this with Grok Build

1. Give Grok Build access to this repo — it's public, no auth needed.
2. Feed it [`GROK-BUILD-PROMPT.md`](./GROK-BUILD-PROMPT.md) as the build instruction.
3. Point it at `content/` and `data/` as its source material.
