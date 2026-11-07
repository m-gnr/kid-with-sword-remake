# Ocarina of Time on Apple M3

This branch tracks performance work for the Switch 2 release of The Legend of Zelda: Ocarina of Time (title ID `0400486024DEC000`) on base M3 (16 GB).

- **P0 (Primary): Normal + Handheld (1280 × 720), targeting stable 60 presented FPS** (16.67 ms per frame).
- **P1 (Secondary): Normal + Docked, targeting stable 60 presented FPS if feasible.**

Preserve visual quality and normal gameplay speed in both modes. Do not use Potato settings.

The test machine is a base Apple M3 MacBook Pro (Mac15,3), with 16 GiB of unified memory and macOS 26.6.2 (25G83). The captures used Xcode 27.2 (27B52). The optimization branch started at `e9c2f7e1f19a8c74e90283e892b8973173eabd61`. That is the source baseline. The exact identity of a measured app must be established separately.

The existing base M3 measurements used Normal + Docked. Phase One observed 21.62 FPS, a 46.25 ms frame interval, 36.29 ms of reported GPU time, 12.37 GB of application memory and 9.66 GB of Metal memory in Kokiri Forest. Game Mode was off. These are historical, uncontrolled measurements. The first base-M3 pass profile measured a 34.36–34.78 ms encoder span in the stationary view. Three ordinary runs per setting measured 20.52 presented FPS with dispatch overlap off and 20.87 with it on. Tagged GPU span p50 fell from 35.26 to 33.97 ms.

## Phase Two status

- [x] Verify the fork and publish the baseline optimization branch.
- [x] Preserve the Phase One measurements and the five investigation areas.
- [x] Verify an isolated launch with a copied Kokiri Forest save and fixed camera.
- [x] Collect and rank a separate base-M3 per-pass profile.
- [x] Attribute the NDSR compute workload and save an Xcode Metal frame capture and Instruments trace.
- [x] Complete three accepted ordinary baselines and three accepted dispatch-overlap runs.
- [x] Rank candidates and select dispatch overlap for wider route validation.
- [ ] Attribute the hottest render pass by draw in Xcode.
- [x] Identify the native-store backport and verify the local shader.
- [ ] Obtain approval before substantial renderer changes or a full game rebuild.

## Phase Three status

The [Phase Three technical report](08-phase-three-technical-report.md) records the base M4 investigation and the order of follow-up tests. A base M4 / 24 GB report reached 39.5 FPS outdoors and 45.2 FPS indoors in Handheld mode. No matched 20 → 40–45 FPS comparison or new base M3 performance result was verified.

## Technical record

- [Baseline](01-baseline.md)
- [Investigation](02-investigation.md)
- [Optimization plan](03-optimization-plan.md)
- [Benchmark protocol and results](04-benchmarks.md)
- [Engineering changelog](05-changelog.md)
- [Known issues](06-known-issues.md)
- [Base M4 community evidence and fork comparison](07-base-m4-investigation.md)
- [Phase Three technical report](08-phase-three-technical-report.md)

Keep game inputs, saves, generated shaders, captures and raw logs outside Git. Put new game-specific findings in the [game note](../../dm/games/0400486024DEC000/README.md) and link them from this project record.
