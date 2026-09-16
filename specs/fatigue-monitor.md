# Spec: Creative Fatigue Monitor

Status: Implemented
Related file: `display-dashboard.html` (Fatigue Monitor tab — `fatOls`,
`fatCusum`, `fatAnalyzeSeries`, `renderFatigueMonitor`)

## Problem

`specs/performance-scoring.md` named this as Phase 2 with zero design:
*"Fatigue Monitor (Baseline/Current/ΔPct, 7-day rolling average, OLS
regression + confidence interval for decay detection, CUSUM changepoint
detection, Half-Life projection, Confidence Tier, Refresh Priority
Score)."* Unlike Benchmarks (math already spec'd, just needed a tab) or
the Scorecard (thresholds bolted onto an existing score), this needed
real statistical methodology decided from scratch — Emily explicitly
chose the fuller OLS + CUSUM approach over a simpler rolling-average
comparison.

## Decisions confirmed with Emily

- **Method**: OLS linear regression (day index → daily CTR or CVR) for
  gradual decline, combined with CUSUM changepoint detection for sudden
  step-changes, rather than a simple baseline-week-vs-current-week
  comparison.
- **State thresholds** (cumulative decline over the observed window):
  Healthy above -10%, Softening -10% to -20%, Fatiguing -20% to -40%,
  Dead beyond -40%.
- **Eligibility bar**: ≥7 days of data and ≥25,000 impressions (lower
  than the reference dashboard's own ≥14 days/50k — Emily's choice,
  evaluates Display creatives sooner given typically lower per-creative
  volume than paid social).
- **Confidence Tier bar** (separate, higher, from eligibility): ≥14 days
  AND ≥50,000 impressions AND a statistically significant OLS slope →
  High; anything meeting only the eligibility bar → Medium.

## Two bugs caught during testing, before wiring to the DOM

Both were found by hand-building synthetic day-series and checking the
math actually did what it was supposed to — not by guessing it was
probably fine.

1. **CUSUM's noise estimate was self-defeating.** The textbook approach
   estimates σ (noise) from the whole series' variance. For fatigue
   detection that's backwards: the shift you're trying to detect is
   itself sitting inside that variance calculation, inflating σ and
   making the detector *less* sensitive to real cliffs — a deliberately
   constructed 50%-drop-at-day-5 test case was missed entirely at first.
   Fixed by estimating σ from consecutive day-to-day differences via a
   robust median-absolute-deviation estimator instead, which is far less
   distorted by a single large shift. Re-tested: the same step-change
   was then correctly caught, and a separately-tested noisy-but-flat
   series still correctly produced no false alarm.

2. **CUSUM then over-fired on smooth linear decline.** After fix #1, a
   perfectly smooth 7-day linear decline (constructed with zero noise,
   specifically to test the "no discrete step" case) started tripping
   CUSUM's threshold too — mathematically not wrong (cumulative
   deviation from the series mean does grow monotonically along any
   trend), but it mislabels ordinary gradual fade as a sudden "shift on
   day X," which is misleading in the UI. Fixed by only trusting a
   CUSUM-proposed changepoint when a two-segment step model (mean before
   / mean after) actually explains the data meaningfully better than the
   single OLS line — specifically, its residual sum of squares must be
   under 70% of the line's. Re-tested against all four prior cases
   (real early step, real late step, noisy flat, smooth decline): the
   two real steps were still correctly accepted, and the smooth decline
   was now correctly rejected back to the trend-based reading.

The 70% cutoff is a chosen model-selection threshold, not a textbook
constant — it requires the step model to explain meaningfully more
variance before being preferred over the simpler trend line, as a guard
against overfitting a spurious changepoint to ordinary noise.

## Scope

- Requires day-level data — only `PS_DATASETS.daily` is used; Monthly/
  Summary crosstabs have no meaningful per-day granularity to analyze,
  and the tab shows an upload prompt if no daily crosstab is loaded.
- Unit of analysis: Creative ID × Publisher × Funnel × Ad Type (the same
  grain as the Composite Score's peer group) — a creative running on
  multiple platforms is evaluated separately per platform, since decay
  is a property of a specific execution context, not the raw ID alone.
- t-test significance for the OLS slope uses a standard t-table (df
  1–30, normal approximation beyond that) — not a from-scratch
  approximation of the t-distribution.
- Half-life: days from the window's start for the fitted trend line to
  reach 50% of its own starting value, capped at 365 days (longer than
  that isn't a meaningful "half-life," just a very slow decline) and
  only computed for a declining trend with a positive starting value.
- Refresh Priority (used to rank the Refresh Queue and pick the Top
  Fatiguing sparklines): `severityWeight × last-7-day impressions`,
  where severity is Dead=3 / Fatiguing=2 / Softening=1 — an explainable
  rank combining how bad and how much spend is actually at stake, not an
  opaque scoring formula.
- "Refresh Now" tile counts only Dead + High-confidence creatives — the
  narrowest, least ambiguous "act now" bucket — while the broader
  Refresh Queue list shows every flagged (non-Healthy) creative.
- Decay curve chart (right pane) shows the raw daily metric line, the
  fitted trend line, and a marker at the changepoint day when one was
  accepted.

## Verification

`fatOls`, `fatRobustSigma`, `fatCusum`, `fatStepModelSSRes`, and the full
`fatAnalyzeSeries` pipeline were each run via `osascript -l JavaScript`
against hand-built synthetic day-series before any DOM wiring — a
too-new series (correctly ineligible), a flat/stable series (correctly
Healthy), a smooth 14-day gradual decline (correctly Dead via the trend
method, High confidence, half-life within a plausible range), a sudden
16-day cliff (correctly Dead via the changepoint method, with the
changepoint day landing exactly where the synthetic data placed it), and
a low-volume-but-eligible series (correctly downgraded to Medium
confidence despite a severe decline, since Confidence Tier and state are
independent checks).
