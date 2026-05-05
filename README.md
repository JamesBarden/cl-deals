# Craigslist deal scanner

A Claude Code scheduled routine that scans Craigslist across multiple regions every 4 hours, surfaces only listings priced far below market, and emails matching subscribers. This repository is the routine's state store and output log — the scanning logic itself lives in the routine prompt, not in any committed file.

## What it does

- **Scans** the `photo`, `electronics`, `furniture`, `instruments`, and `bikes` categories across `boston`, `daytona`, and `newhaven` Craigslist sites
- **Qualifies** a listing only when *all* hold:
  - Asking price ≤ 40% of estimated used market value
  - Absolute savings ≥ $200
  - Posted within the last 24h
  - Passes a scam screen (≥30-word description for items >$200; no deposit/Zelle/wire/shipping/overseas signals)
- **Estimates market value** with eBay sold comps, MPB, AptDeco, Chairish, Reverb (instruments), and BicycleBlueBook / pinkbike (bikes), adjusted by a condition factor (×0.50 good, ×0.65 like-new)
- **Emails** each qualifying deal to subscribers whose `regions` and `categories` in `subscribers.json` both match
- **Commits** every run — with a digest when deals are found, state-only otherwise

## What's in this repo

| Path | Purpose |
|---|---|
| `seen-listings.json` | Dedup state, keyed by Craigslist listing ID. Auto-managed — do not hand-edit. |
| `digests/digest-YYYY-MM-DD-HH.md` | One file per run that produced qualifying deals, including a methodology check per listing. |
| `subscribers.json` | Email routing. Each subscriber lists `regions` and `categories`; `["*"]` matches all values. |
| `CLAUDE.md` | Guidance for Claude Code instances working in this repo. |

## Subscribing

Edit `subscribers.json` and commit. Example:

```json
{"email": "you@example.com", "regions": ["boston"], "categories": ["photo", "electronics"]}
```

A subscriber receives a deal only when it matches **all** their criteria (region AND category).

## Routine

https://claude.ai/code/routines/trig_016RMx4nmXZzhaJsPcfysMME
