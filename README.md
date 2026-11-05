# DXVK — Steam Frame DFR

English | [日本語](README.ja.md)

An experimental DXVK fork that brings **Dynamic Foveated Rendering (DFR)** to
Windows VRChat rendered locally on Steam Frame. It uses a Vulkan fragment
shading rate attachment to preserve detail around the gaze point while reducing
shading work in the surrounding area.

This fork is based on DXVK and existing gaze-driven VRS work by ReinaS-64892,
poprox24 and other contributors. The `steam-frame-dfr` branch contains the
Frame-specific integration. See [Source provenance](frame/PROVENANCE.md) for the
exact source revisions and retained upstream components.

## Frame-specific changes

- A native OpenVR Background publisher reads per-eye foveation centers through
  `GetEyeTrackedFoveationCenter`.
- NDC-to-UV conversion, optional vertical inversion and per-user calibration
  connect those centers to the existing GAZ2 protocol.
- VRS uploads and dynamic state handling preserve the secondary command buffer
  path used by the tiler renderer.
- Devices without layered shading-rate attachments use the supported
  single-layer path; unsupported layered eye passes retain full-rate shading.
- An analytic-boundary CPU map kernel fills contiguous spans and reuses storage.
- Standalone tests cover map compatibility, boundaries, invalid inputs,
  projection and the GAZ2 packet layout.

The upstream asynchronous worker, GAZ2 protocol and VRS resource management
remain part of the implementation.

## How to use

Start with an existing standalone PCVR setup that can already run Windows
VRChat in VR on Frame. This repository supplies the DFR source and gaze bridge;
it does not install FEX, Proton, SteamVR or VRChat.

1. Build the Windows x86-64 `d3d11.dll` and `dxgi.dll` from this branch.
2. Install them into a separate, writable Proton environment with the original
   DLLs retained for restoration.
3. Build and run the native Linux ARM64 OpenVR gaze publisher.
4. Pass `DXVK_GAZE_VRS=1` and the correct `DXVK_GAZE_FILE` path to the Windows
   DXVK process. Both processes must see the same GAZ2 file.

The [build and usage guide](frame/docs/BUILD_AND_USE.en.md) covers DLL placement,
FEX environment forwarding, gaze alignment and restoration.
The bridge runs as an OpenVR Background application and does not create another
scene session. Invalid or stale gaze data falls back to full-rate shading.

### Requirements

- An existing Steam Frame standalone PCVR environment.
- Vulkan attachment fragment shading rate support through
  `VK_KHR_fragment_shading_rate`.
- An OpenVR runtime and matching SDK exposing
  `GetEyeTrackedFoveationCenter`.
- DXVK's build dependencies for the Windows x86-64 DLLs.
- A Linux ARM64 C++17 toolchain and OpenVR library for the gaze publisher.
- Python 3 and a C++17 compiler for the standalone tests.

### Configuration

| Variable | Purpose |
| --- | --- |
| `DXVK_GAZE_VRS` | `1` enables gaze VRS; `0` disables it |
| `DXVK_GAZE_FILE` | Wine-visible path to the publisher's GAZ2 file |
| `DXVK_GAZE_DOT` | `1` enables alignment dots on supported SBS passes; normally `0` |

See the [usage guide](frame/docs/BUILD_AND_USE.en.md) for region controls and
bridge calibration. FEX guest environment forwarding depends on the launcher;
setting only the host's Steam launch options may be insufficient.

## Build instructions

Clone this branch with its submodules:

```sh
git clone --branch steam-frame-dfr --recurse-submodules https://github.com/ji10me/dxvk-steam-frame-dfr.git
cd dxvk-steam-frame-dfr
```

### Building DLLs

With DXVK's dependencies and a suitable MinGW-w64 toolchain installed:

```sh
meson setup build-frame --cross-file build-win64.txt --buildtype release \
  -Denable_d3d8=false -Denable_d3d9=false -Denable_d3d10=false
meson compile -C build-frame
```

The relevant outputs are `build-frame/src/d3d11/d3d11.dll` and
`build-frame/src/dxgi/dxgi.dll`. They run as Windows x86-64 code under FEX;
the map kernel is not offloaded to a native ARM64 service.

For toolchain requirements, see the [original DXVK build instructions](README.upstream.md#build-instructions).
The [Frame guide](frame/docs/BUILD_AND_USE.en.md) contains the native OpenVR
publisher build commands.

### Standalone tests

```sh
python3 -m unittest discover -s frame/tests -p 'test_*.py' -v
```

These tests do not connect to a VR runtime or a game.

## Validation

The original headset setup has been tested in VRChat, with the user confirming
rendering and gaze alignment. The published renderer matches that tested
analytic-map source, apart from a license comment.
A fresh DLL build and three standalone tests passed during publication.
See [Validation](frame/docs/VALIDATION.md) for the checks and their limits.

CPU map microbenchmarks are not measurements of game-wide FPS gains.
Performance depends on the scene, CPU load and frame pacing.

## Credits and license

- [DXVK](https://github.com/doitsujin/dxvk): the translation layer and base renderer.
- [ReinaS-64892's DFR branch](https://github.com/ReinaS-64892/dxvk/tree/dfr)
  and [vrchat-dfr-linux](https://github.com/ReinaS-64892/vrchat-dfr-linux):
  related DFR work. This fork imports the pinned patch identified in
  [PROVENANCE](frame/PROVENANCE.md).
- poprox24 and the source contributors: retained upstream work and notices.
- ji10me: the Frame-specific follow-ups described above.

DXVK and the Frame additions use the **zlib/libpng license (SPDX: Zlib)**;
see [LICENSE](LICENSE) and [frame/LICENSE](frame/LICENSE).
The GAZ2 shared header retains its **MIT** notice, with the upstream license
statement in [DFR-UPSTREAM.txt](frame/licenses/DFR-UPSTREAM.txt).
The separate standalone PCVR kit's personal-evaluation restrictions do not
apply to this public DFR fork.

The fork contains DFR source, tests, documentation and licenses. Device logs,
credentials, recorded gaze, video decoder patches and proprietary runtime
binaries are not included. The original DXVK README is retained as
[README.upstream.md](README.upstream.md).
