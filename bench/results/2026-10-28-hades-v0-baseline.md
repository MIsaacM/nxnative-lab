# Hades v0 baseline — 2026-10-28

- Game: Hades `0100535012974000`, v0 (1.0.34820), profile derived from v589824 (1.0.38233). Vulkan on Metal.
- Build: `build_ipa_from_nsp.py --platform mac`, lab branch at upstream `2f1a044`.
- Mac: M1 Pro, 16 GB, macOS 27, Xcode 27.1 (Metal 32023.921, matches `dmsh_chain.lock`).
- Run: windowed, `DM_WINDOW=1 DM_AUDIO=1`, played by hand for ~15 min (menu and first rooms). Not a scripted route:
  `scripts/bench.py` cannot measure Vulkan games yet (`DM_NVN_TSV` is NVN-only, see `docs/lab/notes.md`).
- Source: the host's periodic log lines (full log kept locally, not committed).

| Metric | Value |
|---|---|
| Display ticks / presents | 54,947 / 54,901 (60 fps) |
| Empty ticks | 47 (0.09 %) |
| Old frames dropped by merging | 40 |
| Present GPU time | p50 0.210 ms, p95 0.215 ms, max 4.587 ms |
| Submission ahead of target display | p50 49.8 ms |
| Audio underruns | 0 |
| Pipelines created | 1,344 |
| Vulkan buffer memory | 960 MB |
| `[RED]` lines | 2: save read-only on exFAT; `dynamic state 3` (depth bias, harmless) |
