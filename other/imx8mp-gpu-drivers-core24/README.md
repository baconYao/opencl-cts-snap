# imx8mp-gpu-drivers-core24

A content-interface "provider" snap that re-shares the NXP i.MX8MP Vivante
(GC7000UL) OpenCL/EGL/GLES driver stack — already installed on the host via
the `imx-gpu-viv` apt package (part of `oem-leuven-imx8mp-gpu-meta`) — so that
confined snaps such as `opencl-cts` can use the real vendor OpenCL
implementation **without** needing `--no-confinement`.

This mirrors the same content-interface pattern used by
`mediatek-genio-g1200-gpu-drivers-core24` and `rz-gpu-snap-core24` for their
respective vendor drivers: the provider snap exposes a `gpu-2404` content
slot at `$SNAP/graphics`, with the actual shared libraries placed under
`graphics/lib/`, which `opencl-cts`'s wrapper scripts (`bin/clinfo`/`bin/test`)
already know how to probe and register with the ICD loader.

## Why this is needed

Machines classified as `"GPU Type": "Debian"` in `machines.json` (G700 1,
CIX P1, and originally NXP Leuven) have their real vendor OpenCL driver
installed purely as host-level apt packages, with no content-interface snap
ever built to expose it — meaning `opencl-cts.clinfo` can only see the real
driver via `--no-confinement` (which requires bypassing `snap run` entirely,
per `record.md`). This snap closes that gap for NXP Leuven specifically by
packaging the already-installed `imx-gpu-viv` driver files into a proper
content-interface provider, the same way it would be done "for real" (i.e.
by the vendor/OEM shipping their own driver snap, as MediaTek and Renesas
already do for their platforms).

## Building

This snap can **only** be built directly on an NXP Leuven-class machine (or
any other i.MX8MP device with `imx-gpu-viv` installed via apt), because its
single part copies files straight from the host filesystem
(`/usr/lib/aarch64-linux-gnu/lib{GAL,VSC,EGL,GLESv2,OpenCL,gbm,wayland-*,...}*`)
rather than building anything from source:

```bash
# On the target machine (e.g. NXP Leuven):
cd imx8mp-gpu-drivers-core24
sudo snapcraft pack --destructive-mode
```

## Installing and connecting

```bash
sudo snap install --dangerous imx8mp-gpu-drivers-core24_1.0_arm64.snap
sudo snap disconnect opencl-cts:gpu-2404          # drop the mesa-2404 placeholder, if connected
sudo snap connect opencl-cts:gpu-2404 imx8mp-gpu-drivers-core24:gpu-2404
```

## Verified result

With this snap connected, `opencl-cts.clinfo` (confined, no
`--no-confinement`) correctly detects **"Vivante OpenCL Platform"**
(GC7000UL) — matching the host apt `clinfo` output exactly.

**Caveat (unrelated to the content-interface mechanism itself) — fixed:**
the Vivante GPU device node `/dev/galcore` on this machine shipped
`root`-only (`crw-------`), so both the host apt `clinfo` *and* the snap's
confined `opencl-cts.clinfo` needed `sudo` to open the device — a
pre-existing host device-node permission characteristic, not something
introduced by the content interface or `opencl-cts` itself. Fixed with the
udev rule in [`99-galcore.rules`](./99-galcore.rules) (see that file for
install/verify steps): after installing it and adding the user to the
`video` group, both host and confined `clinfo` find the platform with
**no `sudo` required**.
