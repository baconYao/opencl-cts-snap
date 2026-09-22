# OpenCL CTS Snap

This snap provides an easy way to install and run the tests found in
[Khronos's OpenCL Conformance Test Suite](https://github.com/KhronosGroup/OpenCL-CTS)

## Snap bases

The snap is maintained for multiple bases, each in its own self-contained
snapcraft project directory:

| Directory | Snap name                  | Base   | GPU content       | Arches       |
|-----------|----------------------------|--------|-------------------|--------------|
| `core22/` | `baconyao-opencl-cts-22`   | core22 | `graphics-core22` | amd64, arm64 |
| `core24/` | `baconyao-opencl-cts-24`   | core24 | `gpu-2404`        | amd64, arm64 |
| `core26/` | `baconyao-opencl-cts-26`   | core26 | `gpu-2604`        | amd64, arm64 |

The `core22`, `core24` and `core26` variants bundle Intel's compute-runtime
ICD on amd64, since it's amd64-only; on arm64 the OpenCL ICD is provided by
the GPU content snap (`mesa-2404`/`mesa-2604`) via the `gpu-2404`/`gpu-2604`
content interface.

Newer hardware needs newer userspace drivers. If a test fails at startup with
`clGetPlatformIDs failed`, the base you installed likely predates your GPU;
use a newer base.

In the Snap Store, each base is published as a separate snap on its `edge`
channel.

## Build

Each directory is a directly-buildable snapcraft project. `cd` into the base
you want and run snapcraft:

```
cd core22 && snapcraft pack
cd core24 && snapcraft pack
cd core26 && snapcraft pack
```

Each project supports both `amd64` and `arm64` via the `platforms` key; run
`snapcraft pack` on (or cross-build for) the target architecture, or use
`snapcraft remote-build` to build all platforms via Launchpad.

## Install

```
snap install --dangerous baconyao-opencl-cts-<base>_<version>_<your_arch>.snap
```

Or install the snap with the base that matches your hardware:

```
snap install baconyao-opencl-cts-22 --edge
snap install baconyao-opencl-cts-24 --edge
snap install baconyao-opencl-cts-26 --edge
```

The GPU content interface auto-connects for store installs. For a sideloaded
(`--dangerous`) install, connect it manually to match the base:

```
snap connect baconyao-opencl-cts-22:graphics-core22 mesa-core22:graphics-core22
snap connect baconyao-opencl-cts-24:gpu-2404 mesa-2404:gpu-2404
snap connect baconyao-opencl-cts-26:gpu-2604 mesa-2604:gpu-2604
```

## Run

Use the command namespace for the installed base. For example, with core24:

```
baconyao-opencl-cts-24.list-tests
```

Then run your chosen test from the previous list like this:

```
baconyao-opencl-cts-24.test basic/test_basic
```

To query the OpenCL platforms/devices visible to the snap, run:

```
baconyao-opencl-cts-24.clinfo
```

## Publishing credentials

Each workflow uses a credential restricted to its own snap. Export the three
credentials and save them as separate repository secrets:

```
snapcraft export-login core22-credentials.txt \
  --snaps=baconyao-opencl-cts-22 \
  --channels=edge
snapcraft export-login core24-credentials.txt \
  --snaps=baconyao-opencl-cts-24 \
  --channels=edge
snapcraft export-login core26-credentials.txt \
  --snaps=baconyao-opencl-cts-26 \
  --channels=edge

gh secret set SNAPCRAFT_STORE_CREDENTIALS_CORE22 < core22-credentials.txt
gh secret set SNAPCRAFT_STORE_CREDENTIALS_CORE24 < core24-credentials.txt
gh secret set SNAPCRAFT_STORE_CREDENTIALS_CORE26 < core26-credentials.txt
```

Do not commit the credential files.
