<p align="center"><img src="https://github.com/realblastex/flamingo/releases/download/progress-2026-11-25/flamingo-512.png" width="160" alt="Flamingo"></p>

# Flamingo

Flamingo is a work-in-progress fork of [victory-fork](https://github.com/fxceless/victory-fork),
which is itself a fork of [VIctory](https://github.com/Marvisak/VIctory). VIctory is a
compatibility layer that runs an Xbox Series title natively on Windows from files you supply.
Flamingo is tuned for one freshly released open-world Xbox Series title, run from a dump of your
own legally obtained copy, on an NVIDIA RTX 4070. Fork itself is mainly tuned for NVIDIA cards.

**This repository contains no game files, code or assets.** You need your own copy. Flamingo is
not affiliated with or endorsed by any game's developer or publisher, or by Microsoft.

## Status (2026-11-26)

Tested on one machine: Windows 11, NVIDIA RTX 4070 12 GB, Series S profile (`--console series-s`),
with the settings under [Build and run](#build-and-run).

| Stage | State |
|-------|-------|
| Boot, menus, start screen | work (30 fps) |
| Cutscenes | play through; faces move (their 16-bit integer data now keeps its bits); shade is lit by a constant floor of light |
| First mission (boat cutscene) | **about 10 fps** (run 222: 10.4 fps median, 96 ms a frame) |
| Characters | mostly whole; a few meshes come out of the skinning passes wrong, the same ones every frame (a body missing in the boat cutscene, a head in the opening one) |
| Distant trees and buildings | left out: the title's impostors draw as huge flat panels on this layer |
| Water | dark blocks, from the water's transmission volumetrics; leaving those two programs out removes them (the water turns grey) |
| Ray-traced light and shadows | do not run; a constant floor of indirect light stands in |
| Stability | runs end without a device loss |

What was found this round, and what to look at next: [docs/handoff.md](docs/handoff.md).

<p align="center">
<img src="https://github.com/realblastex/flamingo/releases/download/progress-2026-11-25/prologue-window.png" width="70%" alt="The first gameplay scene">
</p>

Screenshots and the logo are attachments of the
[progress release](https://github.com/realblastex/flamingo/releases/tag/progress-2026-11-25), not
files in the tree: the asset policy ([docs/asset-policy.md](docs/asset-policy.md)) keeps
everything derived from the game out of the repository. Every change and every measured run is
logged in [FLAMINGO.md](FLAMINGO.md).

## What Flamingo adds over victory-fork

Flamingo's own work:

- **Stats overlay** (RivaTuner-style). **F9** goes round a compact display (fps and frame
  time, GPU load and temperature, VRAM, CPU load, RAM, a frame-time graph, as MSI Afterburner
  shows them), the full diagnostics (also frames, draws, preparing submissions with its copies
  and textures, waiting for the GPU, copy-back in ms and MB, shaders, pipelines, dropped
  draws), and nothing. `FLAMINGO_OSD=1` starts with the compact display, `FLAMINGO_OSD=2` with
  the full one, `FLAMINGO_OSD=0` turns it off. GPU load and temperature need an NVIDIA card.
  <img src="https://github.com/realblastex/flamingo/releases/download/progress-2026-11-25/stats-overlay.png" alt="Stats overlay">
- **Discord Rich Presence**: the game's name, menu or world, frame rate and time played.
- **DualSense support**: a PlayStation 5 controller (and the DualSense Edge), over USB or
  Bluetooth, is the title's first pad while no Xbox controller is connected. It is read over HID,
  so Steam Input and DS4Windows are not needed. Buttons map by position (cross is A, circle is B,
  and so on); Options is Menu, and Create or the touch pad click is View. `FLAMINGO_DUALSENSE=0`
  turns it off. Rumble and the light bar are not driven yet.
- **Faster submissions**:
  - title-memory copies that recover from a fault instead of a kernel call per copy;
  - table constant views read the buffer's own GPU copy, with no CPU fill and no wait;
  - host objects are released on a thread of their own;
  - fewer clock reads per frame.
- **A lean build**: the launcher build has no diagnostics counters, which is about 9% faster.
  The `keys` feature keeps the scripted keys for timing runs.
- **Intro and menu flicker fix**: recently changed guarded textures are hashed at every check.
- **Pipeline syncs order shader writes**: the console's pipeline syncs were no-ops, so a later
  dispatch could read what an earlier one had not written yet; they are now a barrier on every
  shader write (`FLAMINGO_SYNC_BARRIER=0` restores the old behavior).
- **Faces move**: 16-bit typed accesses through integer views keep their bits, also for views in
  descriptor tables (VIctory 4243a96a by way of avatarmanz/VIctory-7900XTX 9a6c829d and #114).
- **`VICTORY_SKIP_PIPELINES` in every build**, so the launcher can leave out the broken impostors.
- **Diagnostics** for tracing a draw's data end to end (`FLAMINGO_DUMP_RANGE` and more; see
  [docs/handoff.md](docs/handoff.md)).

Pull requests merged from victory-fork and VIctory that have not landed upstream yet:

- victory-fork #665: copy-back fetches go on a copy queue of their own.
- victory-fork #670: no write-watch asking for known view memory.
- victory-fork #799: 64 KiB constant-view clamp and translator fixes. The loose-LDS fallback is
  opt-in here (`VICTORY_WAVE_LDS_LOOSE=1`) because it made the world grainy.
- victory-fork #812: unmapped view blocks are parked instead of asked about every frame
  (+1 fps).
- victory-fork #798: the Series S profile keeps the game's own upscaler instead of FSR 4
  (+2 fps).
- victory-fork #823: four refused compute programs run on 32-lane GPUs, and the ray-tracing
  structure builder has all its pipelines.

- victory-fork #768 (opt-in, `VICTORY_WIDE_TEXTURES=1`): a submission's texture checks run
  several at a time.
- victory-fork d4a528ba (its pull request #1007's worker; opt-in, `VICTORY_HEAP_WORKER=1`):
  textures picked by index are uploaded by a thread of their own (7.1 to 9.1 fps here).
- avatarmanz/VIctory-7900XTX #204: on 32-lane GPUs, LDS atomics in a lane branch keep their order
  between the two halves of a console wave.

Being tried (see [docs/handoff.md](docs/handoff.md)): `VICTORY_CARD_BUFFERS_MIN_KB=64
VICTORY_CARD_BUFFERS_MB=1536` keeps every title buffer from 64 KB up on the card: 11.1 fps in the
boat cutscene against 9.8, with the picture as mixed as before.

Experiments, all off by default, kept for testing (each has its result in FLAMINGO.md):

- `VICTORY_AMBIENT_STUB=1`, `VICTORY_SHADOWMASK_STUB=1`: stand-ins for the ray-traced lighting
  passes, ported from [ScarlettPort](https://github.com/ZigZagBoi1/ScarlettPort) (GPL-3.0). Here
  they did not brighten the dark areas.
- `FLAMINGO_BVH_MISS=1`: every ray-tracing node test misses. The cutscene gets brighter, but one run
  hung.
- `FLAMINGO_CARD_FILLS=1`: reads of memory that shaders wrote lately are made on the card. Slower.
- `VICTORY_AMBIENT=<level>`: after the indirect-light resolve, its output is replaced by a constant
  (from [Belobug's fork](https://github.com/Belobug/VIctory-Belobug-AMD), GPL-3.0). At 0.6 the
  opening cutscene once washed out grey; in the boat cutscene it lights the shade, and the
  settings below use it.
- `FLAMINGO_QUAD_HELPERS=1`: in pixel shaders, reads of the quad's other lanes reach helper lanes
  (also from Belobug's fork). Sharper fine detail, about 18% slower.

The full list, with the runs behind each one, is in [FLAMINGO.md](FLAMINGO.md).

## Build and run

You need Rust (stable) and Windows 11.

```
cargo build --profile diag -p victory
```

Run it with the settings this build is tuned for (about 9-10 fps on an RTX 4070):

```
set VICTORY_NO_WATCHDOG=1
set VICTORY_PIPELINE_WAIT_MS=24
set VICTORY_RELAY=1
set VICTORY_LATE_WAIT=3
set VICTORY_CARD_BUFFERS_MIN_MB=1
set VICTORY_CARD_BUFFERS_MB=1024
set VICTORY_INDIRECT_WINDOW=0
set VICTORY_SHADER_OPT=0
set VICTORY_WIDE_MERGE=1
set VICTORY_WIDE_TEXTURES=1
set VICTORY_VIEW_GUARD=0
set FLAMINGO_BVH_MISS=1
set VICTORY_HEAP_WORKER=1
set VICTORY_AMBIENT=0.6
set VICTORY_SKIP_PIPELINES=7115e46a79c78f98,59b1ba494f18c894,b8ca32a6899b87fb,659fe9cae59dbd88
target\diag\victory.exe run <path to your dump>\<the title's executable> --execute --console series-s
```

The last line leaves out the title's distant-object impostors, which draw as huge panels here:
far trees and some far buildings are then missing. For a sharper picture at about 5 fps, use
`--console series-x`. If the Series S profile shows a black screen once the game has loaded
(seen on some PCs), use `--console series-x` too.

The first launch after a build translates and compiles shaders for a few minutes. Later launches
use the cache.

For Discord Rich Presence, put your own Discord application id in
`%LOCALAPPDATA%\flamingo\discord_app_id.txt` (FLAMINGO.md has the steps).
`FLAMINGO_DISCORD=0` turns it off.

## Reporting a problem (logging build)

When something goes wrong (a black screen, a crash, missing pictures), a log tells us what
happened on your PC. `flamingo-log.cmd` makes one:

1. Install [Rust](https://rustup.rs) and [Git](https://git-scm.com) if you have not.
2. Get the newest code: `git clone https://github.com/realblastex/flamingo` (or, in a clone you
   already have, `git pull`).
3. Drag the title's executable from your own dump onto `flamingo-log.cmd` in the repository's
   folder, or run it from a command prompt:

   ```
   flamingo-log.cmd "D:\path\to\your\dump\the-title.exe"
   ```

   It builds Flamingo (several minutes the first time), runs it with the settings above and
   its log on, and remembers the path for next time (`flamingo-game.txt`).
4. Play until the problem shows (for a black screen, wait a minute in it), then close the game
   window.
5. The log is saved in the `logs` folder as `flamingo-<date>-<time>.log`, with a `.zip` of it
   next to it (Explorer opens there). Attach the `.zip` to your issue, and say what you saw and
   when.

The log starts with your Windows version, processor, memory and graphics cards with their
driver versions. It also holds file paths of your PC, which can include your Windows user
name: look it over before you share it. `set FLAMINGO_SERIES_S=1` before running uses the
Series S profile instead of Series X.

## Credits

- **[VIctory](https://github.com/Marvisak/VIctory)** by Marvisak and its contributors is the base
  of everything here: the loader, the runtime services, the Direct3D 12 back end and the shader
  recompiler. It is licensed GPL-3.0-or-later, and so is this fork.
- **[victory-fork](https://github.com/fxceless/victory-fork)** by fxceless and its contributors
  did the world rendering and speed work this tree starts from, plus the merged pull requests
  above. The fork's own README is in [docs/victory-fork-README.md](docs/victory-fork-README.md).
- **[victory-amd](https://github.com/myokex/victory-amd)** by myokex has an analysis of the dark
  shadowed areas that pointed at the missing ray-traced light.
- **[ScarlettPort](https://github.com/ZigZagBoi1/ScarlettPort)** by ZigZagBoi1: the opt-in
  stand-ins for the ray-traced lighting passes are ported from its VIctory-based runtime
  (GPL-3.0-or-later).
- **[Belobug's VIctory fork](https://github.com/Belobug/VIctory-Belobug-AMD)**: the opt-in ambient
  pass and quad helper reads are ported from it (GPL-3.0).
- The mesh primitive edge-flag fix came as pull request #1 to this repository, a port of upstream
  VIctory #1718.
- **[vx-layer](https://github.com/Geckomacabre/vx-layer)** by Geckomacabre (GPL-3.0): the copy-back
  of textures that shaders write and the title reads on the CPU (`VICTORY_TEXTURE_BACK`), which
  keeps boats afloat.
- Pull request #2 by rafaconvaitis: plain copies recover from guard-page faults.
- The audio fixes of victory-fork #959 (upstream VIctory's XMA and audio ring fixes).
- **[XCL](https://github.com/ivall/xcl)** by ivall and its contributors (GPL-2.0-only) was used as
  a reference only. No XCL code is copied here.
- Flamingo's changes were developed with [Claude Code](https://claude.com/claude-code).

## License

GPL-3.0-or-later, as VIctory. See [LICENSE](LICENSE).
