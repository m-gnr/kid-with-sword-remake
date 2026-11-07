# Investigation

The rewritten game runs on the host in `dm/host/`. NVN2 graphics commands are recorded and replayed onto Metal in [nvn_mtl.m](../../dm/gfx/nvn/nvn_mtl.m). The [shader build](../../dm/gfx/tools/dmsh_build.py) translates guest programs through SPIR-V and SPIRV-Cross to Metal libraries. [dmsh_meshstage.py](../../dm/gfx/tools/dmsh_meshstage.py) wraps task and mesh kernels for Metal. The game runs its own NDSR scaling before the host presents a drawable, which may introduce a second scaling stage.

The order below is the Phase One investigation order. The [base-M3 GPU profile](#base-m3-encoder-profile) ranks the work in the stationary scene.

## A. Foliage mesh shader

The build cache's `b-dc815e0f38ca.img` contains `626fbf48041cb82b_b.metal`. Read-only inspection with [shader_lib.py](../../dm/tools/shader_lib.py) confirms a threadgroup word CAS helper, packed index reads and threadgroup barriers. Reflection in the build and app identifies a 32-thread mesh function with no task stage for this key and both `main0` and `main0_z`. The [shader map](../../dm/games/0400486024DEC000/shader_map.tsv) identifies the foliage program.

Upstream [PR #1503](https://github.com/seedds/nxnative/pull/1503) and [commit ffebfd59342514d4fe947f140dd7b0844729cd92](https://github.com/seedds/nxnative/commit/ffebfd59342514d4fe947f140dd7b0844729cd92) identify two implementations. The M3 Pro pass went from 8.3–8.6 to 3.5–4.0 ms with the PR's intermediate atomic AND/OR form while the game changed from 854 × 480 to 1120 × 630. Simultaneous stores to the same byte can combine their values in that form. The merged implementation uses native `uchar` and alias-safe aligned `ushort` stores. Upstream reported 14.0–14.6 ms GPU p50 across the CAS, AND/OR and native variants on M6, with overlapping results. The merged native-store speedup on M3 is unknown.

The translator already lowers subword stores in `SharedStoreSmallIntCas` to `HelperFunctionManager.GenerateSharedStore`. The current helper has a `SharedStore*` name that does not survive as the upstream rewrite's stable MSL identifier. Port the producer name contract and `shared_store_msl` consumer together. The baseline lacks upstream's `split_params` and `block_end` helpers. Adapt the existing template-aware parameter parsing in `hmma_msl` or add the narrow parser needed for this rewrite. Insert the postpass in `finish` before mesh wrapping or `tg_spill`. Keep the stage memory layout, primitive indices and count, depth entry points, and producer/consumer barriers intact. The upstream `DM_NVN_SKIP_MESH` diagnostic is independent of the store rewrite. The [live-input foliage jitter fix](https://github.com/seedds/nxnative/pull/1343) addresses a separate compute issue.

## B. Texture residency and memory

The high footprint and the scene-entry upload are observations. Steady-state re-uploads, paging and residency churn are hypotheses. The historical log reports 21 cumulative texture re-uploads at frames 1500, 2100 and 2400, while GPU allocation held around 9,200 MiB. Residency additions fell from 2,442 MiB in the window ending at frame 1500 to 171 MiB at frame 2100. This suggests heavier earlier churn without resolving the remaining high footprint. In `nvn_mtl.m`, follow `pool_gpu` and `ptb_retire` for bindless tables, `alias_retire_idle` for texture backing, and `rel_push` / `rel_run` for guest storage release after upload completion. Distinguish guest pages, Metal backing and the lifetime of a finalized texture or a retained view. Compare depth-only compressible targets and resolution variants against the [rendering reference](../reference/rendering.md#live-resolution-variants-share-physical-image-storage). Start from the [game note](../../dm/games/0400486024DEC000/README.md) for known release and terrain regressions. [nxnative issue #1265](https://github.com/seedds/nxnative/issues/1265)

## C. MetalFX and scaling

`DM_UPSCALER` selects the game's NDSR path or the host's MetalFX replacement. `DM_PRESENT_UPSCALE` selects presentation filtering when game output and the drawable differ. The historical log shows a recognized `fa3033d05c0a41d6` replacement at 1920 × 1080 → 2560 × 1440, then an unrecognized `d99fbeeb53d38847` ending. Both keys belong to `agl_ndsr_vr_light`. At the unknown ending, `dispatch` sets `g_up_chain_refused` after first stages were skipped. The full game chain resumes the following frame. Presentation MetalFX stays independent. Use `DM_NVN_UPDUMP=1` on a diagnostic launch to identify bindings and sizes for the unrecognized variant before extending the chain. Compare one stage at a time at matching internal, game-output and drawable sizes.

## D. Compute scheduling

`dispatch_enc` and `dispatch_overlap` govern eligible consecutive game dispatches. The profile records a Docked M4 Pro result of 27.1 → 24.9 ms per GPU frame and 5.7 → 3.7 ms across an early 117-dispatch batch, attributed to fork commit `5208584a`. The base-M3 result is unknown. The switch is cached at first use and acts on consecutive eligible game dispatches. A game barrier, host dispatch, copy, fill or inline update keeps ordering. Use a new process for each setting. Inspect compute encoder intervals and waits, command-buffer segmentation and barriers before considering any broader synchronization change. [Fork notes](../games/kid-with-sword-remake.md)

## E. Frame pacing and presentation

Trace `fstep_tick` in [hle_nvn.c](../../dm/gfx/nvn/hle_nvn.c) and `link_sync`, `present_enqueue` and `present_now` in `nvn_mtl.m`. The historical log shows failed adaptive probes from three vsyncs back to one. A matched trace can separate GPU completion from acquire waits, display-link gaps and actual presentation. A simulation-rate fix would need to preserve real-time gameplay through refresh and frame-step changes. Compare presented timestamps with host submission rates and run a timed gameplay check before attributing any gain to pacing.

## Base-M3 encoder profile

`profile-2` used the same copied save and stationary view, with sparse encoder timestamps and a separate nearby pass dump. The 45.019 s diagnostic window covers frames 5485–6421. Four profiles in that window (5520, 5760, 6000 and 6240) had 34.36–34.78 ms between their first and last encoder. The gaps between instrumented encoder intervals were 1.38–1.65 ms. Those gaps do not identify GPU idle outside the profiled queue or prove a barrier dependency.

| Ranked workload | Base-M3 result | Attribution |
| --- | --- | --- |
| 854 × 480 color plus depth passes | 20.19 ms mean summed per frame across 16 passes. The longest individual pass was 17.03–17.70 ms and had 1,059–1,077 draws. | The nearby pass dump includes foliage programs, including mesh key `e6df2f14035e7be7` and foliage fragments from the same BNSH program as `626fbf48041cb82b`. It does not name that mesh key or time a specific draw. The whole-pass cost does not estimate the byte-store savings. |
| Compute and copy encoders | 9.48 ms mean summed per frame. The leading 115-dispatch batch took 5.53–5.71 ms. The late four-dispatch batch took 3.51–3.95 ms. | The nearby dump starts with foliage compute and shows four consecutive NDSR stages near the end. Per-dispatch attribution requires the separate serialized diagnostic. |
| Remaining rendering | About 3.36 ms mean summed per frame. The 128 × 128 color/depth group averaged 0.63 ms, and 1600 × 1600 depth-only passes averaged 0.54 ms. | Timing groups come from pass shape and attachment format. |

The game rendered the main 3D passes at 854 × 480, produced 2560 × 1440 and presented through MetalFX to 3600 × 2025. `DM_NVN_UPDUMP` reproduced the unrecognized `d99fbeeb53d38847` stage at 854 × 480 → 2560 × 1440. The fallback left the game NDSR chain active in the sampled frames. The host separately reported 2.48 ms p50 and 3.09 ms p95 on the last 600 presentation MetalFX command buffers. These timings can overlap with game rendering.

The diagnostic window recorded zero re-uploads and zero synchronous pipeline creations. Sampled peaks were 10,507 MiB physical footprint and 7,182 MiB Metal allocation. System swap continued to move. Those counters constrain the investigation for this view. A separate matched scene-transition run is needed to assess upload and release behavior.
