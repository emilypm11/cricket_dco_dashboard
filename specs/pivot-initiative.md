# Spec: Initiative & Offer Type dimensions on Pivot

Status: Implemented (redesigned 2026-09-16 against real trafficker data —
see "Redesign" below; original text-matching approach is superseded)
Related files: `display-dashboard.html` and `index.html` (kept identical —
`PV_INITIATIVES`, `PV_OFFER_TYPES`, `pvExtractInitiative`,
`pvExtractOfferType`, `pvNormalizeForMatch`, `pvWordsOf`,
`pvContainsPhrase`, `PV_DIMS.initiative`, `PV_DIMS.offerType`, `pvPool`)

## Problem

Emily asked whether Pivot could split performance by Deal Drop/Strike
Sale vs. other initiatives. The earlier Benchmarks-tab work
(`specs/benchmarks-tab.md`) flagged Pivot as blocked: its dimensions
come from BigQuery's `Creative Offer Name`, a different ID space from
the crosstab's `Creative ID`, with no established mapping between them.

Emily clarified: the initiative label is embedded **inside** the
Creative Offer Name string itself, so this isn't a cross-dataset join —
it's parsing a string Pivot's rows already carry.

## First attempt and what was wrong with it (2026-09-16, superseded same day)

The first version matched 3 literal needles — "do the most", "deal
drop", "strike sale" — as case-insensitive substrings, with hyphens/
underscores normalized to spaces. It shipped, then **every single row
showed "Other"** in production. The root cause, found only after
reading Emily's real trafficker file (`Argonaut - Cricket DCO
Trafficker (2026).xlsx`, "Creative Offer Name" column): real names are
**camelCase-squashed with zero delimiter between words** — e.g.
`Q3-DoTheMost-Offer-4Linesfor$25/mo.each-...` — not
"Do The Most" or "Do-The-Most". The original spec had explicitly called
this exact pattern "deliberately unrealistic" and declined to guard
against it. That assumption was wrong, confirmed by real data, and is
retracted here rather than left standing.

## Redesign, based on real data

Reading actual rows from the trafficker file's "Creative Taxonomy" (AD),
"Creative Campaign Initiative" (AE), and "Creative Offer Name" (AJ)
columns revealed the real structure. "Creative Offer Name" follows a
hyphen-delimited pattern:

```
[Quarter]-[Campaign Initiative]-[Offer Type]-[Messaging]-...
```

e.g. `Q3-DoTheMost-Offer-4Linesfor$25/mo.each-4/$25-None-DCOText-Led_None_4/$25`
or `Q3-BTS-CricketDealDrop-FreePhone-None-SamsungA37-SamsungA375G_...`.

Critically, **Campaign Initiative and Offer Type are independent axes**
— a real row can be BTS+Offer, BTS+CricketDealDrop, Do The Most+Brand,
Do The Most+CricketDealDrop, etc. This matches exactly what Emily
described: *"there is Do the Most, BTS, but then Cricket Deal Drop is
sometimes part of Do the most and BTS"* — Cricket Deal Drop isn't a
4th initiative bucket, it's a value on a second, orthogonal dimension.

### Decisions confirmed with Emily

- **Initiative** = Do the Most / BTS / Other — matches the real
  "Creative Campaign Initiative" column's actual values in this file.
  Deal Drop and Strike Sale are removed from this dimension entirely.
- **Offer Type** (new, separate `PV_DIMS` entry) = Cricket Deal Drop /
  Brand / Offer / Apple / Other — an independent dimension, combinable
  with Initiative in Pivot's Rows/Columns pickers (e.g. Rows=Initiative,
  Columns=Offer Type). "Strike Sale" is folded into the "Cricket Deal
  Drop" label here too, carrying over Emily's earlier decision that the
  two are the same thing in practice, even though "strike sale" hasn't
  actually appeared in the real data checked so far.
- **Static vs. Animated, and device model, explicitly deferred.** Both
  only exist in the trafficker file's "Creative Taxonomy" column, which
  is **not** present anywhere in Pivot's BigQuery data (confirmed
  absent from "Creative Offer Name" itself — "Animated"/"Static" is a
  token Creative Taxonomy has that Creative Offer Name does not).
  Getting it into Pivot would need a new lookup-join capability Pivot
  doesn't have today (parallel to the crosstab's Lookup Table feature,
  but keyed on Creative Offer Name instead of Creative ID) — Emily
  chose to hold off on that scope increase for now.

## Implementation

Still **needle/word matching, not strict positional splitting** —
deliberately. A rigid parser that always expects exactly this dash
pattern would produce garbage on creative names that don't follow it,
e.g. this same file's own static `AN_CREATIVES` fallback data ("Standard
Template – Cricket pricing = simple, transparent."). Word/substring
matching just falls through to "Other" for those instead, which
degrades safely — same principle as the original spec, just with a
corrected normalizer and vocabulary.

`pvNormalizeForMatch(s)`:
1. Splits camelCase word boundaries **first** — `.replace(/([a-z])([A-Z])/g, '$1 $2')`
   — so `"DoTheMost"` → `"Do The Most"`, `"CricketDealDrop"` →
   `"Cricket Deal Drop"`. `"BTS"` (all-caps, no lowercase→uppercase
   transition) passes through unchanged.
2. Lowercases, then collapses hyphens/underscores/whitespace to single
   spaces.

Matching moved from a raw substring search to **whole-word matching**
(`pvWordsOf` splits the normalized string into a word array;
`pvContainsPhrase` checks for a needle's words as a contiguous
subsequence). This matters specifically for "BTS" — a bare 3-letter
acronym is exactly the kind of needle a plain `.includes('bts')` could
false-positive-match inside an unrelated longer word; comparing actual
word arrays avoids that (verified — see below).

`PV_DIMS` gained `offerType` alongside `initiative`, both
`creativeOnly:true` (same restriction as `creative` — only meaningful at
creative grain, not stage rollups). `pvPool()` attaches both
`.initiative` and `.offerType` in the same non-mutating `.map()` pass
used for `.initiative` before. Every part of the pivot machinery
(row/column pickers, grouping, sorting, cell rendering) already keys off
`r[rowKey]`/`r[colKey]` generically, so `offerType` slots in with zero
further special-casing, identically to how `initiative` already did.

## Verification

Re-verified via `osascript -l JavaScript`, extracting the actual
functions from **both** `display-dashboard.html` and `index.html`
(kept in sync) and running them against real strings copied from the
trafficker file:
- `Q3-DoTheMost-Offer-...` → Initiative "Do the Most", Offer Type "Offer" ✓
- `Q3-BTS-CricketDealDrop-...` → Initiative "BTS", Offer Type "Cricket Deal Drop" ✓
- `Q3-BTS-Offer-Transparency-...` → Initiative "BTS", Offer Type "Offer" ✓
- `Q3-DoTheMost-Brand-...` → Initiative "Do the Most", Offer Type "Brand" ✓
- An unrelated static-template name → both "Other" ✓
- A hypothetical Apple row → Offer Type "Apple", Initiative unaffected ✓

Word-boundary safety for "BTS" was verified separately (`Nabts-...` and
`AbtsInc-...`, where "bts" is embedded inside a longer word with no
standalone token, both correctly resolve to Initiative "Other"; a real
standalone `BTS` token still matches) — an initial version of this test
was itself flawed (the test sentence accidentally contained "bts" as a
genuine separate word elsewhere), caught and corrected before treating
the result as a pass.
