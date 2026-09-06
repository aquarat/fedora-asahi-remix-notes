# GPU inference on Honeykrisp: what the driver costs, what the runtimes cost

Following on from the [Frigate work](frigate-avd-vulkan.md): once object
detection ran on the M1 GPU through Mesa's Honeykrisp Vulkan driver, the
question was why it was slower than the CPU, whether the driver changes from
the [Ghost of Tsushima work](honeykrisp-ghost-of-tsushima.md) would help, and
whether transformer models (Immich's CLIP ViT-H-14-378) could run on the GPU
at all. One day, 2026-09-06, on the M1 Mac mini
([../machines/m1-mac-mini-fedora-44.md](../machines/m1-mac-mini-fedora-44.md)),
with the live NVR sharing the GPU throughout (light contention, roughly one
detector inference per second; every figure below carries it).

## Outcome

Three separate answers.

**The weak CDM barrier from got-bringup does nothing for inference.** A chain
test of 1000 back-to-back dispatches in one command buffer, stock Mesa 26.1.8
against the fork, loaded per process through a private ICD (never installed):

| chain shape | stock | patched |
|---|---:|---:|
| dependent (Vulkan barrier between every dispatch), empty shader | 3.47 µs/dispatch | 3.17 µs |
| dependent, shader with a 1000-iteration loop | 33.6 µs | 31.9 µs |
| independent (no barrier), empty shader | 3.30 µs | 0.32 µs |
| independent, shader loop | 31.6 µs | 0.71 µs |

10–45x where dispatches are independent, within noise where they depend on
each other. Inference graphs are dependent chains (ncnn puts a barrier after
every layer, MNN as shipped ends the command stream after every op), so
YOLOv9-t 320 measured 19.0 vs 19.0 ms on MNN-Vulkan and 36.3 vs 36.6 ms in a
detector A/B at 25 requests/s; outputs bit-identical. CLIP on MNN-Vulkan did
run 1.33–1.45x faster on the fork, but an ablation with the barrier and the
constant-table change both disabled was exactly as fast, and plain upstream
26.2.2 was as slow as 26.1.8, so that gain belongs to the fork's other
compiler commits. The driver was not deployed on the NVR host.

**The runtimes were leaving most of the GPU idle.** Profiled with the fork's
firmware profiler, `strace` and `perf`:

| finding | before | after |
|---|---|---|
| MNN-Vulkan recorded one `VkCommandBuffer` per op; Honeykrisp makes each a control stream with a ~23 µs gap. MNN's whole-graph mode exists but is unreachable because `ScheduleConfig::mode` is a union with `numThread`. Patch makes it the default. | 18.7 ms fp16 (21.2 fp32), 6.4 ms CPU per inference, ~474 control streams, GPU 50% busy | 11.6 ms fp16 (15.7 fp32), 0.7 ms CPU, ~3 streams, 78% busy, md5-identical output |
| ncnn kept SiLU as a separate layer after every convolution (a quarter of them as Split+Sigmoid+Mul). Patch adds Swish as `activation_type 7` on all backends plus an `ncnnoptimize` pass. | 655 layers, 978 dispatches, fp16 ~29.0 ms (loaded pass) | 388 layers, ~740 dispatches, 25.2 ms same pass; fp32 max abs diff vs onnxruntime 0.0016 (was 0.0021) |
| ncnn issued 2–3 `vkCmdPipelineBarrier` calls per dispatch. Patch coalesces them. | ~1680 barrier calls per inference | ~580; latency within noise (Honeykrisp already makes back-to-back barriers nearly free) |

What remains is kernel time: ~700 dispatches × 14 µs (MNN) or ~740 × 27 µs
(ncnn) over a 3.5 µs dependent-dispatch floor. The best GPU path (MNN image
fp16, batched, 11.6 ms at 0.75 ms of CPU) is still 2.4x the latency of NEON
fp16 (4.8–4.9 ms at ~20 thread-ms) on this chip; the GPU's value stays CPU
offload.

**ggml's Vulkan backend runs ViT-H efficiently on this driver.** The earlier
"Vulkan is hopeless for transformers here" (MNN-Vulkan ViT-H at 7 s, 2% of
peak) was a runtime finding. ggml records one command buffer per graph with
minimal barriers and uses tiled fp16 GEMM shaders (scalar paths: Honeykrisp
has no cooperative matrix):

| | ggml-Vulkan | for comparison |
|---|---:|---|
| Qwen2.5-0.5B Q8_0 prompt processing (llama-bench pp512) | 783 t/s ≈ 0.77 TFLOPS, ~30% of the 2.5 TFLOPS clpeak fp16 figure | ggml CPU Q8_0, 4 threads: 743 t/s |
| CLIP ViT-H-14-378, one 378 px image (1007 GFLOP) | 1.76–1.87 s, 566 GFLOPS = 23% of peak; 75 MB host RSS plus 0.68–1.27 GB of weights in unified memory | onnxruntime CPU fp32 3.7 s (3.2 s idle) at 2.5 GB RSS; MNN CPU fp16 1.9 s; MNN-Vulkan 5.8–7.7 s |
| ViT-B-32 / ViT-L-14 | 39 ms / 348 ms | ORT CPU 33 ms / 563 ms |

Cosine against ORT fp32 is 0.9985–0.9994 with fp16 shader arithmetic;
`GGML_VK_DISABLE_F16=1` gives 0.99999 at 3.1 s, still under ORT-CPU. No
existing ggml tool produced a CLIP embedding (llama.cpp's `mtmd` emits
projector patch tokens and drops `visual.proj`; monatis/clip.cpp is CPU-only
on a 2023 ggml), so a converter from Immich's own ONNX and a ~250-line runner
were written. Turning that into an Immich backend is separate, ongoing work.

## Where things are

| repository | what it holds |
|---|---|
| [aquarat/gpu-inference-asahi](https://github.com/aquarat/gpu-inference-asahi) (private) | This thread: the MNN and ncnn patches (`git am` on `bef71b9` / tag `20260526`), the ggml CLIP converter and runner with build notes for llama.cpp `9e0e220`, the chain test and per-process driver loading, benchmark sources, the three write-ups and the raw logs. Start with its `README.md`. |
| [aquarat/frigate-asahi](https://github.com/aquarat/frigate-asahi) | The NVR deployment and the first round of runtime evaluation (`research/GPU_INFERENCE.md`: ncnn/MNN/IREE/OpenCL compared; `research/GPU_RESCALE.md`: zero-copy Vulkan scaling of decoded frames), the host detector service these patches target. |
| [aquarat/got-bringup](https://github.com/aquarat/got-bringup), [aquarat/mesa](https://github.com/aquarat/mesa) `local-deploy` | The driver fork that was measured, its RPM, the `cstest`/`coherence` tests reused unchanged. |
| [aquarat/apple-avd-driver](https://github.com/aquarat/apple-avd-driver) | The other half of the GPU pipeline on this machine: the AVD decoder driver whose output the Vulkan scalers consume. |
| [aquarat/FFmpeg](https://github.com/aquarat/FFmpeg) `avd-readback`, `avd-readback-vk` | The ffmpeg side of that pipeline (cacheable capture buffers, Vulkan DRM import). |
| [aquarat/MNN](https://github.com/aquarat/MNN) (private, empty) | Intended home for the MNN branches (`gcc16-build-fix`, the batch-recording default). Still pending a `workflow`-scoped token: MNN's history carries `.github/workflows` files, which GitHub refuses from a `repo`-only token. The same commits are the patch files in `gpu-inference-asahi/mnn/`. |

## The short version of what was learned

* **Per-dispatch cost on Honeykrisp is ~3.2–3.5 µs for a dependent chain
  and ~80–160 µs for a submit/fence round trip.** Neither explains a 20 ms
  inference; submission structure and kernel efficiency do.
* **One command buffer per op is the single most expensive thing a runtime
  can do on this driver**: each becomes a GPU control stream with a ~23 µs
  gap and a DRM ioctl. Half of MNN's inference time was that. Check what a
  runtime's default mode actually records before tuning kernels.
* **Consecutive barriers with no work between them are free here.** A profile
  showing "2000 barriers" is not 2000 drains; the dependent-dispatch count
  is what matters.
* **Fusion pays only for the dispatches it removes.** Removing ncnn's
  element-wise SiLU dispatches (26% of dispatches) gave ~13% of time; the
  convolution kernels themselves (27 µs each vs MNN's 14 µs) are the ceiling.
* **Small CNN: CPU wins on latency, GPU wins on CPU time.** 4.8 ms NEON fp16
  vs 11.6 ms GPU, but 20 thread-ms vs 0.75 ms. Large-M fp16 GEMMs
  (ViT-H, LLM prompt processing) are where this GPU is 2x the CPU; int8
  GEMMs are a draw (the M1's `sdot` path is nearly as fast) and batch-1
  decode is faster on the CPU.
* **Relevance to the game work.** The runtime fixes here do not transfer:
  games already record whole frames into few command buffers and the
  got-bringup barrier change already covers their independent dispatches.
  What would transfer in the other direction is anything that cheapens the
  submit path (~80 µs per submit is visible to both) and cooperative-matrix
  support in Honeykrisp, which ggml would pick up automatically (its coopmat
  shaders are compiled in and selected at runtime) and which would lift the
  scalar fp16 GEMMs that cap both ViT-H at 23% of peak and, presumably, any
  compute-heavy game shader doing matrix work.
