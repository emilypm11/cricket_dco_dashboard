# Spec: Asset Links

Status: Implemented
Related file: `display-dashboard.html` (Asset Links tab — `AL_LINKS`,
`AL_LOCAL_ASSETS`, `AL_CF_ASSETS`, `alSuggestionFor`, `renderAssetLinks`,
`renderAssetGrid`; preview column wired into `PS_BASE_COLS`)

## Problem

The reference PCE dashboard's Asset Links tab lets you link a creative
with no image/video to an existing Cloudflare asset (drag-and-drop or an
auto-suggested match) or upload a new file. Before building, two things
turned out to be less "just reuse what exists" than they first looked:

1. The Cloudflare matching this codebase already has (`cfFetchImages`,
   `cf-images.js`, `cf-sign-images.js`) is keyed to BigQuery's
   `Version`/`Creative Offer Name` — a completely different ID space
   from the crosstab's `Creative ID`. There is no automatic path between
   them, so **every crosstab creative starts out "missing an asset"**
   until linked here — this isn't a small edge case, it's the whole tab.
2. "Upload a new file straight to Cloudflare Images/Stream" needs a real
   upload endpoint with API credentials. This codebase only has list
   (`cf-images.js`) and sign (`cf-sign-images.js`) endpoints — no upload
   one — and building one isn't testable without Emily's own Cloudflare
   credentials wired into a new serverless function.

## Decisions confirmed with Emily

- **Suggestions are simple exact-match, not fuzzy scoring.** A creative
  gets a suggestion only when another creative that already has a
  linked asset shares an identical, non-empty value in **every** column
  of the uploaded Lookup Table (`PS_LOOKUP.extraCols`) — e.g. Concept and
  Creative Campaign if those are the columns Emily's lookup table
  happens to have. No weighted/partial-match scoring was invented. If no
  Lookup Table is loaded, there's nothing to match on, so no suggestions
  are offered at all (not a fabricated 100%-confidence guess).
- **New uploads reuse the existing local-fallback pattern** (the same
  approach Creative Library already uses when Cloudflare is
  unavailable: `<input type="file">` + IndexedDB blob storage), rather
  than a new Cloudflare upload endpoint. This works today with zero new
  credentials.
- **Separate IndexedDB database** (`crkt-asset-links`, not the existing
  `crkt-previews` Creative Library already uses) specifically so this
  feature's storage can never be wiped by Creative Library's "Remove
  All" manual-preview button, or vice versa. Link metadata itself
  (Creative ID → asset type/id/filename) is a small JSON blob in
  `localStorage` (`crkt_al_links_v1`), independent of both IndexedDB
  stores.

## Scope

- Missing-asset list: one row per distinct Creative ID in the selected
  crosstab dataset (`PS_DATASETS[daily|monthly|summary]`) with no entry
  in `AL_LINKS`, showing Publisher/Funnel/Ad Type, summed impressions,
  any Lookup Table columns as tags, and a suggestion chip when one
  exists — with its own Publisher/Funnel/Ad Type/search filters,
  independent of every other tab's filters.
- Asset browse pane: Cloudflare Images (`cfFetchImages`, raw list
  fetched once per hour or on explicit "Refresh Assets") merged with
  locally-uploaded files, searchable by filename, draggable onto a
  missing row's drop zone. Capped at 120 results shown at once (matching
  the reference's own "search to narrow" framing) — thumbnails are only
  signed (`cfSignIds`) for the assets actually rendered, not the whole
  Cloudflare library, to avoid needlessly signing hundreds of unused
  images.
- No video-format toggle — `cf-images.js` only calls Cloudflare's Images
  API, not Stream, so there are no video assets to filter to; a toggle
  that always showed zero results would be worse than not having one.
- Preview column added to the Performance Data table (leftmost, no
  header label) showing the linked thumbnail, a generic icon if linked
  but not yet resolved (e.g. an unsigned Cloudflare asset), or a plain
  dash if nothing is linked — reusing the exact same `AL_LINKS` data, so
  linking an asset here is immediately visible on Performance Data
  without extra wiring. Links load eagerly on sign-in (`alLoadLinks()`,
  `alLoadFiles()`, `alSignLinkedCfAssets()`), not only on first visit to
  the Asset Links tab, so Performance Data is correct even if a user
  never opens Asset Links this session.
- "Accept N Suggestions" bulk-applies every current exact-match
  suggestion in the visible missing list at once.

## Verification

`alGetCreativeSummaries`, `alSuggestionFor`, `alPreviewUrl`, and
`alUsageCount` were run via `osascript -l JavaScript` against a
synthetic fixture (4 creatives, one already linked, two sharing its
exact Concept/Creative Campaign, one with a different Concept, one with
no Lookup Table row at all) before wiring anything to the DOM: impression
summation across multiple rows per Creative ID was correct, the
matching creative was suggested for the two sharing Concept/Campaign,
and no suggestion was produced for the mismatched or lookup-less
creatives.
