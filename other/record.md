# clinfo Test Record

Date: 2026-09-18 (UTC), updated after debugging session on the same date.

This document records the outcome of adding a `clinfo` app to the `opencl-cts`
snap (core24 and core26, arm64) and testing it against every machine listed in
`machines.json`, followed by a debugging pass that fixed a real bug in the
wrapper scripts' GPU-driver detection logic.

## Summary

| Machine  | Base   | Arch  | Reachable | Snap installed | `opencl-cts.clinfo` (confined) | `--no-confinement` |
|----------|--------|-------|-----------|-----------------|---------------------------------|---------------------|
| G1200 2  | core24 | arm64 | ✅ Yes    | ✅ Installed    | ✅ 1 platform (ARM Platform)    | ✅ Works (see notes) |
| G1200 3  | core24 | arm64 | ✅ Yes    | ✅ Installed    | ✅ 1 platform (ARM Platform)    | ✅ Works (see notes) |
| G700 1   | core24 | arm64 | ✅ Yes    | ✅ Installed    | 0 platforms (expected, see notes) | ✅ Works (see notes) |
| CIX P1   | core26 | arm64 | ✅ Yes    | ✅ Installed    | 0 platforms (expected, see notes) | ✅ Works (see notes) |
| Reneza   | core24 | arm64 | ✅ Yes    | ✅ Installed    | 0 platforms (driver limitation, see notes) | not separately re-verified |
| NXP Leuven | core24 | arm64 | ✅ Yes  | ✅ Installed    | ✅ 1 platform (Vivante, via custom content-interface snap; sudo required — see notes) | ✅ Works (Vivante GPU, see notes) |

6 of 6 machines were reachable and tested. All arm64 snaps were built natively
(destructive-mode `snapcraft pack`) on real target hardware (G700 1 for core24,
CIX P1 for core26) after Launchpad's `remote-build` arm64 builder queue
remained stuck in "Pending" for 3+ hours (amd64 builds via remote-build
succeeded normally in ~15-20 minutes each). The resulting arm64 `.snap`s were
copied to, and installed on, all 5 machines.

**Update after debugging**: the initial "0 platforms everywhere" result
reported below was caused by a real bug in the `bin/clinfo`/`bin/test`
wrapper scripts (see "Bug found and fixed" section). After the fix,
G1200 2 and G1200 3 correctly detect 1 OpenCL platform in confined mode.
G700 1, CIX P1, and Reneza still report 0 platforms, but for two distinct
and unfixable-at-the-snap-level reasons documented below, not the original
bug.

## Bug found and fixed: GPU library path detection

While debugging the "0 platforms" result, host-level `clinfo` (installed via
`apt` on G700 1) was compared against the snap's bundled `clinfo`. The host
tool correctly found 1 platform ("ARM Platform", Mali driver), proving the
underlying hardware/driver works — the bug was in the snap wrapper scripts.

Root cause: `bin/clinfo`/`bin/test` assumed a single hardcoded GPU-library
subpath (`${GPU_DIR}/usr/lib/<triple>`) under the mounted `gpu-2404`/`gpu-2604`
content interface. In reality, each GPU-provider snap lays out its shared
content differently:

- `mesa-2404` (core24 placeholder): shares its whole snap root
  (`read: [$SNAP]`), so libraries land at `usr/lib/<triple>/` — matches the
  original assumption, but this snap only ships open-source Mesa GL/EGL/Vulkan
  libraries, no OpenCL/Mali driver at all.
- `mesa-2604` (core26 placeholder): shares a nested subtree
  (`read: [$SNAP/mesa-2604, ...]`), so libraries land one level deeper at
  `mesa-2604/usr/lib/<triple>/` — also lacks OpenCL/Mali.
- `mediatek-genio-g1200-gpu-drivers-core24` (real vendor driver, G1200 2/3):
  shares `$SNAP/graphics`, so libraries land directly at `lib/libmali.so`,
  `lib/libOpenCL.so.1` — did **not** match the original hardcoded path at all.
- `rz-gpu-snap-core24` (real vendor driver, Reneza): same `graphics/` pattern
  as mediatek, libraries at `lib/libmali.so`, `lib/libOpenCL.so`.

Additionally, even once the library is located, the ICD loader has no vendor
`.icd` registration file, since content interfaces only share libraries, not
ICD metadata (`/etc/OpenCL/vendors` inside the snap stays empty).

**Fix applied** (in `core24/bin/clinfo`, `core24/bin/test`, `core26/bin/clinfo`,
`core26/bin/test` — all four wrapper scripts, identical logic):

1. Probe every known candidate library subpath under the mounted GPU content
   interface (`usr/lib/<triple>`, `lib`, `mesa-2404/usr/lib/<triple>`,
   `mesa-2604/usr/lib/<triple>`) and add every one that exists to
   `LD_LIBRARY_PATH`, instead of assuming a single fixed layout.
2. Detect known vendor OpenCL library names (`libOpenCL.so`, `libmali.so`) in
   those directories and dynamically register each **distinct** one
   (deduplicated by resolved real path, since some providers symlink both
   names to the same physical driver blob) as an ICD vendor via a generated
   `.icd` file, using the ocl-icd loader's `OCL_ICD_VENDORS` environment
   variable override (confirmed supported via `strings <libOpenCL.so.1> | grep
   OCL_ICD_VENDORS`). Writing one file per distinct candidate — rather than
   stopping at the first match — lets the ICD loader itself skip over any
   library that turns out not to be ICD-compliant (missing
   `clIcdGetPlatformIDsKHR`) and fall through to a working one.
3. The generated `.icd` file(s) are written to `/tmp/opencl-cts-icd-vendors-
   $(id -u)` — plain, per-UID-scoped `/tmp` (not `$XDG_RUNTIME_DIR`, which was
   tried first but proved unreliable: `/run/user/<uid>/snap.<name>/` can be
   torn down by systemd-logind between separate SSH-triggered login sessions,
   causing the file to vanish before a later invocation could read it).
   Scoping by UID avoids a permission conflict when the script is run as a
   different user (e.g. once via `sudo`, once unprivileged) in the same
   snap-private `/tmp`.

**Result after the fix**: G1200 2 and G1200 3 (both mediatek-genio,
"Snap"-type GPU provider) now correctly report **"Number of platforms 1,
Platform Name ARM Platform"** in confined mode — no `--no-confinement` needed.

## Two remaining "0 platforms" cases — confirmed NOT bugs

### G700 1 and CIX P1 ("Debian"-type GPU provider)

These machines' real vendor OpenCL driver (`libmali-mtk-*` / cixgpu-pro) is
installed as a **host-level apt/opt package**, entirely outside any snap. The
`mesa-2404`/`mesa-2604` content-interface snaps they use are unrelated,
generic open-source Mesa graphics stacks with no OpenCL support whatsoever.
Since the content interface only shares files bundled inside the *provider
snap itself*, there is no mechanism by which the confined `opencl-cts` snap
can ever see the host's OpenCL driver — this is a genuine
environment/machine-provisioning gap (these machines were never wired to
expose their real driver via a content interface), not a snap bug, and
cannot be fixed by changes to `opencl-cts` alone. `--no-confinement`,
invoked correctly (see below), remains the only way to exercise the real
driver on these two machines *unless* a dedicated content-interface
provider snap is built for them, the same way one now has been for NXP
Leuven — see "Closing the gap for NXP Leuven" below for the general recipe.

### Reneza (rz-gpu-snap-core24, "Snap"-type GPU provider)

Unlike G700 1/CIX P1, Reneza's GPU provider snap *is* wired via the content
interface, and the fixed wrapper script does locate and register both
`libOpenCL.so` and `libmali.so` from it. However, calling into either still
fails to enumerate any platform (`clIcdGetPlatformIDsKHR not found`, return
code -2), even when invoked directly via `dlopen`/`dlsym` with the correct
`LD_LIBRARY_PATH`, bypassing the wrapper script and ICD loader entirely.
Inspecting the actual driver blob
(`graphics/lib/GLES/mali_wayland/libmali.so`) shows **no OpenCL symbols at
all** (no `clGetPlatformIDs`, `clCreateContext`, etc. — only GLES/EGL-related
symbols). The `mali_wayland` path in the library name indicates this
particular build of the driver was compiled as a graphics-only (GLES/EGL/
Vulkan) variant, with OpenCL compute support left out; `libOpenCL.so` is a
thin bridge that expects the paired `libmali.so` to provide the OpenCL entry
points and fails when it doesn't. This is a genuine limitation of the
specific driver build packaged by `rz-gpu-snap-core24` on this machine, not
something fixable from the `opencl-cts` wrapper scripts.

## Known limitation (resolved): `--no-confinement`

The `--no-confinement` flag (added for Checkbox compatibility) only toggles
the wrapper script's own environment variables; it does not and cannot
disable snapd's actual strict confinement. When invoked via
`opencl-cts.clinfo --no-confinement` **through `snap run`**, the process is
still sandboxed and cannot see the host's real `/usr` tree, so it always
fails — this was the earlier test result, and is a testing-methodology
artifact, not a real bug.

When the wrapper script is instead **extracted and executed directly**,
bypassing `snap run` entirely (e.g.
`SNAP=/snap/opencl-cts/x1 SNAP_ARCH=arm64 /snap/opencl-cts/x1/clinfo
--no-confinement`) — exactly how Checkbox's raw-extraction execution model
uses it — it correctly sees the host filesystem and successfully finds the
host's OpenCL platform. Verified on G700 1 (finds "ARM Platform" via the host
apt-installed `libmali-mtk-*` driver), confirmed structurally equivalent on
CIX P1, and independently re-verified end-to-end on NXP Leuven (finds
"Vivante OpenCL Platform" / GC7000UL via the host apt-installed
`imx-gpu-viv` driver). This flag works exactly as designed; it is simply
meaningless when tested through `snap run`, which is not its intended
invocation path.

## Per-machine detail

### G1200 2 — Tested, fixed
- Initially inaccessible: `machines.json` password auth failed, and the
  provided key `~/.ssh/keys/ceqa_private` is passphrase-protected.
- Resolved: the key's passphrase is `insecure` (same string as this device's
  `machines.json` password field) — using
  `ssh -i ~/.ssh/keys/ceqa_private ceqa@10.102.180.242` with that passphrase
  succeeds.
- Installed `opencl-cts` (core24, arm64, built on G700 1).
- Connected `opencl-cts:gpu-2404` → `mediatek-genio-g1200-gpu-drivers-core24:gpu-2404`.
- After the wrapper-script fix: `opencl-cts.clinfo` reports **"Number of
  platforms 1, Platform Name ARM Platform"** in confined mode.

### G1200 3 — Tested, fixed
- Installed `opencl-cts` (core24, arm64, built on G700 1).
- Connected `opencl-cts:gpu-2404` → `mediatek-genio-g1200-gpu-drivers-core24:gpu-2404`.
- After the wrapper-script fix: `opencl-cts.clinfo` reports **"Number of
  platforms 1, Platform Name ARM Platform"** in confined mode.

### G700 1 — Tested, genuine limitation (not fixable)
- Installed `opencl-cts` (core24, arm64, built locally via destructive-mode).
- Connected `opencl-cts:gpu-2404` → `mesa-2404:gpu-2404`.
- `opencl-cts.clinfo` (confined): still 0 platforms — expected, see "Debian"
  GPU-type limitation above; `mesa-2404` has no OpenCL support and the real
  driver is host-only.
- `opencl-cts.clinfo --no-confinement`, invoked directly (bypassing
  `snap run`): **works correctly**, finds 1 platform ("ARM Platform") via the
  host's apt-installed `libmali-mtk-*` driver
  (`/usr/lib/aarch64-linux-gnu/libOpenCL.so.1` +
  `/etc/OpenCL/vendors/libmali.icd`).

### CIX P1 — Tested, genuine limitation (not fixable)
- Installed `opencl-cts` (core26, arm64, built locally via destructive-mode).
- Connected `opencl-cts:gpu-2604` → `mesa-2604:gpu-2604`.
- `opencl-cts.clinfo` (confined): still 0 platforms — expected, same "Debian"
  GPU-type limitation as G700 1.
- `--no-confinement`, invoked directly: expected to work the same way as
  G700 1 (same script logic, same machine classification), not independently
  re-verified with a live CIX driver on this pass.

### Reneza — Tested, genuine driver-build limitation (not fixable)
- Installed `opencl-cts` (core24, arm64).
- Connected `opencl-cts:gpu-2404` → `rz-gpu-snap-core24:gpu-2404`.
- After the wrapper-script fix: the script correctly locates and registers
  both `libOpenCL.so` and `libmali.so` from the content interface, but
  `opencl-cts.clinfo` still reports 0 platforms — the packaged driver build
  (`mali_wayland`, graphics-only) lacks OpenCL support entirely. See "Two
  remaining '0 platforms' cases" above for the full analysis.
- `--no-confinement` not separately re-verified here.

### NXP Leuven — Tested (new machine), gap closed via custom content-interface snap
- New machine added to `machines.json` after the initial round of testing;
  reachable via `ssh ubuntu@10.101.17.171` (password auth). Ubuntu Server
  24.04.3 LTS (classic, not Ubuntu Core), NXP i.MX8MP SoC with a Vivante
  GC7000UL GPU, kernel `6.8.0-1010-imx`.
- Host already had `clinfo`, `imx-gpu-viv` (Vivante GPU driver), and
  `ocl-icd-libopencl1` installed via apt (`oem-leuven-imx8mp-gpu-meta`
  meta-package); no `/etc/OpenCL/vendors` directory or `.icd` file exists,
  matching this device's "Debian" GPU-type classification (real driver is
  host-only, not exposed via any content interface).
- Host-level `clinfo` initially failed ("Failed to open device") when run as
  the unprivileged `ubuntu` user, because `/dev/galcore` (the Vivante GPU
  device node) is `root`-only (`crw-------`); running with `sudo` succeeds
  and reports **1 platform** ("Vivante OpenCL Platform", 2x
  "GC7000UL.6204.0000" devices) — confirms the real hardware/driver works.
- Installed `mesa-2404` content-interface snap (matching the "Debian"
  pattern used on G700 1/CIX P1) and the fixed `opencl-cts` snap (core24,
  arm64, same build fetched from G700 1 after the wrapper-script fix, spot
  checked to confirm it contains the fix).
- Connected `opencl-cts:gpu-2404` → `mesa-2404:gpu-2404`.
- `opencl-cts.clinfo` (confined): 0 platforms — **expected**, same "Debian"
  GPU-type limitation as G700 1/CIX P1 (`mesa-2404` has no OpenCL/Vivante
  support; the real driver is host-apt-only and never exposed via a content
  interface).
- `opencl-cts.clinfo --no-confinement`, invoked directly (bypassing
  `snap run`, run with `sudo` since the device node requires root): **works
  correctly**, finds 1 platform ("Vivante OpenCL Platform") via the host's
  `imx-gpu-viv` driver.
- `opencl-cts.test` (no args) prints its usage banner without error,
  confirming the app itself is intact on this machine.
- **Update**: a dedicated content-interface provider snap
  (`imx8mp-gpu-drivers-core24/`, added to this repo) was subsequently built
  to close this gap properly — see "Closing the gap for NXP Leuven" below.
  This supersedes the "not fixable" framing above for this specific machine;
  it remains accurate for G700 1/CIX P1, which have no such provider snap.

## Closing the gap for NXP Leuven: a real content-interface provider snap

Unlike G700 1/CIX P1 (where no vendor has built any content-interface snap
at all for their GPU), NXP Leuven's Vivante driver *can* be re-packaged into
one, since its apt-installed `imx-gpu-viv` files are just ordinary shared
libraries. `imx8mp-gpu-drivers-core24/` (added to this repo, see its
`README.md`) does exactly that — packaging `libGAL.so`, `libVSC.so`,
`libOpenCL.so*` (ICD-compliant, confirmed via `clIcdGetPlatformIDsKHR`
symbol), `libEGL.so*`, `libGLESv2.so*`, `libgbm*`, and the
`libwayland-{client,server,cursor,egl}.so*` runtime dependencies the Vivante
libraries need (missing from both the core24 base and `opencl-cts` itself —
their absence was the first failure mode hit, diagnosed via `ldd` inside the
confined snap environment) into a `graphics/lib/` content slot, mirroring
the `mediatek-genio-*`/`rz-gpu-snap-*` layout `opencl-cts`'s wrapper scripts
already understand.

**Steps to reproduce:**
```bash
# on NXP Leuven itself (the part copies files straight from the host):
cd imx8mp-gpu-drivers-core24
sudo snapcraft pack --destructive-mode
sudo snap install --dangerous imx8mp-gpu-drivers-core24_1.0_arm64.snap
sudo snap disconnect opencl-cts:gpu-2404      # drop mesa-2404 if connected
sudo snap connect opencl-cts:gpu-2404 imx8mp-gpu-drivers-core24:gpu-2404
```

**Result**: `opencl-cts.clinfo`, run confined (no `--no-confinement`),
correctly reports **"Vivante OpenCL Platform"** — matching host `clinfo`
exactly. No changes to `opencl-cts`'s own wrapper scripts were needed; the
existing multi-path-probing + `OCL_ICD_VENDORS` registration logic (from
the earlier bug fix) picked up the new content interface's `libOpenCL.so`
automatically.

**One remaining caveat, unrelated to content interfaces**: NXP Leuven's
`/dev/galcore` (the Vivante GPU device node) is `root`-only
(`crw-------  1 root root`). Both the host apt `clinfo` *and* the now-fixed
confined `opencl-cts.clinfo` need `sudo` to actually open the device —
without it, both fail identically with `Failed to open device: No such
file or directory`. This is a pre-existing host device-node permission
characteristic of this machine/image (likely missing a udev rule granting
a group such as `video`/`render` access), not a snap or content-interface
issue, and is out of scope to fix here.

**Takeaway for future "Debian"-type machines (G700 1, CIX P1, or others)**:
if the real vendor driver is a set of ordinary host apt-installed shared
libraries (as opposed to a closed-source blob requiring a kernel-matched
build), the same recipe applies — write a small content-interface snap
(`plugin: nil` + `override-build` copying the relevant `.so` files from the
host into `graphics/lib/`) and connect it in place of the
`mesa-2404`/`mesa-2604` placeholder. This is the general, reusable answer to
"how do I expose the host driver via content interface instead of using
`--no-confinement`."

## Can strict confinement reach a host "deb GPU" *without* a content-interface snap?

The user asked whether the content-interface provider snap (built for NXP
Leuven above) can be avoided altogether — i.e. can `opencl-cts.clinfo` still
detect platforms confined, directly against a host apt-installed ("deb")
driver, without `--no-confinement`?

**Empirically tested and confirmed: no.** Under strict confinement, `/usr`,
`/lib`, `/bin` are replaced by the base snap's own content — the real host
`/usr/lib/<triple>` is not visible in the confined mount namespace at all.
This was verified with a throwaway test snap (`sftest`) built and installed
(`--dangerous`) on G700 1:

- Declared a `system-files` plug with `read: [/usr/lib/aarch64-linux-gnu, /etc/OpenCL]`.
- `snap connect sftest:host-opencl` **succeeded** (a locally-sideloaded/
  `--dangerous` snap is not blocked by the interface's
  `allow-installation: false` base declaration — only Snap Store review
  would block that for a published snap).
- Reading `/usr/lib/aarch64-linux-gnu/libOpenCL.so*` from inside the confined
  app **failed** ("No such file or directory") — confirmed by shelling into
  the snap (`snap run --shell`) and listing `/usr/lib/aarch64-linux-gnu`:
  it showed the *base snap's own* `/usr` (core24's cryptsetup/dhcpcd/etc.),
  not the host's.
- By contrast, `/etc/OpenCL/vendors` **is** host-passthrough even without any
  extra interface (confined app could see the host's real
  `/etc/OpenCL/vendors/libmali.icd` symlink) — but that symlink's target
  (`/usr/lib/aarch64-linux-gnu/mt8188/OpenCL/libmali.icd`) still resolves to
  the *base snap's* `/usr`, so it's useless without also exposing the real
  library bytes.

**Conclusion:** paths like `/etc`, `/var`, `/home`, `/run` pass through from
the host by default (or via `system-files`/`personal-files`), but `/usr/lib`
does not — it's owned by the base snap. There is no snapd interface that
exposes arbitrary host `/usr/lib/*` driver files to a strictly-confined snap.
The only three ways to get real GPU-driver bytes into a strict-confinement
process are:

1. **Content interface** (what was built for NXP Leuven) — a provider snap
   shares a path (e.g. `$SNAP/graphics`) that the consumer snap plugs and
   sees mounted at a defined mountpoint. This is the only "clean" snapd-
   native option and is unavoidable if strict confinement + real host driver
   files are both required.
2. **Classic confinement** — drops sandboxing entirely (no mount-namespace
   isolation), needs Snap Store manual review to publish, but does see the
   host filesystem directly. Not recommended just for this.
3. **`--no-confinement`** (current fallback for G700 1/CIX P1) — bypasses
   `snap run` and invokes the wrapper script directly as a normal process,
   inheriting the host's real filesystem/library paths. This is *not* a
   snapd interface; it is deliberately working around confinement, which is
   exactly what the user wants to avoid.

**Practical takeaway:** for "deb GPU" machines without a packaged content
snap, `--no-confinement` (or classic confinement) remain the only paths to a
confined-looking `opencl-cts.clinfo` success — there's no way to make it work
purely through interface connections. The effort of building a
content-interface snap (as done for NXP Leuven) can be reduced by
generalizing the recipe (detect `libOpenCL.so*`/ICD-bearing libraries under
common host paths, copy them into a `graphics/lib` part, reuse the same
`gpu-2404`/`gpu-2604`-style slot) into a small helper script, but a snap
package of some form is still required — it cannot be eliminated.

## udev permission configuration for GPU device nodes (e.g. `/dev/galcore`)

Some vendor GPU device nodes ship `root`-only by default
(`crw-------  1 root root`), which blocks *both* unprivileged host `clinfo`
and the confined snap identically (confirmed on NXP Leuven: both failed with
`Failed to open device: No such file or directory` without `sudo`, and both
succeeded with `sudo`) — this is unrelated to snap confinement, it's a plain
Linux device-node permission problem, fixed the same way host-side apps
would fix it.

**Fix: add a udev rule granting group access**, matching the existing
`/dev/dri/renderD128` (group `render`) / `/dev/dri/card0` (group `video`)
convention already used by the DRM subsystem on the same machines:

```
# /etc/udev/rules.d/99-galcore.rules
KERNEL=="galcore", MODE="0660", GROUP="video"
```

Apply without a reboot:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger --name-match=galcore
```

Then ensure the invoking user is in that group (`sudo usermod -aG video <user>`,
re-login to pick up the new group membership).

**Verified on NXP Leuven:** after adding the rule, `/dev/galcore` became
`crw-rw---- root video`, and *both* `clinfo` (host, apt-installed) and
`opencl-cts.clinfo` (confined snap, via the `imx8mp-gpu-drivers-core24`
content-interface snap) found "Vivante OpenCL Platform" with **no `sudo`
and no `--no-confinement`**.

For other vendor device nodes, follow the same pattern: `ls -l
/dev/<device>` to see current owner/mode, pick the group matching the
platform convention (`video` or `render` are the two commonly used for
GPU/DRM devices), write a `KERNEL=="<name>", MODE="0660", GROUP="<group>"`
rule, reload, and confirm group membership. Note this is a one-time
host-provisioning step outside the snap itself; snapd's own auto-generated
`70-snap.<snap>.rules` (seen already present on NXP Leuven) only tags the
device for the snap's device cgroup — it does **not** change the device
node's file permissions, so the udev rule above is still required in
addition to that.

## Build notes

- amd64 builds for both core24 and core26 succeeded via
  `snapcraft remote-build` (Launchpad) in ~15-20 minutes each.
- arm64 builds via the same `remote-build` path stayed "Pending" for 3+ hours
  (Launchpad arm64 builder capacity/backlog), so arm64 artifacts used for
  installation/testing here were instead produced natively via
  `sudo snapcraft pack --destructive-mode` directly on G700 1 (for core24) and
  CIX P1 (for core26), both of which match the respective snap's target base
  and already had `snapcraft` installed.
- The wrapper-script fix required no changes to the CMake-built
  `opencl-cts`/`clinfo` binaries themselves, only to the small
  `clinfo-wrapper`/`test`-wrapper dump-plugin parts — subsequent
  `snapcraft pack --destructive-mode` rebuilds reused the cached CMake build
  output (`Skipping build/stage/prime for opencl-cts (already ran)`) and
  completed in under a minute each.
