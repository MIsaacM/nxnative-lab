# Lab notes

## 2026-10-28 — Hades (0100535012974000) v0 1.0.34820, derived profile, M1 Pro 16 GB, `--platform mac`

Baseline: `bench/results/2026-10-28-hades-v0-baseline.log` (~15 min windowed, played by hand).
54,901 presents over 54,947 display ticks (60 fps), 47 empty ticks, 40 merged frames, present GPU p50 0.210 ms,
0 audio underruns, 1,344 pipelines, 960 MB Vulkan buffer memory. Image, audio, pacing and a DualShock 4 all fine.

### Upstream candidates (not yet reported or fixed; user chose not to report for now)

Upstream state checked 2026-10-28: no issue or PR covers items 1–3 below. Related:
- [#460](https://github.com/seedds/nxnative/pull/460) (open): Mac app `dataFolder`, `DM_LOG_DIR`, and `DM_SAVE`
  relative to the folder holding the `.app` — the portable save we wanted. Its writability check creates a real file,
  which passes on exFAT, so a save beside an app on exFAT would still mount read-only (item 1).
- [#475](https://github.com/seedds/nxnative/pull/475) (open): `--work-dir` for the build folder (could keep builds on
  APFS with games on exFAT). [#555](https://github.com/seedds/nxnative/pull/555) (open): AppleDouble files in c1 guard.
  [#476](https://github.com/seedds/nxnative/pull/476) (merged): earlier exFAT fixes in the NSP script.

1. **Save on exFAT is silently read-only.** `hle_fs.c` clones `$DM_SAVE` with `clonefile` (APFS only). With the NSP
   folder on exFAT the clone fails (`errno=45`, ENOTSUP), the save mounts read-only and every write is dropped:
   `[RED] save transaction: cloning the work directory failed (…/run/save.work, errno=45)`.
   `--platform mac` puts `run/save` under `<id>_build/` on the NSP drive, and docs say exFAT NSP folders are supported.
   Fix ideas: fall back to a plain recursive copy when `clonefile` returns ENOTSUP, or have the NSP script put
   `run/save` on an APFS volume / warn at build time. The Mac app is not affected (save in ~/Library).
2. **`DM_NVN_TSV` is NVN-only, so `scripts/bench.py` gets no frames for Vulkan games.** The file is opened only in
   `dm/gfx/nvn/hle_nvn.c:1348`; Hades never creates it. `bench.py` reads all its numbers from it. Check upstream
   issues first; then either write the rows from the shared present path or make `bench.py` say so clearly.

3. **`--platform mac-app` fails to sign on exFAT.** The default output is the NSP folder; on exFAT macOS writes
   AppleDouble files into the bundle and `codesign` stops on them:
   `code object is not signed at all — In subcomponent: …/Hades.app.partial/Contents/MacOS/._dmhost`.
   Workaround: `-o /Applications/Hades.app` (APFS). Even then the packer copies the build folder's `._*` files into
   the bundle (`Contents/Resources/dm/._main.patched.bin`, `msl/vk/._vk_shaders.metallib`, … listed in `Hades.app.key`);
   signing passes because they sit under Resources, but they are junk in the app. Fix ideas: build `<app>.partial` on an APFS temp dir and move it,
   or `dot_clean`/strip `._*` before signing, or refuse a non-APFS output up front with a clear message.

### Streaming test (2026-10-29)

- Sunshine on macOS drops gamepad input (official docs: "Gamepads do not work"); macOS has no virtual gamepad API.
- Steam Remote Play to an iPhone (Steam Link) works with a DualShock 4 once the shortcut's Steam Input layout maps
  the pad to nxnative's keyboard keys (the default "Gamepad" layout emulates a pad, which never reaches the game).
  Needs Accessibility/Input Monitoring for Steam, `DM_LAUNCHER=0` in the player's `env.txt` (Steam Link cannot show the
  launcher's settings window) and the game window key on the Mac (keys go to the frontmost app).
- The Mac keyboard had no right stick (`push()` passed 0, 0). Branch `exp/kbd-right-stick`: keypad 8/4/2/6 drive it,
  released with the other keys on focus loss and when the text box opens. Small, self-contained upstream candidate.

### Mac app on an external drive (exFAT)

- Building/signing on APFS and then copying the app to exFAT (`ditto`) works: `codesign --verify --deep --strict`
  passes on the copy (the `._*` files carry `com.apple.provenance` and are read through the xattr layer). Item 3 only
  bites when the bundle is assembled and signed on exFAT.
- First open of the copy took ~25 s before the settings window; reopening is instant. Likely cause (not proven): the
  first-launch signature check hashes the whole sealed bundle, including the 4 GB `Contents/Resources/romfs.img`, read
  cold over USB. Will scale with RomFS size (TotK ~16 GB). Idea to study: keep the RomFS outside the code-signing seal,
  since the launcher already checks its recorded files (`DMModuleFiles`) — security trade-off to weigh before proposing.

### Open questions

- `submission ahead of target display p50 49.8 ms`: ~3 frames between submit and display. Find what it measures
  before treating it as input latency.
- `[vk] dynamic state 3 not implemented` (VK_DYNAMIC_STATE_DEPTH_BIAS): the game imports no `vkCmdSetDepthBias`,
  so harmless here.
- Profile was brought up on v589824 (1.0.38233); this run is v0 from a derived profile. Re-measure on the update.

### Right stick (2026-10-30)

- Steam Link on macOS has no virtual gamepad, so the right stick is mapped to keypad 8/4/2/6 (branch `exp/kbd-right-stick`).
- Hades doesn't use right-stick movement: in-game it only uses the stick press (to reload one weapon), which already worked. So Hades can't confirm the keypad mapping; test it with a game that uses the right stick for the camera or aiming.

### Shader split speed (hypothesis, 2026-10-30, not measured)

The shader split (build step 5/8, `dm/gfx/tools/shader_split.py`) is the slowest first-build step for NVN games (Breath of the Wild: ~5 of ~10 min, per the build script's header). Vulkan games such as Hades skip it.

Already in place:
- Parallel scan: a process pool, with `J` set to `os.cpu_count()` by `build_ipa_from_nsp.py` (10 on the M1 Pro). More workers won't help.
- Reuse: a repeat build with unchanged inputs skips the step (`split_shaders` fingerprint).

Where the time goes: it walks, decompresses and scans **every** RomFS file, although only a fraction holds shaders.

Idea: for a game with its own (non-provisional) profile, scan only the files named in column 5 (`source`) of the profile's `shader_map.tsv`, plus `main`, and skip textures, audio, video and models.
- Self-checking: the split already requires its tables to be byte-identical to the profile's (`data_rows` comparison), so a missed file stops the build instead of silently dropping shaders.
- Column 5 records only where a key *first* appears; duplicates elsewhere are dropped anyway, so the key set should be the same.
- Edge cases:
  - Unity serialized files borrow type trees from bundles that may hold no shaders, so their bundles must still be scanned.
  - Tears of the Kingdom's zstd dictionaries (`zs_dicts`) come from separate files.
  - Archive-range jobs (`kpka_jobs`, `smash_arc_jobs`, …) name archive entries, not just files.
  - The "file(s) could not be read" count would stop covering skipped files.
- Fallback: provisional profiles and any mismatch run the full scan.

Next: implement on `exp/shader-split-sources` and measure first-build time with `scripts/bench.py` or plain wall-clock, before and after, on an NVN game dumped from the user's own console. Hades (Vulkan) can't measure it. Candidate for upstream if the gain is real.
