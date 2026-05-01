# Boston CL deal scanner

Auto-managed by a Claude Code scheduled routine.

- Scans Boston Craigslist (electronics, photo/video, furniture) every 4 hours
- Surfaces only listings at <=40% of estimated market value with >=$200 absolute savings
- State: `seen-listings.json` (auto-managed)
- Output: `digests/digest-YYYY-MM-DD-HH.md` when deals are found

Routine: https://claude.ai/code/routines/trig_016RMx4nmXZzhaJsPcfysMME
