# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

There is **no application source code here**. This repo is the state store and output log for a Claude Code scheduled routine that scans Craigslist for underpriced listings. The scanning logic lives in the routine prompt at https://claude.ai/code/routines/trig_016RMx4nmXZzhaJsPcfysMME — not in any committed file. If you are looking for "the code that does the scanning," it does not exist on disk; edits to behavior are made by editing the routine prompt in the Claude Code UI.

What lives here:
- `seen-listings.json` — dedup state, keyed by Craigslist listing ID. Each entry: `{ first_seen, title, price, category }`. Append-only in practice; the routine reads it to skip listings already evaluated.
- `scan-log.json` — append-only array of every listing the routine evaluated, with the per-criteria pass/fail breakdown. Powers the dashboard. One entry per *unique* listing (same dedup as `seen-listings.json`). See "Scan log schema" below.
- `index.html` — static dashboard (vanilla JS) that fetches `scan-log.json` and renders a sortable, filterable grid. Designed to be served by GitHub Pages from the repo root.
- `digests/digest-YYYY-MM-DD-HH.md` — one file per run **only when qualifying deals are found** (timestamp components are America/New_York local time). State-only runs produce no digest.
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
  "rough_market_estimate": 2400,
  "tier1_passed": true,
  "tier1_skip_reason": null,
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
  "near_miss_too_old": false,
  "notes": "optional model commentary, especially for near-misses"
}
```

`savings_pct` is `listing_price / market_price` (i.e. % of market — lower is better; `under_40pct_market` is true when this is ≤ 0.40). Unknown numeric fields should be `null`, not omitted, so the dashboard renders blanks instead of breaking sorts.

`scanned_at` and `posted_at` are stored as UTC ISO 8601 (the trailing `Z`). The dashboard converts to America/New_York local time for display; commit messages and digest filenames use local time directly. Keep the stored values UTC so sorts and inter-tool comparisons stay unambiguous.

### Two-tier evaluation fields

- `rough_market_estimate` — the routine's in-context (no-WebSearch) market guess from the title alone. Populated for every non-vague entry that reached Tier 1; null for `vague_title` skips and entries dropped before then.
- `tier1_passed` — whether the listing survived the cheap pre-filter and got promoted to real comp lookup.
- `tier1_skip_reason` — exhaustive enum: `"vague_title"` or `"rough_market_too_low"` when `tier1_passed` is false; null otherwise. Older entries (pre-2026-05-07) may also contain `"savings_below_floor"` (the Tier 1 absolute-savings pre-filter, removed because in-context market guesses were too conservative and dropped legitimate deals — the $200 rule still applies at final qualification using real Tier 2 comps) or the bug value `"not_assessed"` (agent invented this; fixed by banning it explicitly in the prompt). Neither value should appear on new entries. The dashboard surfaces the skip reason as a hover tooltip on the ≤40% column when it's null.
- `near_miss_too_old` — true if the listing passes everything *except* the 24h recency rule (i.e., posted within 30d, ≤40% market, ≥$200 savings, scam screen passed). NOT emailed or digested — only surfaced on the dashboard so we can see whether the 24h rule is leaving real deals on the table.

Only entries with `tier1_passed: true` will have `market_price`, `comp_source`, `savings`, `savings_pct`, `under_40pct_market`, or `savings_over_200` populated — the comp lookup runs only at Tier 2.

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
Run YYYY-MM-DD HH:MM <TZ>: N deals, X scanned[, took Mm Ss] (state only)
```

`<TZ>` is the America/New_York abbreviation at commit time — `EDT` in summer, `EST` in winter (the routine emits whichever applies via `TZ=America/New_York date '+%Z'`). `(state only)` is omitted when a digest was produced.

Examples:
- `Run 2026-05-06 14:05 EDT: 0 deals, 162 scanned, took 14m 32s (state only)`
- `Run 2026-01-15 09:55 EST: 1 deal, 155 scanned`

Older commits (through 2026-05-06) used `UTC` — that's expected. The cutover happened with a routine prompt edit on that date.

For non-routine commits (config, doc, subscriber edits), use a normal short imperative subject — see `bf849ca`, `8f31e7c`.

## Editing guidance

- **Subscriber changes** are the most common manual edit. Validate region/category values against the enums above; the routine has no schema validation.
- **Do not hand-edit `seen-listings.json` or `scan-log.json`** unless removing a known-bad entry. Both are routine-managed and grow over time.
- **Do not delete old digests.** They are the historical record of what the routine surfaced.
- **No build, no tests, no lint.** There is nothing to run locally. The only "execution" is the scheduled routine.
- **Commit and push after every change.** When you modify any file in this repo (subscribers, README, CLAUDE.md, manual digest edits, etc.), commit the change with an appropriate message and `git push` before ending the turn. Use a short imperative subject for human-driven edits (e.g. `update subscribers`, `fix daytona region typo`); reserve the `Run YYYY-MM-DD HH:MM <TZ>: ...` format for routine-style runs only. Do not batch unrelated edits into one commit.
