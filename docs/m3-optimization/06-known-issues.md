# Known issues

| Issue | Severity and status | Reproduction and evidence |
| --- | --- | --- |
| Docked fullscreen performance below target | High, open | Phase One observed 21.62 FPS in Kokiri Forest. Reproduce with a fixed camera and presented timestamps. Pass ranking and repeatability are unverified. |
| Unified memory pressure and scene-entry upload | High, open | Load Kokiri Forest and measure the transition separately. Phase One observed 12.37 GB of app memory, 9.66 GB Metal memory and about 7.6 GiB system swap in use. Swap causality and steady-state churn are unknown. |
| Blank or magenta textures after area changes | High, documented in game note, local status unverified | Test crossing Hyrule Castle Town and Hyrule Field. The [game note](../../dm/games/0400486024DEC000/README.md#found-on-this-game) reports textures remade over released pages with `DM_TEX_RELEASE=2`. Establish the app's effective level before changing it. |
| Foliage jitter, artifacts and shader failures | Medium, historical or upstream, local status unverified | Watch foliage at rest and while turning in Kokiri Forest. Check missing-shader diagnostics and geometry in color and depth. The [game note](../../dm/games/0400486024DEC000/README.md#shaders) tracks task and mesh shader issues. The byte-store performance candidate is separate from [the live-input foliage jitter fix](https://github.com/seedds/nxnative/pull/1343). |
| Unrecognized NDSR chain ending | Medium, open | Reproduce with the same settings and record the ending key, input/output sizes and fallback behavior. Phase One did not establish the trigger. |
| Save compatibility across builds | High if confirmed, unverified | Record the fixture's game build and load a fresh copied save. Keep its profile, slot and both metadata and progress files together. Confirm that the experimental build can load and save independently. No incompatibility was observed in Phase One. |
| Upstream translator dependency | Medium, open | Verify helper signatures and the pinned translator artifact before a native-store backport. A shader chain mismatch can reject libraries and invalidate an A/B. |
