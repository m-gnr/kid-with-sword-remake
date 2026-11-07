# Benchmarks

## Protocol

1. Run on AC power and keep Game Mode, the refresh mode, Metal HUD state and background workload consistent. Record the executable hash, source commit or provenance, game build and chain version.
2. Load a fresh copy of the same Kokiri Forest save for each launch. Set Normal, Docked, fullscreen and a 60 FPS target. Use a stationary camera at the same landmark and save reference screenshots. Record the 3D, game-output and drawable dimensions. Use a fixed separate route for visual correctness and scene-entry measurements.
3. Warm up for 30 seconds after reaching the scene and setting the camera. Record the following 60 seconds. Repeat three times for each setting, alternating baseline and candidate where possible. If the log still shows loading or pipeline compilation at the measurement boundary, record the failed run and repeat.
4. Collect `DM_NVN_TSV`, host log timings and system swap I/O deltas. Compute presented FPS from presentation feedback over wall time. Use the reciprocal of the mean of the slowest 1% of presented intervals for the 1% low. Report frame-time p50, p95 and p99, CPU submission, footprint, Metal allocation and displayed/discarded counts. Record units and mark unavailable counters.
5. Profile a separate series with `DM_NVN_GPUPROF`, use `dm/tools/gpuprof.py` to rank pass shapes and inspect GPU spans and gaps. Match hot passes to shader keys using pass dumps or an Xcode Metal capture. Use `DM_NVN_GPUPROF_CS=1` only for a separate compute diagnostic, since it changes execution ordering. Use a separate Metal System Trace for GPU idle and CPU/GPU waits. Keep these traces out of the ordinary A/B series.

See [GPU profiling](../agents/05-debugging.md#gpu-time-on-the-device) and the [Mac benchmark harness](../development.md#a-benchmark-on-the-mac). Verify that a harness launch exercises the same fullscreen app settings before comparing its output with the working app.

## Results

| Date | Git commit | Hardware | Configuration / Game Mode | Scene and camera | Duration | Average FPS / 1% low | GPU / frame-time percentiles | Memory | Visual validation |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Phase One, exact date unavailable | `e9c2f7e1` source baseline, measured binary unverified | Base M3, 16 GiB, macOS 26.6.2 | Docked, Normal, fullscreen, 60 FPS target / Off | Kokiri Forest, camera unrecorded | Unavailable | 21.62 observed, presentation basis unverified / unavailable | 36.29 ms reported GPU, span unavailable. Historical p50/p95 in [Baseline](01-baseline.md). p99 unavailable. | 12.37 GB app, 9.66 GB Metal; swap deltas unavailable | Playable scene reached; no matched image comparison. |

These observations were uncontrolled. Keep raw logs, TSVs, traces and copied saves in the title's external work directory. Record only summaries and the local run ID in Git.
