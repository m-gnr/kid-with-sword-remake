# Investigation

The rewritten game runs on the host in `dm/host/`. NVN2 graphics commands are recorded and replayed onto Metal in [nvn_mtl.m](../../dm/gfx/nvn/nvn_mtl.m). The [shader build](../../dm/gfx/tools/dmsh_build.py) translates guest programs through SPIR-V and SPIRV-Cross to Metal libraries. [dmsh_meshstage.py](../../dm/gfx/tools/dmsh_meshstage.py) wraps task and mesh kernels for Metal. The game runs its own NDSR scaling before the host presents a drawable, which may introduce a second scaling stage.

The order below is the Phase One investigation order. Base-M3 pass durations are unavailable.

## A. Foliage mesh shader

Phase One identified mesh key `626fbf48041cb82b` in the [shader map](../../dm/games/0400486024DEC000/shader_map.tsv) and observed a threadgroup byte-store CAS loop in generated MSL. nxnative has a native subword store rewrite in `shared_store_msl`. Phase One reports about 8.3–8.6 ms for the relevant pass before that change on an M3 Pro. The pass cost includes other work and does not estimate the CAS cost on base M3. [nxnative PR #1503](https://github.com/seedds/nxnative/pull/1503), [commit ffebfd593425](https://github.com/seedds/nxnative/commit/ffebfd59342514d4fe947f140dd7b0844729cd92)

Identify the exact patch and dependencies, then check which calls are eligible for the rewrite and where they feed primitive indices. Separate this mesh-store work from the foliage compute jitter fix in [nxnative PR #1343](https://github.com/seedds/nxnative/pull/1343).

## B. Texture residency and memory

The high footprint and the scene-entry upload are observations. Steady-state re-uploads, paging and residency churn are hypotheses. In `nvn_mtl.m`, follow `pool_gpu` for bindless tables and `rel_mode` and its neighboring release code for guest storage. Distinguish guest pages, Metal backing and the lifetime of a finalized texture or a retained view. Compare depth-only compressible targets and resolution variants against the [rendering reference](../reference/rendering.md#live-resolution-variants-share-physical-image-storage). Start from the [game note](../../dm/games/0400486024DEC000/README.md) for known release and terrain regressions. [nxnative issue #1265](https://github.com/seedds/nxnative/issues/1265)

## C. MetalFX and scaling

`DM_UPSCALER` selects the game's NDSR path or the host's MetalFX replacement. `DM_PRESENT_UPSCALE` selects presentation filtering when game output and the drawable differ. Check the chain roles at `up_chain_role` and the MetalFX handoff. Phase One observed an unrecognized NDSR ending. Its key, dimensions and actual fallback path require a correlated log and capture. Compare one stage at a time at matching internal, game-output and drawable sizes.

## D. Compute scheduling

`dispatch_enc` and `dispatch_overlap` govern eligible consecutive game dispatches. The profile records a Docked M4 Pro result of 27.1 → 24.9 ms per GPU frame and 5.7 → 3.7 ms across an early 117-dispatch batch, attributed to fork commit `5208584a`. The base-M3 result is unknown. Inspect compute encoder intervals and waits, command-buffer segmentation and barriers before considering any broader synchronization change. [Fork notes](../games/kid-with-sword-remake.md)

## E. Frame pacing and presentation

Trace `fstep_tick`, display-link scheduling, acquire/present waits and actual presentation feedback. A simulation-rate fix would need to preserve real-time gameplay through refresh and frame-step changes. Compare presented timestamps with host submission rates and run a timed gameplay check before attributing any gain to pacing.
