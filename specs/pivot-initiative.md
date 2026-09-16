# Spec: Initiative dimension on Pivot

Status: Implemented
Related file: `display-dashboard.html` (`PV_INITIATIVES`, `pvExtractInitiative`,
`pvNormalizeForMatch`, `PV_DIMS.initiative`, `pvPool`)

## Problem

Emily asked whether Pivot could split performance by Deal Drop/Strike
Sale vs. other initiatives. The earlier Benchmarks-tab work
(`specs/benchmarks-tab.md`) flagged Pivot as blocked: its dimensions
come from BigQuery's `Creative Offer Name`, a different ID space from
the crosstab's `Creative ID`, with no established mapping between them
— the same problem that limits Asset Links' Cloudflare matching.

Emily clarified: the initiative label is actually embedded **inside**
the Creative Offer Name string itself (e.g. "Do the Most", "Deal Drop",
"Strike Sale"). That changes the picture entirely — this isn't a join
across two datasets, it's parsing a string Pivot's rows already carry.
The earlier blocker doesn't apply here.

## Decisions confirmed with Emily

- **Vocabulary**: exactly 3 known initiative labels — "Do the Most",
  "Deal Drop", "Strike Sale". Anything else falls into a generic
  "Other" bucket rather than being guessed at or expanded on
  speculatively.
- **Match position**: case-insensitive substring match **anywhere** in
  the Creative Offer Name (not restricted to a prefix).

## Implementation

`pvExtractInitiative(creativeOfferName)`:
1. Normalizes the name via `pvNormalizeForMatch` — lowercases it and
   collapses hyphens/underscores into spaces (this codebase's own
   naming conventions, per `CLAUDE.md`, use underscores heavily — e.g.
   `CRKT [job#] [Quarter] [Description]_[Platform]_[Dimensions]` — so
   "Deal_Drop" or "Deal-Drop" still match "Deal Drop" without expanding
   the 3-item vocabulary itself).
2. Checks the normalized string for each of the 3 needles in order,
   returns the first hit's label, or `PV_INITIATIVE_OTHER` ("Other") if
   none match.

`initiative` was added to `PV_DIMS` as a `creativeOnly:true` dimension
(same restriction as the existing `creative` dimension — Initiative can
only be derived where a Creative Offer Name exists, i.e. creative-grain
rows, not stage-level rollups). `pvPool()` attaches `.initiative` to
each row when `pvState.src === 'creatives'`, computed fresh from
`r.creative` on every call — via `.map()` into new row objects, not
mutating the cached rows other tabs (Analytics, Creatives) also read
from. Every other part of the pivot machinery (row/column selection,
grouping, sorting, cell rendering) already keys off `r[rowKey]`/
`r[colKey]` generically, so `initiative` slots in as a normal dimension
with no further special-casing needed — the two places that *do*
special-case a dimension key (`rowKey==='creative'` → run it through
`offerLabel()`) correctly fall through to the plain value for
`initiative`, since "Deal Drop" etc. are already human-readable.

## Known limitation

No-delimiter concatenations (e.g. a hypothetical "StrikeSale" with
zero separator between the words) are **not** matched — deliberately.
Cricket's actual naming conventions always delimit tokens with spaces,
hyphens, or underscores (see `CLAUDE.md`'s file/creative naming
patterns), so this isn't a realistic case to guard against, and adding
camelCase-boundary splitting to catch it would risk false-positive
matches on unrelated names elsewhere in the string.

## Verification

`pvExtractInitiative` was run via `osascript -l JavaScript` against 8
hand-built cases before wiring anything to the DOM: a natural-language
name with "Do the Most" embedded, an all-caps "DEAL DROP" with hyphen
separators, an underscore-delimited real-world-style Creative ID with
"Deal_Drop" buried mid-string, a name with no initiative keyword at all
(correctly "Other"), an exact-match lowercase string, and empty/null
input (both correctly "Other"). 7 of 8 passed exactly as expected; the
8th (the deliberately-unrealistic no-delimiter "StrikeSale" case) is
the documented limitation above, not a defect.
