# Phase Four — Upstream Optimization Research and Integration Planning

## Executive summary and performance target

Phase Four investigates upstream optimizations in [seedds/nxnative](https://github.com/seedds/nxnative) and related community developments to establish an evidence-based integration plan for *The Legend of Zelda: Ocarina of Time* (Switch 2 release, title ID `0400486024DEC000`).

- **Single performance objective**: Normal + Handheld (1280 × 720 game output) on base Apple M3 (Mac15,3, 8-core CPU, 10-core GPU, 16 GiB unified memory).
- **Target framerate**: Minimum **40 actual presented FPS** (25.0 ms frame budget) with consistent frame pacing and correct gameplay speed; ideal target is **stable 60 actual presented FPS** (16.67 ms frame budget).
- **Quality and simulation constraints**: Preserve native rendering resolution, correct asset rendering, and visual quality. Do not use Potato presets, dynamic resolution degradation hacks, or simulation-speed alterations. Docked mode is retained solely as historical reference.
- **Current measured baseline**: Approximately **21–22 presented FPS** in Normal + Handheld, with **27–28 ms reported GPU time** and frequent **50.0 ms presentation intervals** (3 vsync holds on 60 Hz display cadence).

---

## Detailed comparison: Upstream vs. our fork

Every targeted pull request, commit, and recent relevant upstream development was verified against the local source tree (`dm/gfx/nvn/nvn_mtl.m`, `dm/gfx/tools/`, `dm/games/0400486024DEC000/`) and Git history.

```
Upstream Changes Evaluated
├── Already in our fork (commit ad998585 / 7781d946):
│   ├── PR #1455: Idle shared texture retirement (alias_retire_idle, ws_bytes)
│   ├── PR #1447: GPU-written ranges (DM_NVN_GPUW_RANGE) & guest texture release (DM_TEX_RELEASE)
│   ├── PR #1394: Raw global memory pool filtering (mtl_pool_buffer_addr)
│   ├── PR #1439: Storage takeover serial lookup (g_by_serial_held)
│   └── PR #1489 (Partial): Bindless pool table patching (ee9b7765, 7781d946)
└── Missing from our fork (Candidates for Phase Four / Five):
    ├── Commit ffebfd59 (PR #1503): Native subword threadgroup stores (uchar/ushort stores)
    ├── PR #1590: PSO archive epoch separation, lookahead return stack, static pool arrays
    ├── Commit e5f1adee (PR #1673): Mesh stage [[position, invariant]] foliage fix
    └── PR #1489 (Remainder): Complete REG_POOL notes and dead/transient ID retake
```

### 1. PR #1455 — Shared texture retirement under memory pressure
- **Upstream source**: [nxnative PR #1455](https://github.com/seedds/nxnative/pull/1455) by PromethOps (`retire-idle-shared-textures`).
- **Problem addressed**: The game retains ~12,000 finalized texture descriptors in texture pools. After the dialogue with Saria in Kokiri Forest, 2,000–2,500 shared textures lost their live objects simultaneously. Textures expanded from 5 GB to 10–16 GB, exceeding the working set of 16 GB and 24 GB Macs and triggering GPU page faults, kernel hangs, or system freezes.
- **Local fork status**: **ALREADY IMPLEMENTED**.
  - Local implementation: Fully integrated in [dm/gfx/nvn/nvn_mtl.m](../../dm/gfx/nvn/nvn_mtl.m) via commit `ad998585` ("Ocarina of Time: upstream fixes, memory cuts for the iPhone, IPA game-file replacements").
  - Identifiers verified: `ws_bytes()` (`DM_NVN_WS_MB` / `recommendedMaxWorkingSetSize`), `alias_retire_idle()`, `g_alias_lseen`, `g_alias_gone`, `tex_init_fill(..., again)`, and test fixture `t_alias_retire`.
- **Performance classification**: Memory footprint and crash prevention. Does not improve steady-state presented FPS, but is mandatory for stability across the Saria transition.

### 2. PR #1447 — GPU-written texture ranges and guest copy release
- **Upstream source**: [nxnative PR #1447](https://github.com/seedds/nxnative/pull/1447) by Peak8D (`oot-host-perf-memory`).
- **Problem addressed**:
  1. *Terrain holes*: `TerrainGround` patches (260 × 260 R32F heightmaps, CPU-written) share a ~200 MB pool with render targets. Under the previous whole-pool rule, the first GPU write marked the entire pool GPU-written, causing heightmap patches to read as zeros and leaving transparent terrain holes (e.g., at the Kokiri Forest pond).
  2. *Guest memory waste*: Approximately 3.1 GB of guest memory held textures already uploaded to Metal private textures.
- **Local fork status**: **ALREADY IMPLEMENTED**.
  - Local implementation: Integrated in [nvn_mtl.m](../../dm/gfx/nvn/nvn_mtl.m) (`mtl_tex_gpu_written`, `DM_NVN_GPUW_RANGE=1`, lines 3568–3573; `DM_TEX_RELEASE` and `rel_mode()`, lines 2631–2665).
  - Note on level selection: Documented in [game note](../../dm/games/0400486024DEC000/README.md#found-on-this-game), `DM_TEX_RELEASE=2` caused magenta/black textures over ground/walls on area transitions; `DM_TEX_RELEASE=1` (CPU_NO_ACCESS pools) is the verified safe default.
- **Performance classification**: Graphical correctness (terrain holes) and host RAM footprint reduction (~2.3 GB reclaimed). No direct presented FPS uplift.

### 3. PR #1394 — Raw global memory pool filtering
- **Upstream source**: [nxnative PR #1394](https://github.com/seedds/nxnative/pull/1394) by david-on-spring (`pr/gmem-buffer-pools`).
- **Problem addressed**: Any raw global memory shader could theoretically load any memory address, causing `gmem_map_gpu` to list every memory pool (>4,500 pools, mostly textures) and wire them into the residency set. On 16 GB Macs, residency reached 11.7 GB and frame times inside Link's house inflated to 60–65 ms (~15 FPS).
- **Local fork status**: **ALREADY IMPLEMENTED**.
  - Local implementation: Present in [nvn_mtl.m](../../dm/gfx/nvn/nvn_mtl.m) (`mtl_pool_buffer_addr`, line 8996), [hle_nvn.c](../../dm/gfx/nvn/hle_nvn.c) (lines 286, 562, 2199), and verified by test `t_gmem_marks.m`. Integrated via commit `ad998585`.
- **Performance classification**: Eliminates the catastrophic 60–65 ms stall in Link's house, lowering frame time to ~26 ms. Already active in our baseline.

### 4. PR #1439 — Storage takeover live object lookup
- **Upstream source**: [nxnative PR #1439](https://github.com/seedds/nxnative/pull/1439) by enjoyerofslop (`perf/alias-takeover-held`).
- **Problem addressed**: In `alias_take_over` (`nvn_mtl.m`), each new shared texture checked overlapping entries by executing a linear walk of the entire 7,500-entry serial table (`serial_holds`). In cutscenes with rapid texture creation, this consumed 61% of submit thread time and produced 100–1000 ms frame spikes (`[longframe]`).
- **Local fork status**: **ALREADY IMPLEMENTED**.
  - Local implementation: `g_by_serial_held` hashmap lookup (`serial_table_holds`) in [nvn_mtl.m](../../dm/gfx/nvn/nvn_mtl.m) lines 2005–2013, integrated in `ad998585`.
- **Performance classification**: Eliminates CPU submit thread stalls during cutscenes. No impact on steady-state GPU frame time.

### 5. PR #1489 — Incremental pool table notes
- **Upstream source**: [nxnative PR #1489](https://github.com/seedds/nxnative/pull/1489) by PucaVaz & TheekshanaA (`pool-table-notes`, base commit `0c7e88b9`).
- **Problem addressed**: Streaming textures and frequent registrations caused `pool_gpu` to rebuild the entire 33,025-entry texture handle table up to 61,000 times during a 7-minute walk, consuming ~53 seconds of CPU time.
- **Local fork status**: **PARTIALLY IMPLEMENTED**.
  - What our fork has: Rebased table patching from #1299 (`ee9b7765`), table buffer reuse (`a3844dc6`), and the storage takeover note with `pool_skip_retired` (`7781d946`).
  - What is missing: The complete upstream `REG_POOL` note for memory pool additions/removals, dead-ID list refresh (`pool_snap.dead`), and next-submission transient-ID retake (`pool_snap.trans`).
- **Performance classification**: CPU submission time reduction (measured 12–15 ms → 3–5 ms submit CPU p50 on M4) and frame pacing tail latency (reduces >50 ms spikes during walking). GPU duration and median FPS are unaffected because the GPU is the throughput bottleneck.

### 6. PR #1590 — Ocarina playthrough, pipeline archive, and renderer fixes
- **Upstream source**: [nxnative PR #1590](https://github.com/seedds/nxnative/pull/1590) by ksolenk38ze (`oot-playthrough-fixes`, commits `85599141` and `708f16ef`).
- **Problem addressed**:
  1. *Pipeline archive compilation hitches*: `pso_save_base` shared a single flush state (`g_base_epoch`) between pulling bases (`vd == 0`) and vertex-layout bases (`vd != 0`). Offline archive compaction (`pso_compact_dir`) discarded pulling bases from `%05d.mtl4a`, forcing subsequent game launches to repeatedly compile pipelines at runtime.
  2. *Sub-command lookahead stalls*: `shader_prefetch_rest` and `pso_predict_rest_` failed to cross `OP_CALL` sub-list boundaries because the return stack was not copied.
  3. *Stack overflow risk*: `pool_gpu` scratch arrays (1.96 MB total) lived on the 1 MB `dm_bigstack`.
  4. *Render correctness*: Missing depth-read render pass boundaries caused black flickering blocks on translucent surfaces (Zora's Sapphire pedestal); unpadded mesh varyings caused `MTL4Compiler` aborts; and primitive attribute block interleaving corrupted Forest Temple painting shaders.
- **Local fork status**: **MISSING**.
  - None of PR #1590's host archive epoch splitting, lookahead return stack copying, static pool patch arrays, or depth-read pass splits exist in our branch.
- **Performance classification**: Eliminates compilation stutter, protects pipeline archives from compaction data loss, and stabilizes 1% lows. Host renderer changes require zero shader rebuilds.

### 7. PR #1663 — Experimental playability and performance checkpoint
- **Upstream source**: [nxnative PR #1663](https://github.com/seedds/nxnative/pull/1663) by bmjask (`codex/ocarina-playability`, commits `971d5e4c`, `621f5c9b`, `ad2b936b`).
- **Problem addressed**: Attempted broad reconciliation of experimental raw quad ranges, HMMA matrix operations, HID vibration queries, and save editor additions.
- **Local fork status**: **INTENTIONALLY DEFERRED / SEPARATED**.
  - Parts exist on the sibling branch `origin/v9-oot2-shaders`, but were excluded from `perf/apple-m3-oot-docked` due to documented instability on Apple Silicon.
  - Upstream PR author explicitly marked this as an incomplete experimental checkpoint with unresolved conflicts against PR #1590.
- **Performance classification**: High regression risk. Broad merge not recommended.

### 8. Commit `ffebfd59` (PR #1503) — Native subword threadgroup stores
- **Upstream source**: [nxnative commit ffebfd59342514d4fe947f140dd7b0844729cd92](https://github.com/seedds/nxnative/commit/ffebfd59342514d4fe947f140dd7b0844729cd92) by seedds / Romen-Adama-Dev (`chain f9f01e4f9994`, PR #1503).
- **Problem addressed**: Task and mesh shaders storing byte or half-word primitive indices (STS.U8 / STS.U16) were translated to an atomic compare-and-swap (CAS) loop on 32-bit words because SPIR-V represents shared memory as a `uint` array. In mesh workgroups of 32 threads, 4 adjacent threads write neighbouring bytes within the same word, resulting in intense atomic CAS contention and ~4 retries per store.
  - In Kokiri Forest, 11 foliage mesh shaders execute ten such stores per lane.
  - On M3 Pro, upstream measured the foliage pass dropping from **8.3–8.6 ms to 3.5–4.0 ms** (saving ~4.5 ms GPU time).
  - The merged commit lowers `dmsh_shared_store8_<id>` / `dmsh_shared_store16_<id>` to native Metal `uchar` and `__attribute__((may_alias)) ushort` threadgroup stores, avoiding atomic retries while preserving single-store semantics.
- **Local fork status**: **MISSING**.
  - Local inspection of `b-dc815e0f38ca.img` and all 11 active foliage mesh shaders in Kokiri Forest (e.g. `17c8ab6419068d4b`, `73dc0b90c4082e43`, `eed1add7c89f5d34`) confirms that our build still executes the atomic CAS loop.
- **Performance classification**: **Direct GPU execution time reduction**. This is the single largest identified GPU throughput optimization for Kokiri Forest.

### 9. Commit `e5f1adee` (PR #1673) — Mesh shader position invariance
- **Upstream source**: [nxnative commit e5f1adee](https://github.com/seedds/nxnative/commit/e5f1adee) by cocoman93 (PR #1673).
- **Problem addressed**: The game renders foliage twice: depth pre-pass (depth LESS, no color) and color pass (depth EQUAL, no depth write). Vertex shaders declare `[[position, invariant]]`, but the mesh stage wrapper in `dmsh_meshstage.py` declared a plain `[[position]]`. Metal compiled the position math of the two mesh shaders to slightly different depths, causing color passes to miss EQUAL and foliage to flash white for single frames.
- **Local fork status**: **MISSING**.
- **Performance classification**: Visual correctness. Directly resolves the known Kokiri Forest white foliage flashing issue.

---

## Analysis of community approaches

1. **Base M4 / 24 GB reports (PucaVaz: 39.5–45.2 FPS)**:
   - Evaluated in [07-base-m4-investigation.md](07-base-m4-investigation.md). The reported 39.5 FPS outdoors / 45.2 FPS indoors on base M4 / 24 GB was achieved with Handheld opmode, upscale 2.0, and an accumulated stack of PRs (#1453, #1455, #1446, #1489).
   - The primary mechanism enabling higher FPS on M4 was a reduction in CPU submission overhead and pass splits, combined with M4's higher GPU clock and unified memory bandwidth (120 GB/s on M4 vs 100 GB/s on M3).
   - No matched 20 → 40–45 FPS jump was demonstrated from a single patch, and no community evidence demonstrates 60 FPS in Kokiri Forest on base M3.
2. **Community base M3 / 10-core GPU 30 FPS configuration**:
   - Multiple community reports on r/macgaming describe achieving smooth playability on base M3 by selecting a **30 FPS cap**.
   - In our fork, commit `2de238f3` introduced `DM_FPS_TARGET=30` / `60` interfacing with the game's internal frame-rate controller.
   - At a 30 FPS cap, the frame interval budget is 33.33 ms. Because base M3 GPU time in Handheld is ~27–28 ms, a 30 FPS cap comfortably fits within the 33.33 ms window, eliminating triple-buffering drops down to 50 ms (20 FPS).
   - However, our project objective is **minimum 40 actual presented FPS (25.0 ms budget) and ideal 60 FPS (16.67 ms budget)**. A 30 FPS cap is an operational fallback, not a solution to the 40/60 FPS objective.
3. **Simulation speed / frame-delta hooks**:
   - Some community patches (`oot-mac` frame-delta) alter game simulation tick timing when frame times drop. This violates our strict project requirement ("Do not artificially increase FPS by changing game simulation speed"). All optimizations must preserve correct real-time simulation.

---

## Build artifacts and rebuild cost minimization

A critical requirement is determining whether the full ~42,854-entry shader rebuild can be avoided.

```
Build and Rebuild Scope
├── Host-only optimizations (No shader rebuild):
│   ├── Target files: dm/gfx/nvn/nvn_mtl.m, hle_nvn.c, game.tsv
│   ├── Rebuild cost: make -C dm/host (~10 seconds)
│   ├── Shader impact: 0 shaders rebuilt; reuses b-dc815e0f38ca.img (11 GB) directly
│   └── Covers: PR #1590 archive/lookahead, PR #1489 table notes, presentation pacing
└── Shader chain optimizations (Requires chain bump):
    ├── Target files: dmsh_build.py, HelperFunctionManager.cs, dmsh_meshstage.py
    ├── Chain version: dmsh_chain.py computes new 12-char hash; bumps dmsh_chain.h
    ├── Recompilation scope: Only 515 shaders out of 42,848 have subword stores
    └── Cache reuse: cas.db (13 GB CAS database) caches unchanged MSL/metallib
```

### Key findings from inspecting local build artifacts

1. **Existing build artifacts**:
   - Location: `/Volumes/MacNativeGames/OcarinaOfTime/0400486024DEC000_build/`
   - Shader image: `shader/cache/b-dc815e0f38ca.img` (11 GiB, 42,854 shaders).
   - Compilation database: `shader/cache/cas.db` (13 GiB SQLite/CAS database).
   - Pipeline archives: `pso/` contains 3.2 GiB of bundled Metal 4 pipeline caches.
2. **Reusing existing shader libraries**:
   - The host runtime loads libraries keyed strictly by `DMSH_CHAIN` in [dm/gfx/nvn/dmsh_chain.h](../../dm/gfx/nvn/dmsh_chain.h) (currently `"dc815e0f38ca"`).
   - As defined in [dmsh_chain.py](../../dm/gfx/tools/dmsh_chain.py), the chain hash covers only translator tools, ABI definitions, and compiler lock files.
   - **Host-only changes (in `nvn_mtl.m`, `hle_nvn.c`, `game.tsv`) do NOT alter `DMSH_CHAIN`**.
   - Therefore, host-only optimizations can be built and evaluated in seconds (`make -C dm/host`) while reusing 100% of the existing 42,854 compiled shaders without regenerating a single shader.
3. **Incremental shader compilation via `cas.db`**:
   - When a tool change does alter the chain (e.g. `shared_store_msl` in `dmsh_build.py`), `cas.db` keys Metal compilation by MSL content hash.
   - For commit `ffebfd59`, upstream logged: **only 515 shaders out of 42,848 contain the subword store** (500 mesh shaders, 15 compute kernels).
   - The remaining 42,333 shaders produce identical MSL and hit `cas.db`, avoiding Metal recompilation.
4. **Isolated test application workspace**:
   - APFS copy-on-write cloning (`cp -cRp`) creates an isolated test app in seconds (sharing disk blocks with the 34 GiB base installation).

---

## Ranked optimization candidates for base M3 Normal + Handheld

Candidates are ranked across the six required criteria:
1. Expected benefit for Normal + Handheld presented FPS.
2. Quality of supporting evidence.
3. Compatibility.
4. Incremental build feasibility.
5. Implementation effort.
6. Regression risk.

| Rank | Candidate | Mechanism & source | Handheld FPS impact | Evidence quality | Build feasibility & effort | Regression risk |
|:---:|---|---|---|---|---|:---:|
| **1** | **Native subword threadgroup stores** | Replace atomic CAS loops with direct `uchar*`/`ushort*` stores in 11 foliage mesh shaders. Upstream `ffebfd59` / PR #1503. | **High (Direct GPU win)**: Shaves ~3–5 ms off the ~17 ms Kokiri Forest pass; primary path to lowering GPU time below 25 ms. | **High**: Measured 8.3–8.6 → 3.5–4.0 ms on M3 Pro. Merged upstream. | Medium: Bumps chain; 515 shaders recompiled via `cas.db`. | Low (verified single-store semantics) |
| **2** | **PSO archive epoch separation & lookahead** | Separate pulling bases (`vd==0`) from vertex bases (`vd!=0`) in `pso_save_base`; copy `OP_CALL` return stack. Upstream PR #1590 (`85599141`). | **High (Pacing & Stutter)**: Eliminates pipeline compilation stalls and 1% low drops; prevents compaction data loss. | **High**: Solves documented archive miss stall in PR #1590 and #1663. | **Outstanding**: Host-only; 0 shaders rebuilt; ~10 s recompile. | Very low (additive cache logic) |
| **3** | **Incremental pool table notes (`REG_POOL`)** | Add `REG_POOL` notes and dead/transient ID retakes to avoid full-table rebuilds. Upstream PR #1489 (`0c7e88b9`). | **Moderate (CPU submit win)**: Reduces host CPU submission from ~10 ms to ~3–5 ms; smooths frame pacing. | **Strong**: Measured 12–15 → 3–5 ms submit CPU p50 on M4. | **Outstanding**: Host-only; 0 shaders rebuilt. | Low |
| 4 | Presentation cadence & whole-vsync adaptation | Tune `fstep_tick` and display synchronization for <25 ms frames to prevent 50 ms triple-buffering lock. | Moderate to High: Allows 40 FPS delivery once GPU duration drops <25 ms. | Moderate: Historical observations show 50 ms holds at ~27 ms GPU time. | Outstanding: Host-only. | Medium (must preserve game speed) |
| 5 | Mesh shader position invariance | Add `[[position, invariant]]` to mesh stage wrapper. Upstream `e5f1adee` / PR #1673. | None on FPS; **High on visual correctness**: Eliminates white foliage flashes. | High: Upstream verified on M4 Max/Pro. | Medium: Bumps chain. | Very low |

---

## Recommended integration sequence

### Smallest independently verifiable first optimization: Candidate 2 (Host-only PSO archive epoch separation)

- **Target**: Integrate the pulling base separation (`g_base_epoch` vs `g_pull_epoch`) and compacted shard ordering from PR #1590 (`85599141`) in [dm/gfx/nvn/nvn_mtl.m](../../dm/gfx/nvn/nvn_mtl.m).
- **Why this first**:
  1. It requires **zero shader rebuilding** and reuses the existing 42,854-shader library and `pso_archive`.
  2. It can be built and verified in an isolated APFS clone in seconds.
  3. It prevents pipeline archive degradation during compaction and immediately improves frame pacing and 1% lows.
- **Verification protocol**:
  1. Build host in test worktree (`make -C dm/host GAME=0400486024DEC000`).
  2. Launch isolated Handheld run; verify `pso_save_base` outputs pulling bases to `p%05d.mtl4a`.
  3. Execute compaction; confirm `p*.mtl4a` files retain pulling bases and subsequent launch prebuild logs zero pulling misses.

### Subsequent phased roadmap (Pending approval)

- **Phase 5A**: Integrate Candidate 3 (PR #1489 incremental pool table notes remainder) into `nvn_mtl.m`. Verify submit CPU drop in TSV output.
- **Phase 5B**: Port Candidate 1 (commit `ffebfd59` native subword stores) and Candidate 5 (`e5f1adee` position invariance). Rebuild the 515 affected shaders using `cas.db`. Conduct controlled Normal + Handheld A/B benchmark in Kokiri Forest to measure GPU duration reduction.
- **Phase 5C**: Evaluate presentation cadence tuning to unlock 40 FPS presentation pacing for the sub-25 ms GPU frames.

---

## Technical risks and limitations

1. **Base M3 thermal envelope**: The base M3 has 10 GPU cores and a single cooling fan (or passive cooling on MacBook Air). Sustained high-frequency GPU execution can induce thermal throttling after several minutes. Benchmark protocols must continue to record thermal status.
2. **Dynamic resolution interactions**: In Normal + Handheld, the game's internal resolution scaler adjusts 3D render dimensions dynamically. All comparisons must record actual 3D render dimensions via pass dumps.
3. **Simulation rate invariance**: Under no circumstances should frame-step multipliers or simulation tick deltas be altered. All presented FPS gains must reflect true throughput improvements.
