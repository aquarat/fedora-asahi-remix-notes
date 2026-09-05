# M1 Max, Fedora Asahi Remix 44

| | |
|---|---|
| hardware | Apple M1 Max (G13C C0), 32 GB |
| OS | Fedora Linux Asahi Remix 44, KDE Plasma edition |
| kernel | 7.1.6-400.asahi.fc44.aarch64+16k (16 KB pages) |
| system Mesa | 26.1.8 (Fedora), with `libvulkan_asahi.so` replaced by the `aquarat/mesa` `local-deploy` build for gaming; see `got-bringup/deploy-system-driver.sh` and `INSTALL.md` |
| muvm / libkrun / libkrunfw | 0.6.0 / 1.19.0 / 5.5.0 |
| FEX | 2604 (Fedora package; upstream is newer, see below) |
| kernel cmdline | `appledrm.show_notch=1` is kept deliberately |

## Transparent huge pages for muvm guests

`/etc/tmpfiles.d/thp-muvm.conf`:

```
w /sys/kernel/mm/transparent_hugepage/enabled - - - - always
w /sys/kernel/mm/transparent_hugepage/defrag - - - - defer+madvise
w /sys/kernel/mm/transparent_hugepage/hugepages-2048kB/enabled - - - - always
```

Why: libkrun never calls `madvise(MADV_HUGEPAGE)` on guest RAM, so under the
stock `madvise` policy every guest mapping reports `THPeligible=0` and the
guest gets no huge-page backing at all. `always` makes it eligible;
`defer+madvise` moves compaction off the fault path, which otherwise shows as
frame stutter; and the 2 MB mTHP size matters because the PMD granule on a
16 KB-page kernel is 32 MB, which is rarely available once memory fragments.

Measured after a reboot, allocating 8 GB of anonymous memory inside the
guest: 8004 of 8192 MB huge-backed, zero fallbacks. Measurement gotcha:
`/dev/shm` inside muvm is the host's tmpfs, so writing there does not
exercise guest RAM.

## Known limits of the stack as packaged

* FEX 2604 is what Fedora 44 ships. Upstream 2605 fixes a cmpxchg flag bug
  and 2607 inlines trig and enables an L1 lookup cache by default; a source
  build has to rebuild `fex-emu-thunks` to match or GL thunking breaks.
* The libkrunfw guest kernel has no `CONFIG_TRANSPARENT_HUGEPAGE`, so THP
  tuning only applies on the host side.
* The guest x86 Mesa is 26.0.3, baked into the rootfs. The supported layering
  slot for a newer one is `/usr/share/fex-emu/overlays/mesa-{i386,x86_64}.erofs`.

## Project write-ups on this machine

* [Honeykrisp driver work for Ghost of Tsushima](../projects/honeykrisp-ghost-of-tsushima.md)
