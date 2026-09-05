# Ghost of Tsushima on Honeykrisp

Making *Ghost of Tsushima DIRECTOR'S CUT* (Steam 2215430) run efficiently on
an M1 Max under Fedora Asahi Remix 44, through muvm + FEX + Proton +
vkd3d-proton on the Honeykrisp Vulkan driver. Work done August to September
2026. Machine details: [../machines/m1-max-fedora-44.md](../machines/m1-max-fedora-44.md).

## Outcome

On the heaviest scene measured, GPU compute time per frame fell from 102.8 ms
to 18.6 ms, from two driver changes. The gains are strongly scene-dependent, so
there is deliberately no single headline number:

| fix | heavy scene | light scene |
|---|---|---|
| Dispatch overlap (weak CDM barrier between compute dispatches) | 5.53x compute | 1.45x |
| Constant tables lifted out of per-invocation scratch (`nir_opt_large_constants` plus GPU-side plumbing) | 2.03x compute | 2.70x |
| Cross-subqueue (render vs compute) overlap | noise | noise; default OFF |

Measured at pinned render resolution, gameplay only, per scene. Frame rate is
not a usable metric on this game because it targets 30 fps with dynamic
resolution and absorbs driver improvements as extra pixels; compute
milliseconds per frame at a matched scene reproduces to 1 to 2 percent. All
of this, including the earlier figures that were withdrawn, is in
`got-bringup/data/per-fix-results.md` and `data/measurement-hazards.md`.

## Where things are

### aquarat/got-bringup (harness, tests, record)

Start with [README.md](https://github.com/aquarat/got-bringup#readme), then:

| document | what it answers |
|---|---|
| `STATE.md` | where the work stands, the frame budget today, what was eliminated |
| `CHECKLIST.md` | the chronological log, 45 items, including every retraction |
| `INSTALL.md` | dnf repository and RPMs for the patched driver, and what enabling it means |
| `data/per-fix-results.md` | what each fix is worth, per scene, and what was withdrawn |
| `data/measurement-hazards.md` | dynamic resolution, two shader-cache-key bugs, scene variation |
| `data/game-settings-pinned.md` | pinning the render resolution through the Wine registry |
| `data/barrier-bit-cost.md` | measured per-bit cost and coherency role of the CDM barrier word |
| `data/subqueue-overlap.md` | cross-subqueue overlap: works, buys nothing here, and why |
| `data/compute-breakdown.md`, `data/fragment.md`, `data/draw-cost.md` | where GPU time goes, by phase |
| `data/what-is-left.md` | what the frame is bound by now |
| `TESSELLATION.md`, `CONFORMANCE.md` | the tessellation-in-compute optimisation work and its CTS runs |
| `CHANGES.md` | every change made to the machine during bring-up |
| `tests/` | standalone Vulkan programs, one question each |
| `ablate.sh`, `autorun.sh`, `reset-session.sh`, `cts-run.sh` | the ablation, the unattended run, session teardown, CTS runner |

### aquarat/mesa (driver)

`main` is upstream Mesa, untouched. Local branches:

| branch | what it carries |
|---|---|
| `local-deploy` | **the branch that ships.** 36 commits: dispatch overlap, constant-data lowering, cross-subqueue overlap (default off), `iadd(amul)` bounds-check lowering, the GPU-time profiler, shader-cache-key fixes, fragment shader interlock, compiler cleanup fixes, and the fixes from an external review. |
| `local-deploy-with-cdm` | an earlier stacking that used upstream MR 44041's barrier instead of the local one |
| `mr44041` | upstream merge request 44041, "use a lighter CDM barrier", as a single commit for comparison |
| `cdm-patched`, `cdm-baseline`, `cdm-ablate`, `cdm-ablate-harness` | the A/B trees used to bisect which barrier bits cost what |
| `cdm-light-unk2` | fragment shader interlock plus two compiler cleanup fixes, before the barrier work |
| `tess-scan`, `tess-tcspack`, `tess-barrier`, `tess-combo` | tessellation work: parallel prefix sum, packing TCS patches per workgroup, skipping the post-dispatch flush, and the three combined |
| `tess-count-fusion` | WIP: fuse the tessellation count pass into the TCS epilogue |
| `tess-scan-optB-rejected` | a subgroup-aggregated atomics variant, measured redundant, kept for the record |
| `latency-sched` | latency-aware scheduling and scoreboard slot assignment in the AGX compiler |
| `virtio-defer-mapping` | request deferred host mapping for blob buffer objects on the virtio path |

`got-bringup/mesa-source.env` pins the exact `local-deploy` commit the
published numbers came from and the packages are built from.

## The short version of what was learned

* The Asahi compute queue serialised every dispatch behind a full cache
  barrier. Bit 7 of the barrier word alone is enough for sequencing; the
  cache bits are what serialise, and only dependent dispatches need them.
* One hot shader built a 512-entry constant table in per-invocation scratch
  on every invocation. Mesa's `nir_opt_large_constants` fixes that, but the
  driver had no way to hand a shader a constant-data section; that plumbing
  is the second fix.
* Two separate bugs left debug and perf-test environment variables out of the
  shader cache key, so one experiment could poison every later measurement.
  Both were found by measurements that did not add up.
* The game's dynamic resolution hid every gain from the frame counter. Pin it
  before measuring anything.
* Render/compute cross-subqueue overlap is real and correct but worth nothing
  on this game, because nearly every render pass carries a driver-generated
  compute pre-pass it genuinely depends on.
