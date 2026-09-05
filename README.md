# Fedora Asahi Remix notes

An index of work done on Apple Silicon machines running Fedora Asahi Remix:
what was changed, where the code and the evidence live, and what was learned.
The repositories linked from here hold the actual work; this one exists so
that there is a single place to start from.

## Index

| area | machine | where |
|---|---|---|
| Honeykrisp (Asahi Vulkan) driver work for Ghost of Tsushima under muvm/FEX/Proton | M1 Max | [projects/honeykrisp-ghost-of-tsushima.md](projects/honeykrisp-ghost-of-tsushima.md) |
| Host tuning for muvm gaming: transparent huge pages | M1 Max | [machines/m1-max-fedora-44.md](machines/m1-max-fedora-44.md) |
| Frigate NVR: object detection on Vulkan, H.264 decode on the Apple AVD with GPU rescaling; four AVD driver bugs, ffmpeg Vulkan import fixes | M1 Mac mini | [projects/frigate-avd-vulkan.md](projects/frigate-avd-vulkan.md) |
| Headless NVR host: AVD driver/firmware on the packaged kernel, out-of-tree module upkeep, GPU monitoring, KSM | M1 Mac mini | [machines/m1-mac-mini-fedora-44.md](machines/m1-mac-mini-fedora-44.md) |

Notes for other machines go under `machines/`, one file per machine, and
project write-ups under `projects/`. Add a row above for each.

## Repositories

| repository | what it is |
|---|---|
| [aquarat/mesa](https://github.com/aquarat/mesa) | Mesa fork. `main` tracks upstream unchanged; every branch listed in the project page is local Asahi/Honeykrisp work on top of it. |
| [aquarat/got-bringup](https://github.com/aquarat/got-bringup) | The measurement harness, standalone Vulkan tests, the written record, and a dnf repository for the patched driver. |
| [aquarat/frigate-asahi](https://github.com/aquarat/frigate-asahi) (private) | The Frigate-on-Asahi deployment, sanitised, with the driver/ffmpeg patch sets, harness, research write-ups and the end-to-end report. |
| [aquarat/apple-avd-driver](https://github.com/aquarat/apple-avd-driver) (private) | The Apple AVD V4L2 kernel driver as an out-of-tree module tree with the local fixes on top of asahi-7.1.6-1. |
| [aquarat/FFmpeg](https://github.com/aquarat/FFmpeg) | FFmpeg fork. `master` tracks upstream; `v4l2-request-n8.1` is Kwiboo's hwaccel base and `avd-readback`/`avd-readback-vk` carry the local V4L2 and Vulkan patches. |
| [aquarat/frigate](https://github.com/aquarat/frigate) | Frigate fork; `apple-avd-vulkan` carries the presets and the detector retry. |
| [aquarat/MNN](https://github.com/aquarat/MNN) (private) | MNN mirror with the GCC 16 build fix. |
| this repository | The index. |

## Conventions

* Numbers are only quoted with the conditions they were measured under. Where
  a figure in a linked repository was later withdrawn, the withdrawal is kept
  in place next to it rather than deleted; `got-bringup/CHECKLIST.md` is the
  running log of what was believed and when it stopped being believed.
* Nothing here contains machine-identifying data (hostnames, LAN addresses,
  registry dumps). Wine registry backups in particular stay out of git.
