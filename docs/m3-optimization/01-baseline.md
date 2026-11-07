# Phase One baseline

The existing Mac application reached playable Kokiri Forest. Phase One used Docked console mode, Normal resolution, fullscreen, a 60 FPS target and Game Mode off on the base M3 with 16 GiB unified memory and macOS 26.6.2. The optimization source baseline is `e9c2f7e1f19a8c74e90283e892b8973173eabd61`. Phase One did not record the app binary hash, exact camera position, time window, refresh rate, thermal state or repeat count.

| Metric | Historical observation |
| --- | ---: |
| Observed FPS | 21.62 |
| Frame interval | 46.25 ms |
| Reported GPU duration | 36.29 ms |
| Application memory | 12.37 GB |
| Metal memory | 9.66 GB |
| System swap already in use when inspected | Approx. 7.6 GiB |
| Draws per frame in one window | Approx. 1,832 |
| CPU submission in one window | Approx. 5.48 ms |

| Window ending at frame | Host FPS | Frame p50 / p95 (ms) | Reported GPU p50 / p95 (ms) |
| ---: | ---: | ---: | ---: |
| 1500 | 21.3 | 47.96 / 55.28 | 37.06 / 45.48 |
| 2100 | 21.1 | 49.55 / 56.08 | 34.39 / 41.70 |
| 2400 | 19.8 | 49.78 / 74.85 | 34.85 / 38.68 |

The [profile](../../dm/games/0400486024DEC000/game.tsv) describes a 2048 × 1152 game presentation surface in Docked mode and 1920 × 1080 in Handheld. Those are configuration expectations. Phase One did not record the internal 3D dimensions, actual game output, final Metal drawable dimensions or display mode. Dynamic resolution can change the 3D input while the game output stays fixed.

The host GPU field can sum durations from overlapping command buffers. A 36.29 ms reading therefore does not establish the GPU critical path. Likewise, host FPS can differ from presented FPS. Physical footprint, Metal allocation and system swap are distinct measurements. Metal memory may overlap process accounting and should not be added to it. The swap level alone cannot attribute memory pressure to the game.

A large texture upload occurred on scene entry. Compare that transition separately from warm, stationary gameplay. Presented FPS, 1% lows, p99, GPU encoder span, pass attribution, displayed and discarded counts, and swap I/O deltas are unavailable for these observations.
