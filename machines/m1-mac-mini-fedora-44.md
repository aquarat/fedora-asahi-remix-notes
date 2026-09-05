# M1 Mac mini (2020), Fedora Asahi Remix 44

| | |
|---|---|
| hardware | Apple M1 (T8103, G13G B1 GPU), 16 GB, 2 TB NVMe |
| role | headless home NVR (Frigate in Docker) |
| OS | Fedora Linux Asahi Remix 44, server (no desktop) |
| kernel | 7.1.6-400.asahi.fc44.aarch64+16k, upgraded from 7.0.13 on 2026-09-05 for the in-tree Apple AVD video decoder driver; 7.0.13 kept as the GRUB fallback |
| system Mesa | 26.1.8 (Fedora); Honeykrisp Vulkan 1.4 and rusticl OpenCL 3.0 both work headless without any env vars, `/dev/dri/renderD128` is world-rw |
| Docker | 29.x, SELinux enforcing (bind mounts need `:z`; a `:ro` bind of `/usr/lib64` is readable by `container_t` without relabelling) |

## Apple AVD decoder: what the packaged kernel gives you

* `kernel-16k` ≥ 7.1.5 carries `apple-avd.ko` and the `apple,t8103-avd` DT
  node; 7.0.x has neither. The driver needs a ~2 KB MIT replacement CM3
  firmware from `AsahiLinux/avd-fw` (`apple/avd-fw-v2-t0.bin` on T8103) that
  **Fedora does not package**: build it with clang + meson and drop it into
  `/lib/firmware/apple/`. Without it the probe fails quietly and there is no
  `/dev/video0`.
* The 7.1.6 driver decodes Baseline and Main streams but fails on every real
  IP-camera stream (High profile 8x8 transform, large or high-bitrate frames,
  mid-GOP starts). Fixes are in the project write-up; this machine runs a
  patched out-of-tree module installed in
  `/lib/modules/<kver>/updates/apple-avd.ko`, which **must be rebuilt after
  every kernel-16k update, before rebooting**. The module is not in the
  initramfs, so no dracut step is needed. Undo = delete the file and run
  `depmod -a`.
* Device nodes are `root:video 0660`; host-side V4L2 tools need sudo or
  the `video` group; Docker `--device` passthrough works under SELinux.

## GPU monitoring: there is none yet

The `asahi` DRM driver on 7.1.6 does not implement fdinfo, so `nvtop` (which
has an Asahi backend from 3.1) reports "No GPU to monitor" and there is no
devfreq node either. What exists: `sudo cat /sys/kernel/debug/dri/128/clients`
(who holds the GPU) and the SMC power rails from `sensors macsmc_hwmon-isa-0000`
(Total System Power, AC Input Power, 3.8 V rail, fan). Four cameras decoding on
the AVD, scaling on the GPU and detecting on the GPU read about 5 W total.

## Things turned off

* KSM (`ksm.service`, a local `ksm-enable.service` tuning unit, and a
  tmpfiles rule writing `run=1`) was consuming ~55% of a core to merge ~230 MB
  on a machine with 10 GB free. Disabled 2026-09-05.

## Packages added for this work

`mesa-vulkan-drivers mesa-libOpenCL vulkan-tools clinfo ocl-icd clpeak nvtop
v4l-utils libva-utils gstreamer1-plugins-bad-free clang llvm meson ninja-build
kernel-16k-devel elfutils-libelf-devel dwarves python3.12 python3.13 uv golang
libva-devel libdrm-devel systemd-devel`. Note that installing `mesa-libOpenCL`
pulled a Mesa point upgrade (26.1.4 → 26.1.8) across the board.

## Project write-ups on this machine

* [Frigate NVR: Vulkan object detection and Apple AVD hardware decode](../projects/frigate-avd-vulkan.md)
