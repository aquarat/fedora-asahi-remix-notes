# Frigate NVR on Asahi: Vulkan detection and Apple AVD hardware decode

Moving a Frigate 0.18 NVR (five IP cameras, H.264 2304x1296 High profile) from
a headless macOS box to an M1 Mac mini running Fedora Asahi Remix 44, then
getting object detection onto the M1 GPU and H.264 decode onto the Apple Video
Decoder with GPU-side rescaling. One working day, 2026-09-05. Machine details:
[../machines/m1-mac-mini-fedora-44.md](../machines/m1-mac-mini-fedora-44.md).

## Outcome

Four live cameras at 5 fps detect rate, everything recorded continuously:

| | before (all CPU) | after |
|---|---|---|
| Frigate container, total | ~107% of one core | ~50–57% |
| per-camera ffmpeg (decode + scale to 640x360) | ~20% of a core each | ~5% each |
| object detector process | ~40% of a core | ~5% |
| whole machine at the wall | ~7 W (with KSM running) | ~5 W |

Measured with `docker stats`/`top` on the live system, 4 cameras, motion-gated
detection, 10-minute soaks. Per-clip numbers, all inside the Frigate image on a
150-frame 2304x1296 High-profile camera clip scaled to 640x360 (user+sys CPU):
software decode + swscale 0.59–0.64 s; AVD decode + read-back + swscale
0.48–0.53 s; AVD decode + zero-copy Vulkan scale 0.21–0.24 s. Decoded frames
are bit-identical to software decode across a 31-clip matrix; GPU-scaled frames
are 31.6 dB luma against swscale bicubic, which is bilinear-vs-bicubic, not an
error.

Detection latency per inference (320x320 YOLOv9-t, four classes): ONNX Runtime
CPU 12 ms idle but 30–130 ms under load; ncnn on Vulkan fp16 31 ms average,
p95 38 ms, flat under load. The GPU is not faster than NEON for a network this
small (about 650 dispatches at 35–100 µs each dominate), its value is offload.
rusticl/OpenCL was rejected: numerically wrong results and kernel compile
failures for these workloads.

## Where things are

| repository | what it holds |
|---|---|
| [aquarat/frigate-asahi](https://github.com/aquarat/frigate-asahi) (private) | The deployment, sanitised: compose, config example, the host detector service with its Vulkan backends, the ffmpeg wrapper, the driver build/test harness, every research write-up, and `REPORT.md` as the entry point. |
| [aquarat/apple-avd-driver](https://github.com/aquarat/apple-avd-driver) (private) | The AVD kernel driver as a standalone out-of-tree module tree. `main` = pristine asahi-7.1.6-1 driver + the eight patches; experiment branches kept for the record. |
| [aquarat/FFmpeg](https://github.com/aquarat/FFmpeg) branches `avd-readback`, `avd-readback-vk` (base `v4l2-request-n8.1` = Kwiboo's out-of-tree V4L2 Request hwaccel) | Four ffmpeg patches: cacheable V4L2 capture buffers (2), Vulkan import of linear multi-plane DRM frames, Honeykrisp plane-offset workaround. |
| [aquarat/frigate](https://github.com/aquarat/frigate) branch `apple-avd-vulkan` | ZMQ detector handshake retry; `preset-apple-avd` / `preset-apple-avd-vulkan` ffmpeg presets. |
| [aquarat/MNN](https://github.com/aquarat/MNN) branch `gcc16-build-fix` (push pending: needs a `workflow`-scoped token because MNN's history carries GitHub workflow files) | One-line CMake fix so MNN builds with GCC 16 (used for the alternative `mnn-vulkan` detector backend). |

Inside `frigate-asahi`, start with `REPORT.md`; the per-topic evidence is
`research/GPU_INFERENCE.md` (runtimes compared), `research/AVD_ROOTCAUSE.md`
(driver analysis), `research/AVD_READBACK.md`, `research/GPU_RESCALE.md`,
`research/AVD_FRIGATE_INTEGRATION.md` (end to end, including the incident
below), `research/UPSTREAM_REVIEW.md` (what each patch still needs before it
can be submitted), and `src/avd-driver/RESULTS.md` (per-configuration clip
matrices).

## The short version of what was learned

* **Four bugs in the 7.1.6 AVD driver stopped every real camera stream.**
  The PPS `transform_8x8_mode_flag` (0x40) was written raw into a 2-bit
  register field and masked to zero (upstream fixed it in 7.1.8); P-slice
  weighted-prediction denominators were wrong (upstream fix); the T8103
  instruction-FIFO mask had to be zero or high-bitrate/CABAC/large frames
  stalled mid-slice (upstream fix in 7.1.12); slices over ~16 KiB in
  multi-slice frames tripped a 1 ms poll timeout (new). A fifth: starting on a
  live stream mid-GOP made the decoder dereference an uninitialised reference
  slot, a DART fault and a 2 s watchdog reset on every ffmpeg start (new; the
  upstream review says other stateless decoders substitute a valid reference
  rather than rejecting, so the patch needs reshaping). The firmware was not at
  fault.
* **Hardware decode was a CPU loss until the read-back was fixed.** The
  decoder's capture buffers are coherent DMA allocations mapped Normal-NC, and
  reading them costs ~250 MB/s whatever copy loop you use (memcpy, NEON,
  non-temporal loads all measured the same). Standard V4L2 cache hinting
  (`allow_cache_hints` in the driver, `V4L2_MEMORY_FLAG_NON_COHERENT` from
  ffmpeg) fixed it. An in-kernel probe found zero stale cache lines in 450
  frames, so the block looks IO-coherent on T8103; DT `dma-coherent` is left as
  an RFC because no Apple node upstream carries it.
* **Zero-copy into Vulkan needed two generic ffmpeg fixes.** The Vulkan
  importer only understood VAAPI-style one-plane-per-layer DRM descriptors,
  not the single NV12 layer v4l2request exports; and Honeykrisp ignores
  `VkSubresourceLayout.offset` in `pPlaneLayouts` (only `rowPitch` is read in
  `hk_image.c`), so chroma came from offset zero. Passing plane offsets through
  `memoryOffset` works on every driver and needs no quirk; the Mesa bug still
  wants filing.
* **Running the host Mesa driver inside a Debian container is possible.**
  Bind-mount the host `/usr/lib64` and its `ld.so`, point `VK_DRIVER_FILES` at
  a copied ICD json, and exec ffmpeg through the host loader; the ICD needs
  glibc 2.38+ and Fedora's libstdc++, which bookworm's loader cannot satisfy.
  Without `VK_DRIVER_FILES` the image's own lvp ICD silently gives you
  llvmpipe. FFmpeg 8.1's Vulkan filters needed a statically built shaderc
  newer than bookworm's.
* **Frigate's record maintainer only trusts a process literally named
  `ffmpeg`.** Exec'd through the host loader the process was named after the
  loader, so Frigate deleted every in-progress recording segment as "missing
  video stream" for twenty minutes after the switch. Exec the loader through
  an `ffmpeg`-named symlink. The standalone test harness could never have
  caught this; only the live maintainer does.
* **Small-CNN inference on this GPU is dispatch-bound, not compute-bound.**
  clpeak measures ~2.5 TFLOPS fp32 = fp16 but ~100 µs per dispatch; fp16 and
  packing options change nothing for a 320x320 detector. Judge GPU inference
  here by CPU freed, not by latency.
* **`nvtop` has an Asahi backend but the kernel side isn't there yet**: no
  fdinfo on the `asahi` DRM driver in 7.1.6, so nothing to read.
