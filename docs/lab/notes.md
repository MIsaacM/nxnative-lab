# Lab notes

## 2026-10-28 — Hades (0100535012974000) v0 1.0.34820, derived profile, M1 Pro 16 GB, `--platform mac`

Baseline: `bench/results/2026-10-28-hades-v0-baseline.log` (~15 min windowed, played by hand).
54,901 presents over 54,947 display ticks (60 fps), 47 empty ticks, 40 merged frames, present GPU p50 0.210 ms,
0 audio underruns, 1,344 pipelines, 960 MB Vulkan buffer memory. Image, audio, pacing and a DualShock 4 all fine.

### Upstream candidates (not yet reported or fixed)

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
