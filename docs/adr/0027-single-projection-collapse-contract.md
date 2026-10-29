# ADR-0027 — Single-projection diff collapse contract with counted span markers

Status: Accepted (absorb of upstream v2.3.0, PR #92 — issues #88–#91, lane `a230`).
Upstream referents: `pi-better-edit@b92e0ec..0925f1a` (`31d67d8` #172, `71e0107` #174,
`ab06895` #175); upstream ADR-0024
(`narrow-p2-interior-exposure-cap-removed-diffs`, 2026-09-21) for the removal-cap
template. Downstream record: `src/edit-diff.ts` (`genDiff` contract comment),
`GLOSSARY.md` (`projection`, `source of truth`), `test/core/edit-diff.gap.test.ts`,
`test/core/edit-diff.projection-audit.test.ts`.

Number note: upstream ADR-0024 and downstream ADR-0024
(`changelog-prepush-guard-retired-gate-owns-boundaries`) are unrelated — the
number is taken on this line (see that file). This record covers the upstream
ADR-0024 decision that was previously behavior-ported but never recorded here
(the removal cap, via FU-1 conformance) together with the v2.3.0 collapse
contract that builds on its marker template.

## Context

`genDiff` renders the one diff the model sees. Two defects shared one root —
rows hidden behind a collapse while the coordinate cursors disagreed about how
many rows were hidden, so every anchor after the gap shifted:

1. Middle gaps of (context, 2×context] rows were sliced away markerless without
   advancing the cursors (upstream #166, fixed by #172).
2. At context 0 the large-gap branch took its tail with `slice(-0)` — which
   returns the whole array — so every middle-gap row rendered behind the marker
   while `skipMiddle` advanced past the same rows (upstream #170, fixed by
   #175, the #166 failure mode through a different branch).

Separately, removed runs rendered every deleted row unbounded (upstream
ADR-0024 decision 3 capped them behind a counted marker), and the two marker
kinds used different diction (`lines omitted` vs bare ` ...`), so a reader
could not tell what a count counted (unified by #174).

## Decision

`genDiff` is the single shared projection — preview, applied, and human
display render the same text with different context params — under this
collapse contract:

1. A middle (untouched) gap smaller than 2×context renders whole; a larger one
   collapses behind exactly one ` ... [N lines untouched] ...` marker, and both
   cursors advance past the exact hidden count N. At context 0 every non-empty
   middle gap collapses to that single marker. Bare ` ...` appears only at
   leading/trailing edges; added spans are never collapsed.
2. A removed run above `DIFF_REMOVED_CAP` renders head + tail rows around one
   ` - ... [N lines deleted] ...` marker, and the old-side cursor advances by
   the exact deleted count N — the upstream ADR-0024 template, with the diction
   unified from `lines omitted` to `lines deleted` so both marker kinds name
   the exact span they hide in one counted-span vocabulary.
3. Every marker's count equals its hidden span, and `servedRows` mirror exactly
   the rendered context/addition rows (position + hash).
4. Projection text is informational only: hidden untouched rows stay
   lease-covered and addressable through the served leases, and coordinates
   resolve through leases, never by re-reading a projection (GLOSSARY
   `source of truth`).

## Consequences

- Anchors after any gap resolve to their true content hash; the #166 desync
  class and its ctx-0 variant are closed by construction, pinned by the gap
  tests, the projection-audit corpus (gap-size × context, including the ctx-0
  leg), and the ctx-0 regression.
- The cap behavior (upstream ADR-0024 decision 3, ported via FU-1) is now
  recorded on this line for the first time; its P1/P2 exposure-gate halves
  (decisions 1–2) belong to the served-verification seam and are out of scope
  here — this ADR covers the rendering contract only.
- A future marker-diction change must keep the invariant: one counted marker
  per hidden span, count equals span, cursors advance by the count.
