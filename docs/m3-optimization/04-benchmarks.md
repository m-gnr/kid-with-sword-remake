# Benchmarks

## Protocol

1. Run on AC power and keep Game Mode, the refresh mode, Metal HUD state and background workload consistent. Record the executable hash, source commit or provenance, game build and chain version.
2. Load a fresh copy of the same Kokiri Forest save for each launch. Start with P0 (Normal + Handheld, 1280 × 720) and a 60 FPS target. If feasible, evaluate P1 (Normal + Docked) in a separate series with the same target. Keep presentation settings fixed within each series and do not use Potato settings. Use a stationary camera at the same landmark and save reference screenshots. Record the 3D, game-output and drawable dimensions. Use a fixed separate route to check visual quality, normal gameplay speed and scene-entry performance.
3. Warm up for 30 seconds after reaching the scene and setting the camera. Record the following 60 seconds. Repeat three times for each setting, alternating baseline and candidate where possible. If the log still shows loading or pipeline compilation at the measurement boundary, record the failed run and repeat.
4. Collect `DM_NVN_TSV`, host log timings and system swap I/O deltas. Compute presented FPS from presentation feedback over wall time. Use the reciprocal of the mean of the slowest 1% of presented intervals for the 1% low. Report frame-time p50, p95 and p99, CPU submission, footprint, Metal allocation and displayed/discarded counts. Record units and mark unavailable counters.
5. Profile a separate series with `DM_NVN_GPUPROF`, use `dm/tools/gpuprof.py` to rank pass shapes and inspect GPU spans and gaps. Match hot passes to shader keys using pass dumps or an Xcode Metal capture. Use `DM_NVN_GPUPROF_CS=1` only for a separate compute diagnostic, since it changes execution ordering. Use a separate Metal System Trace for GPU idle and CPU/GPU waits. Keep these traces out of the ordinary A/B series.

See [GPU profiling](../agents/05-debugging.md#gpu-time-on-the-device) and the [Mac benchmark harness](../development.md#a-benchmark-on-the-mac). Verify that a harness launch exercises the same console mode, resolution and presentation settings before comparing its output with the working app.

## Isolated Mac launch

The launch example below records the settings used for the historical Normal + Docked fullscreen runs.

Use an APFS scratch directory under the game's `0400486024DEC000-work/`. Close any other instance of the title. Clone the working app into a new app path using `cp -cRp` and snapshot the committed `save/`, optional `save_device/`, `pso_user.tsv` and `pso_archive/` into a private fixture. Copy the fixture into a new run directory on every launch. Avoid the live `save.work/`. Keep cloud saves disabled and leave the original app, saves and archives in place.

The clone's launcher stalled before it wrote a log. The same clone's embedded host started in fullscreen with its data paths set explicitly. Create fresh `home`, `temp`, and `tmp` directories and an empty `ctl` in the run directory. Set `app` to the clone's absolute path, `run` to the run directory, and `overlap` to 0 for baseline or 1 for the overlap test. Start in a signed-in GUI session.

```sh
res="$app/Contents/Resources"
cd "$run"
env -i PATH=/usr/bin:/bin HOME="$run/home" TMPDIR="$run/tmp" LANG=en_US.UTF-8 \
  DM_ROOT="$run" DM_SAVE="$run/save" DM_TEMP="$run/temp" DM_SAVE_SYNC=0 \
  DM_PSO_USER="$run/pso_user.tsv" DM_PSO_ARCHIVE="$run/pso_archive" \
  DM_ROMFS="$res/romfs.img" DM_MSL_DIR="$res/msl" DM_FONT_DIR="$res/dm/font" \
  DM_NVN_SHMAP="$res/dm/shader_map.tsv" DM_PSO_BUNDLE="$res/pso" \
  DM_WINDOW=1 DM_FULLSCREEN=1 DM_AUDIO=1 DM_OPMODE=1 DM_RES_PRESET=0 DM_FPS_TARGET=60 \
  DM_UPSCALER=metalfx DM_PRESENT_UPSCALE=metalfx DM_NVN_DISPATCH_OVERLAP="$overlap" \
  DM_METAL_HUD=0 DM_NVN_TSV="$run/frames.tsv" DM_NVN_TSV_CALLS=0 DM_CTL="$run/ctl" \
  "$app/Contents/MacOS/dmhost" "$res/dm" >"$run/log.txt" 2>&1
```

At the title, press A to advance to the Load list, A to select the top Kokiri Forest save, and A to confirm Load. In the local script, three frame-indexed presses are followed by one conditional A only if the menu remains active. Visual verification of the scene is required after each run. Leave the character and camera at the spawn point, facing the stepping stones with the sunlit forest and wooden structure ahead. Use `DM_CTL` for screenshots outside the timing window. Keep the game foreground during capture and reject runs with covered or paused windows. The clone's `LSSupportsGameMode` is false. Record the active display mode and the workload at the timing boundary.

Read `@pres_at_ms` for actual present times. Count distinct timestamps inside the fixed 60 s window. Take intervals between successive observed presentations, including gaps left by discarded frames. Count discarded frames separately using `@presented_ms=-1` and report missing feedback separately. Use `@frame_ms` for host percentiles, `@sub_cpu_ms` for CPU submission, and `@gpu_end_ms - @gpu_beg_ms` where available for the host's command-buffer span. Report `@gpu_ms` separately as summed duration. Leave a few seconds after the measurement window for delayed GPU and presentation feedback. Snapshot `vm_stat` and `sysctl vm.swapusage` on both boundaries. Use the host's memory windows for physical footprint and GPU allocation.

For a separate pass run, add `DM_NVN_GPUPROF=every:240` before the executable and retain only frames in the loaded scene after warmup. Append `passdump 1` to the control file outside the timing interval. Summarize with `python3 dm/tools/gpuprof.py "$run/log.txt" --frames <frame IDs>`, then `--timeline <frame>` and the nearby pass dump. A separate compute run adds `DM_NVN_GPUPROF_CS=1`. For a timeline trace after warmup, use `xcrun xctrace record --template 'Metal System Trace' --attach <host PID> --time-limit 10s --output "$run/metal.trace"`. For per-draw inspection, launch separately with `MTL_CAPTURE_ENABLED=1`, a private `DM_GPU_CAPTURE_DIR`, and `DM_GPU_CAPTURE_KEEP=1`, then set a future frame via `DM_CTL`. Use Xcode to inspect the capture.

## Results

| Date | Git commit | Hardware | Configuration / Game Mode | Scene and camera | Duration | Average FPS / 1% low | GPU / frame-time percentiles | Memory | Visual validation |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Phase One, exact date unavailable | `e9c2f7e1` source baseline, measured binary unverified | Base M3, 16 GiB, macOS 26.6.2 | Docked, Normal, fullscreen, 60 FPS target / Off | Kokiri Forest, camera unrecorded | Unavailable | 21.62 observed, presentation basis unverified / unavailable | 36.29 ms reported GPU, span unavailable. Historical p50/p95/p99 by window in [Baseline](01-baseline.md). | 12.37 GB app, 9.66 GB Metal. Swap deltas unavailable. | Playable scene reached. Matched image comparison unavailable. |

These observations were uncontrolled. Keep raw logs, TSVs, traces and copied saves in the title's external work directory. Record only summaries and the local run ID in Git.

Phase Two calibration used game version v131072, chain `dc815e0f38ca`, input stamp `60600e433778`, and app host SHA-256 `1b5940ad1a5ea09f369e172c56b695b583e6804ba696e5d3460bbe0bf11cbdfa`. This records the artifact used for local runs. The checkout's baseline commit identifies the source context and does not prove a byte-reproducible build. Run files are local to `0400486024DEC000-work/m3-phase-two.se3HhL/`.

| Run | Date / commit context | Duration | Actual presented FPS / 1% low | Host frame p50/p95/p99 | Tagged GPU span p50/p95/p99 | Memory and validation |
| --- | --- | ---: | ---: | --- | --- | --- |
| `direct-calibration` | 2026-11-07 / `e9c2f7e1` source baseline | 60.209 s | 18.70 / 2.01 | 49.76/51.77/56.92 ms | 36.86/39.70/50.80 ms | 10,545 MiB footprint, 7,102 MiB Metal sampled peaks. Verified forest view. Focus loss inside the measurement window invalidated the timing pilot. |
| `profile-2` | 2026-11-07 / `e9c2f7e1` source baseline | 45.019 s | 20.32 / 12.90 | 49.90/51.63/52.77 ms | 35.31/37.94/41.89 ms | 10,507 MiB footprint, 7,182 MiB Metal sampled peaks. 915 distinct presents, 22 discarded, swap-in/out deltas 9,107/12,196 system pages. Stationary forest view. Diagnostic instrumentation excludes this run from ordinary A/B. |

Both runs used the base M3, 16 GiB, macOS 26.6.2 and the verified host artifact above. Settings were Docked, Normal, fullscreen, 60 FPS target, Game Mode off and dispatch overlap off. `profile-1` stopped in the title after the local reader encountered a partial TSV line. The retry ignored incomplete lines and used a manual confirm in the same copied save.

### GPU diagnostic artifacts

- `compute-1/log.txt` contains the separately serialized compute samples, a same-frame all-bindings pass dump and shader attribution in [Investigation](02-investigation.md#serialized-compute-and-same-frame-attribution).
- `capture-1/captures/dm_20261107-191459_f3330.gputrace` is the Xcode Metal frame capture from the copied save. The host logged the start and stop, then exited cleanly after the trace write window. The package occupies 7.7 GiB on the external APFS volume. The capture path compiled pipelines through `MTLDevice` and took about 165 s to prewarm. Keep its timings separate. Per-draw analysis in Xcode is pending.
- The first `Metal System Trace` attempt in `compute-1` reached the 10 s time limit but timed out during finalization. `xctrace export --toc` rejected its partial bundle with `Document Missing Template Error`. The retry recorded an 8 s Metal System Trace after the last ordinary timing window and let finalization finish after the game exited. `overlap-4/metal.trace` is 218 MiB. Both the table of contents and the GPU intervals exported successfully.

## Ordinary dispatch-overlap A/B

2026-11-07, source baseline `e9c2f7e1`, cloned app host hash above. Base M3, 16 GiB, macOS 26.6.2, AC power, Game Mode off, Docked, Normal, fullscreen and a 60 FPS target. The same copied save loaded at the same stationary stepping-stone view. Every accepted run had 30 s of warmup and about 60 s of timing-only TSV output. The post-window pass dumps confirm 854 × 480 for the main 3D passes. The game output is 2560 × 1440 and the MetalFX presentation target is 3600 × 2025. The game-side MetalFX request fell back to native NDSR in both configurations.

`base-*` sets `DM_NVN_DISPATCH_OVERLAP=0`. `overlap-*` sets it to 1. The first six runs alternated settings. `base-1` and `overlap-3` are retained below and excluded because start-boundary CPU samples showed `replayd` at 86.8% and the Codex renderer at 140.2%, respectively. The later runs replaced them. Workload samples are snapshots, and system swap and background work can vary within a window. The Mac host reported its thermal state as unavailable. The accepted windows had no focus events, missing presentation feedback, synchronous pipeline creations or reported re-uploads.

| Run | Window (s) | Presented FPS / 1% low | Presented p50/p95/p99 (ms) | Host p50/p95/p99 (ms) | Tagged GPU span p50/p95/p99 (ms) | GPU sum / submit CPU p50 (ms) | Shown / discarded |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| `base-2` | 60.026 | 20.52 / 13.33 | 50.00/50.00/75.00 | 49.85/51.55/52.99 | 35.23/37.99/40.85 | 35.06 / 9.91 | 1232 / 22 |
| `base-3` | 60.209 | 20.56 / 13.33 | 50.00/50.00/75.00 | 49.86/51.73/53.85 | 35.26/37.91/40.69 | 35.05 / 9.88 | 1238 / 20 |
| `base-4` | 60.130 | 20.42 / 12.00 | 50.00/50.00/75.00 | 49.80/51.85/53.72 | 35.41/38.47/39.99 | 35.03 / 8.54 | 1228 / 35 |
| `overlap-1` | 60.045 | 20.75 / 14.31 | 50.00/50.00/66.67 | 49.86/51.63/52.67 | 34.07/36.21/37.45 | 33.66 / 8.78 | 1246 / 12 |
| `overlap-2` | 60.331 | 21.07 / 14.18 | 50.00/54.17/66.67 | 49.74/51.55/52.76 | 33.97/36.79/39.05 | 33.65 / 9.21 | 1271 / 10 |
| `overlap-4` | 60.221 | 20.87 / 14.86 | 50.00/50.00/66.67 | 49.78/51.80/53.38 | 33.84/36.67/41.00 | 33.48 / 9.67 | 1257 / 7 |
| `base-1` (excluded) | 60.170 | 20.46 / 12.00 | 50.00/50.00/75.00 | 49.80/51.70/53.88 | 35.53/38.05/39.85 | 35.10 / 9.13 | 1231 / 30 |
| `overlap-3` (excluded) | 60.007 | 20.98 / 13.93 | 50.00/50.00/66.67 | 49.81/51.67/52.58 | 33.87/36.16/37.80 | 33.35 / 9.44 | 1259 / 7 |

| Run | Peak sampled footprint / Metal (MiB) | Swap in/out (16 KiB pages) | dHash bits / RGB16 distance from base-2 |
| --- | ---: | ---: | ---: |
| `base-2` | 10,584 / 7,166 | 17,655 / 31,424 | 0 / 0.00 |
| `base-3` | 10,542 / 7,148 | 9,051 / 9,920 | 4 / 4.61 |
| `base-4` | 10,530 / 7,102 | 31,149 / 46,424 | 7 / 6.84 |
| `overlap-1` | 10,556 / 7,149 | 30,644 / 21,632 | 3 / 6.76 |
| `overlap-2` | 10,531 / 7,131 | 48,161 / 66,140 | 3 / 6.65 |
| `overlap-4` | 10,713 / 7,188 | 28,197 / 24,160 | 3 / 5.00 |
| `base-1` (excluded) | 10,486 / 7,120 | 22,433 / 15,036 | 5 / 4.79 |
| `overlap-3` (excluded) | 10,515 / 7,101 | 19,015 / 20,992 | 5 / 5.08 |

The run medians were 20.52 → 20.87 presented FPS (+0.35, 1.7%), 13.33 → 14.31 FPS for the 1% low and 35.26 → 33.97 ms for tagged GPU span p50 (−1.29 ms, 3.7%). GPU span p95 went from 37.99 to 36.67 ms, and host frame p50 from 49.85 to 49.78 ms. Median discarded counts were 22 → 10. Median on-screen interval stayed at 50.00 ms, consistent with the game's three-refresh holds. Tagged GPU span comes from the first and last timestamps of host-tagged command buffers. It differs from both summed GPU duration and the display interval.

The screenshots match in camera, terrain, depth appearance and HUD on visual spot check. The accepted overlap screenshots were 3 dHash bits and at most 6.76 RGB16 units from `base-2`, within the variation of the baseline screenshots. This checks the stationary view. Moving foliage, transitions and simulation speed along a gameplay route still need validation before adopting the toggle as a default.
