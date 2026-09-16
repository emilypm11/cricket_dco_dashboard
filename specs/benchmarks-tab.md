# Spec: Benchmarks tab

Status: Implemented (grouping made configurable 2026-09-16 — see
"Configurable Group By" below)
Related file: `display-dashboard.html` (Benchmarks tab — `BM_METRIC_DEFS`,
`bmState`, `bmGroupDimDefs`, `bmRowDimValue`, `bmAggregateGroups`,
`renderBenchmarks`)

## Problem

The reference Paid Social dashboard (PCE) has a dedicated Benchmarks tab:
one row per Publisher × Funnel × Ad Format, with sortable Impressions /
Orders / CVR columns. Display's dashboard had the *math* for a Benchmark
Group (`specs/performance-scoring.md`'s Publisher × Funnel × Ad Type
median) but no dedicated view for it — it only surfaced as extra columns
on the per-creative Performance Data table.

## Decision: pooled rollup, not the median

`performance-scoring.md`'s Benchmark Group value is a **median** across
creatives — used to judge one creative against its peers. The reference
PCE screenshot's Benchmarks tab shows different numbers: large pooled
totals (e.g. 17.2M impressions, 990 orders) that only make sense as
**summed raw counts across the whole group**, not a median of per-creative
rates.

So this tab computes a second, complementary rollup: `bmAggregateGroups`
groups the same normalized crosstab rows by Publisher × Funnel × Ad Type
only (one level up from `psAggregateScoringRows`'s Creative × Publisher ×
Funnel × Ad Type), sums raw counts, and computes every rate once from
those sums — same "sum first, divide once" rule as the rest of Performance
Scoring. The per-creative *median* benchmark still lives unchanged on
Performance Data's `Bench ___` columns; this tab doesn't replace it.

## Scope

- Data source: the same uploaded crosstab (`PS_DATASETS`) as Performance
  Data — independent Publisher/Funnel/Ad Type filters and its own date
  range, not shared state with the Performance Data tab's filters.
- Value columns are togglable via metric pills (same interaction pattern
  as Pivot's `pvMetricPills`) — default Impressions/Orders/CVR to match
  the reference screenshot, with CTR/LPVR/TDOR/VCR100%/VVR25%/Rate of
  Completion/Social ER available too.
- A group is flagged low-confidence (visible via a `*` and tooltip) when
  fewer than 2 distinct creatives fall into it — same threshold
  `performance-scoring.md` uses for its median benchmark.
- No BigQuery involvement — this tab is empty (shows an upload prompt)
  until a crosstab is uploaded on the Performance Data tab, consistent
  with "BigQuery stays primary, crosstab supplemental" for this whole
  reskin.

## Configurable Group By (2026-09-16)

Emily asked whether performance could be split by Deal Drop/Strike Sale
vs. other initiatives. That "Initiative" dimension doesn't exist in the
crosstab's own required/optional columns, and doesn't exist anywhere
else in this codebase yet — it's not currently tracked as clean data,
per Emily (it lives in the trafficking matrix, unformatted). The one
place it *can* reach this dashboard today is the **Lookup Table**
upload (Creative ID + arbitrary extra columns) — the same mechanism
Asset Links' suggestion-matching already uses.

Rather than hardcoding "Initiative" as a fourth fixed grouping key, the
Group By row was generalized: `bmGroupDimDefs()` returns the 3 base
crosstab dimensions (Publisher, Funnel, Ad Type) plus one entry per
column in whatever Lookup Table is currently loaded (`PS_LOOKUP.extraCols`)
— so "Initiative" becomes a toggleable Group By chip automatically the
moment a Lookup Table with that column is uploaded, with no code change
needed, the same way any other lookup column (Concept, Campaign Type,
Region, ...) would. `bmRowDimValue(r, dim)` resolves a row's value for
a given dimension — straight from the row for the 3 base dims, or
joined by Creative ID from `PS_LOOKUP.byId` for anything else — joined
at render time only, same rule as the rest of Performance Scoring.

At least one Group By dimension must stay selected at all times (same
"keep at least one" guard as the Values pills); grouping by zero
dimensions would collapse every row into a single meaningless total.

**Not extended to Pivot.** Pivot's dimensions (`PV_DIMS`: Stage/Quarter/
Market/Creative) come from a completely different data source — the
BigQuery-driven `AN_STAGES`/`AN_CREATIVES` model, keyed on BigQuery's
own "Creative Offer Name," not the crosstab's "Creative ID." There's no
established mapping between those two ID spaces anywhere in this
codebase (this is the identical mismatch already documented in
`specs/asset-links.md` for Cloudflare image matching) — joining a
Lookup Table's Initiative column onto Pivot's BigQuery rows would
require inventing a new, untested name-matching bridge, not reusing
anything that already works. Flagged to Emily rather than built.

## Verification

`bmAggregateGroups` was checked against a hand-built 3-row fixture via
`osascript -l JavaScript` before wiring it to the DOM: two Video creatives
under PubX/F1 pooled correctly (18,000 imps, 300 clicks → 1.667% CTR;
150 LPVs, 15 orders → 10% CVR; one had no Social Engagements column, the
other did, and the group's Social ER still computed from the columns that
existed) and correctly flagged low-confidence at 1 creative for a third,
separate Static group.

The Configurable Group By change was separately verified against a
4-creative fixture with a synthetic Lookup Table (`Initiative`: Deal
Drop ×2, Strike Sale ×1, BAU ×1): grouping by Initiative alone correctly
pooled the two Deal Drop creatives (18,000 imps, 20 orders, 8.7% CVR)
and correctly kept Strike Sale/BAU separate, each flagged low-confidence
at 1 creative; grouping by Publisher + Initiative together produced the
correct 2-key composite groups.
