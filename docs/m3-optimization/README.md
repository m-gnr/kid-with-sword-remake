# Ocarina of Time on Apple M3

This branch tracks performance work for the Switch 2 release of The Legend of Zelda: Ocarina of Time (title ID `0400486024DEC000`). The target is stable 60 presented FPS, a 16.67 ms frame budget, with Docked mode, Normal resolution and fullscreen presentation while preserving visual quality and simulation speed.

The test machine is a base Apple M3 MacBook Pro (Mac15,3), with 16 GiB of unified memory and macOS 26.6.2 (25G83). The optimization branch started at `e9c2f7e1f19a8c74e90283e892b8973173eabd61`. That is the source baseline. The exact identity of a measured app must be established separately.

Phase One observed 21.62 FPS, a 46.25 ms frame interval, 36.29 ms of reported GPU time, 12.37 GB of application memory and 9.66 GB of Metal memory in Kokiri Forest. Game Mode was off. These are historical, uncontrolled measurements. A controlled M3 comparison and a pass-level GPU ranking are pending.

## Phase Two progress

- [x] Verify the fork and publish the baseline optimization branch.
- [x] Preserve the Phase One measurements and the five investigation areas.
- [ ] Verify an isolated launch with a copied Kokiri Forest save and fixed camera.
- [ ] Collect three ordinary baselines and a separate per-pass profile.
- [ ] Rank candidate changes from base-M3 evidence and select the first implementation.
- [ ] Obtain approval before substantial renderer changes or a full game rebuild.

## Technical record

- [Baseline](01-baseline.md)
- [Investigation](02-investigation.md)
- [Optimization plan](03-optimization-plan.md)
- [Benchmark protocol and results](04-benchmarks.md)
- [Engineering changelog](05-changelog.md)
- [Known issues](06-known-issues.md)

Keep game inputs, saves, generated shaders, captures and raw logs outside Git. Put new game-specific findings in the [game note](../../dm/games/0400486024DEC000/README.md) and link them from this project record.
