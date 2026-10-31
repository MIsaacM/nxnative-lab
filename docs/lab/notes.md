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

### Mac shaders and pipeline archive speed (hypotheses, 2026-10-31, not measured)

On a large NVN game, the two slowest first-build steps come after the split:
- step 6, Mac shaders (`dm/gfx/tools/dmsh_build.py`): tens of minutes;
- step 8, pipeline archive (`dm/gfx/tools/pso_offline.py`, `--platform mac-app` only): about half as long again.

**Step 6, Mac shaders: little room.**
- Already parallel: the translator runs in `J` batches (`dmsh_build.py` `xl`, one `dmsh_xlate` process per chunk). Each shader's MSL compile runs in a pool of `J` threads (`os.cpu_count()` via the build script).
- Already cached: the content-addressed cache (`DMSH_CAS`) and the libraries of older chain versions (prefill) make a rebuild of the same game near-instant. A first build of a game has 0 hits by definition.
- Per-shader process overhead is not the cost: `xcrun -f metal` takes about 6 ms per call on the M1 Pro. The time is Apple's Metal front end, about 1 s of CPU per shader at `-O1`.
- Possible small win: one `metal` call straight to a `.metallib` instead of `metal -c` + `metallib` (`mac_compile`), saving one process per shader. It changes `MAC_RECORD`, which invalidates the compile cache once. Probably a few percent; only worth it with numbers.
- Big win only by design change: build only the shaders a game actually uses, lazily on the Mac. Not possible on iOS (no JIT), so not an upstream direction.

**Step 8, pipeline archive: possibly a large win.**
- `pso_offline.py` runs **one** `metal-tt` process for all bases (`cmd = [... 'metal-tt', '-arch', arch, ...]`).
- If `metal-tt` is single-threaded, splitting the bases script into `J` parts and running them in parallel could cut this step several-fold.
  - Cost: the host would load several `.mtl4a` archives instead of one (`DM_PSO_BUNDLE`), or the parts would need merging. `metal-tt -help` lists no merge option.
  - First check: Activity Monitor during step 8. If `metal-tt` sits near 100% CPU, it uses one core and this is worth doing. If it sits near 800-1000%, it's already parallel: drop the idea.
- `metal-tt -enable-disk-cache` (caches intermediate products) isn't used anywhere in the repo. It may speed up a rebuild after small changes; untested.
- Existing escape hatch: `--gpu none` skips the step, and the app builds its pipeline bases on first start (stutter instead of build time).

**Upstream since (2026-10-31 sync):** `0b3fa7e7` parallelizes the rewriter scans (step 3); `89074294` compiles the shared host shaders during builds; `8b06fab7` removes older chain versions' libraries after a build (disk, not time). None touches steps 6 or 8.

Next: on an NVN game dumped from the user's own console, (1) watch `metal-tt`'s CPU use in step 8; (2) if single-threaded, prototype the split on `exp/pso-parallel`, then time the full build before and after.
