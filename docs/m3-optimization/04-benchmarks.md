# Benchmarks

## Protocol

1. Run on AC power and keep Game Mode, the refresh mode, Metal HUD state and background workload consistent. Record the executable hash, source commit or provenance, game build and chain version.
2. Load a fresh copy of the same Kokiri Forest save for each launch. Set Normal, Docked, fullscreen and a 60 FPS target. Use a stationary camera at the same landmark and save reference screenshots. Record the 3D, game-output and drawable dimensions. Use a fixed separate route for visual correctness and scene-entry measurements.
3. Warm up for 30 seconds after reaching the scene and setting the camera. Record the following 60 seconds. Repeat three times for each setting, alternating baseline and candidate where possible. If the log still shows loading or pipeline compilation at the measurement boundary, record the failed run and repeat.
4. Collect `DM_NVN_TSV`, host log timings and system swap I/O deltas. Compute presented FPS from presentation feedback over wall time. Use the reciprocal of the mean of the slowest 1% of presented intervals for the 1% low. Report frame-time p50, p95 and p99, CPU submission, footprint, Metal allocation and displayed/discarded counts. Record units and mark unavailable counters.
5. Profile a separate series with `DM_NVN_GPUPROF`, use `dm/tools/gpuprof.py` to rank pass shapes and inspect GPU spans and gaps. Match hot passes to shader keys using pass dumps or an Xcode Metal capture. Use `DM_NVN_GPUPROF_CS=1` only for a separate compute diagnostic, since it changes execution ordering. Use a separate Metal System Trace for GPU idle and CPU/GPU waits. Keep these traces out of the ordinary A/B series.

See [GPU profiling](../agents/05-debugging.md#gpu-time-on-the-device) and the [Mac benchmark harness](../development.md#a-benchmark-on-the-mac). Verify that a harness launch exercises the same fullscreen app settings before comparing its output with the working app.

## Isolated Mac launch

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
| Phase One, exact date unavailable | `e9c2f7e1` source baseline, measured binary unverified | Base M3, 16 GiB, macOS 26.6.2 | Docked, Normal, fullscreen, 60 FPS target / Off | Kokiri Forest, camera unrecorded | Unavailable | 21.62 observed, presentation basis unverified / unavailable | 36.29 ms reported GPU, span unavailable. Historical p50/p95/p99 by window in [Baseline](01-baseline.md). | 12.37 GB app, 9.66 GB Metal; swap deltas unavailable | Playable scene reached; no matched image comparison. |

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
- The first `Metal System Trace` attempt in `compute-1` reached the 10 s time limit but timed out during finalization. `xctrace export --toc` rejected its partial bundle with `Document Missing Template Error`. A retry is planned outside the last ordinary timing window.
