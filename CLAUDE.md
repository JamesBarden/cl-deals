# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

There is **no application source code here**. This repo is the state store and output log for a Claude Code scheduled routine that scans Craigslist for underpriced listings. The scanning logic lives in the routine prompt at https://claude.ai/code/routines/trig_016RMx4nmXZzhaJsPcfysMME — not in any committed file. If you are looking for "the code that does the scanning," it does not exist on disk; edits to behavior are made by editing the routine prompt in the Claude Code UI.

What lives here:
- `seen-listings.json` — dedup state, keyed by Craigslist listing ID. Each entry: `{ first_seen, title, price, category }`. Append-only in practice; the routine reads it to skip listings already evaluated.
- `scan-log.json` — append-only array of every listing the routine evaluated, with the per-criteria pass/fail breakdown. Powers the dashboard. One entry per *unique* listing (same dedup as `seen-listings.json`). See "Scan log schema" below.
- `index.html` — static dashboard (vanilla JS) that fetches `scan-log.json` and renders a sortable, filterable grid. Designed to be served by GitHub Pages from the repo root.
- `digests/digest-YYYY-MM-DD-HH.md` — one file per run **only when qualifying deals are found**. State-only runs produce no digest.
- `subscribers.json` — email routing config (see below).
- `README.md` — points humans at the routine.

## Scan log schema

Each entry in `scan-log.json` is appended once per listing — not once per run. Shape:

```json
{
  "id": "7930290311",
  "scanned_at": "2026-05-06T16:04:00Z",
  "url": "https://boston.craigslist.org/...",
  "region": "boston",
  "category": "photo",
  "title": "Canon 5D Mark IV body",
  "description_excerpt": "first ~300 chars of the listing body",
  "posted_at": "2026-05-06T14:00:00Z",
  "listing_price": 800,
  "market_price": 2500,
  "condition_factor": 0.65,
  "comp_source": "MPB / KEH / eBay sold",
  "savings": 1700,
  "savings_pct": 0.32,
  "criteria": {
    "posted_within_30d": true,
    "posted_within_24h": true,
    "under_40pct_market": true,
    "savings_over_200": true,
    "scam_screen_passed": true
  },
  "qualified": true,
  "notes": "optional model commentary, especially for near-misses"
}
```

`savings_pct` is `listing_price / market_price` (i.e. % of market — lower is better; `under_40pct_market` is true when this is ≤ 0.40). Unknown numeric fields should be `null`, not omitted, so the dashboard renders blanks instead of breaking sorts.

## The qualifying filter (encoded in the routine, surfaced in digests)

A listing qualifies only if **all** hold:
- Asking price ≤ 40% of estimated used market value
- Absolute savings ≥ $200
- Posted within the last 24h
- Passes scam screen (notably: ≥30-word description for items >$200; no deposit/Zelle/wire/shipping/overseas signals)

The condition factor (×0.50 good, ×0.65 like-new, etc.) and market reference comps are documented inline in each digest's "Methodology check" section. Comp sources by category: photo → MPB / KEH / eBay sold; furniture → AptDeco / Chairish / 1stDibs; electronics → eBay sold / Swappa; instruments → Reverb sold listings (the canonical source) / eBay sold; bikes → BicycleBlueBook / pinkbike buy-sell / The Pro's Closet / eBay sold. Match the digest format if you write one by hand.

## Regions and categories

Per `subscribers.json`:
- Regions: `boston`, `daytona`, `newhaven`
- Categories: `photo`, `electronics`, `furniture`, `instruments`, `bikes`
- A subscriber receives a deal only if it matches **all** their criteria (region AND category). `["*"]` matches all values in that field.

The README still says "Boston only" — it is out of date relative to subscribers.json. Trust subscribers.json.

## Commit message convention

The routine commits with this exact shape — match it for any automated/scripted run, including state-only runs:

```
Run YYYY-MM-DD HH:MM UTC: N deals, X scanned[, took Mm Ss] (state only)
```

`(state only)` is omitted when a digest was produced. Examples from history:
- `Run 2026-05-05 14:05 UTC: 0 deals, 162 scanned, took 14m 32s (state only)`
- `Run 2026-05-05 02:55 UTC: 1 deal, 155 scanned`

For non-routine commits (config, doc, subscriber edits), use a normal short imperative subject — see `bf849ca`, `8f31e7c`.

## Editing guidance

- **Subscriber changes** are the most common manual edit. Validate region/category values against the enums above; the routine has no schema validation.
- **Do not hand-edit `seen-listings.json` or `scan-log.json`** unless removing a known-bad entry. Both are routine-managed and grow over time.
- **Do not delete old digests.** They are the historical record of what the routine surfaced.
- **No build, no tests, no lint.** There is nothing to run locally. The only "execution" is the scheduled routine.
- **Commit and push after every change.** When you modify any file in this repo (subscribers, README, CLAUDE.md, manual digest edits, etc.), commit the change with an appropriate message and `git push` before ending the turn. Use a short imperative subject for human-driven edits (e.g. `update subscribers`, `fix daytona region typo`); reserve the `Run YYYY-MM-DD HH:MM UTC: ...` format for routine-style runs only. Do not batch unrelated edits into one commit.
