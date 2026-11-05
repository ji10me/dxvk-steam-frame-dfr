# Build and usage guide

English | [日本語](BUILD_AND_USE.ja.md)

## 1. Prerequisites

This guide assumes Windows VRChat already runs in VR on a standalone Steam Frame.
This source alone does not install FEX, Proton, SteamVR or VRChat.
The GPU driver must support the attachment feature of
`VK_KHR_fragment_shading_rate`, and the OpenVR runtime must provide
`GetEyeTrackedFoveationCenter`.

The implementation has two parts:

1. A Linux ARM64 bridge reads per-eye foveation centers through OpenVR Background.
2. Windows x86-64 DXVK reads a GAZ2 file and applies a VRS map to supported rendering passes.

Invalid or stale gaze data and unsupported rendering passes use full-rate shading.

## 2. Build the DXVK DLLs

Install the dependencies described in the
[DXVK build instructions](../../README.upstream.md#build-instructions).
The following example uses MinGW-w64, Meson and Ninja on Linux.

```sh
git submodule update --init --recursive
meson setup build-frame --cross-file build-win64.txt --buildtype release \
  -Denable_d3d8=false -Denable_d3d9=false -Denable_d3d10=false
meson compile -C build-frame
```

The relevant outputs are `build-frame/src/d3d11/d3d11.dll` and
`build-frame/src/dxgi/dxgi.dll`. For LLVM-MinGW, provide a Meson cross file that
matches your compiler. These DLLs target Windows x86-64 and run under FEX on
Frame. The CPU map kernel is not moved to a native ARM64 background service.

## 3. Build the OpenVR bridge

Build the bridge on Frame or a compatible Linux ARM64 system.
Obtain Valve's [OpenVR SDK](https://github.com/ValveSoftware/openvr) with the
required API. SDK revision `0924064316de3effbcd1acf1e309182a2deb1c05` was checked
when this fork was prepared for publication. The SDK interfaces must match the
SteamVR runtime used on the device.

```sh
mkdir -p frame/external frame/bin
git clone https://github.com/ValveSoftware/openvr.git frame/external/openvr
git -C frame/external/openvr checkout 0924064316de3effbcd1acf1e309182a2deb1c05
OPENVR_SDK="$PWD/frame/external/openvr"
OPENVR_LIB_DIR="$OPENVR_SDK/lib/linuxarm64"
c++ -std=c++17 -O2 -Wall -Wextra -Werror \
  -I"$OPENVR_SDK/headers" -Iframe/include frame/tools/frame_gaze_bridge.cpp \
  -L"$OPENVR_LIB_DIR" -Wl,-rpath,"$OPENVR_LIB_DIR" -lopenvr_api -pthread \
  -o frame/bin/frame-gaze-bridge
```

`frame/external` and `frame/bin` are ignored by Git.
For independent testing on Linux x86-64, select the matching SDK library directory.

## 4. Place the DLLs in a private Proton environment

Use a separate, writable Proton environment and retain the original DLLs so that
you can restore them exactly. A common location is `files/lib64/wine/dxvk/`, but
check the actual layout of your Proton version.
Do not overwrite DLLs used by a running game or a shared Proton installation.
Overwriting a hard-linked DLL in a Prefix can also modify the original file;
replace it with a new file instead.

FEX and Proton launch arrangements differ between setups, so this fork does not
include an automatic deployment script. DLL placement alone does not guarantee
that they are selected. Verify that the game uses the intended private Proton
and DXVK build.

## 5. Enable DFR

Pass the following environment variables to the Windows DXVK process:

| Variable | Setting |
| --- | --- |
| `DXVK_GAZE_VRS` | `1` to enable, `0` to disable |
| `DXVK_GAZE_FILE` | Wine-visible path to the file written by the bridge |
| `DXVK_GAZE_DOT` | Normally `0`; use `1` for alignment |
| `DXVK_GAZE_FULL_AREA` / `DXVK_GAZE_MID_AREA` | Optional central/intermediate region controls, retaining upstream semantics |

The publisher writes to the current user's
`$HOME/.local/share/frame-pcvr/gaze/frame-gaze-v1.bin`.
With Wine's usual `Z:` mapping, the DXVK-side path looks like
`Z:\home\YOUR_USER\.local\share\frame-pcvr\gaze\frame-gaze-v1.bin`.
Replace `YOUR_USER` with your own user name and ensure both processes see the
same underlying file.

With FEX, host Steam launch options alone may not reach the Windows guest.
Check the FEX app config's `Env` settings and how the Steam container forwards
them. This public fork does not automatically modify device-specific FEX settings.

Launch VRChat, then identify its Linux process:

```sh
pgrep -a -x VRChat.exe
```

Run the bridge as the same user as SteamVR, using the PID you verified.
Create the parent directory; the bridge creates its own private directory and file.

```sh
mkdir -p "$HOME/.local/share/frame-pcvr"
VRCHAT_PID=12345  # Replace with your verified game process PID
frame/bin/frame-gaze-bridge "$VRCHAT_PID" 900 --flip-y
```

The second argument is the duration in seconds, from 1 to 1800.
The bridge exits at the deadline, when the game exits, or when it loses scene focus.
After a timed exit, verify the target PID again before restarting it.
No autostart entry or indefinitely running service is installed.
The bridge checks the game PID, start time and Steam App ID rather than following
an unrelated replacement process. It does not create another VR scene/session or
attach to game memory. On shutdown it invalidates the gaze packet and clears the
last gaze values. Raw gaze data is not logged.

## 6. Align the gaze position

`DXVK_GAZE_DOT=1` displays alignment dots on supported SBS passes.
If vertical movement is inverted, toggle the bridge's `--flip-y` option.
Do not apply an additional vertical flip in the DXVK reader at the same time.

If needed, add `--calibration GAIN_X GAIN_Y OFFSET_X OFFSET_Y`.
The defaults are gains of 1 and offsets of 0. Check and adjust these for your
own setup, then return `DXVK_GAZE_DOT` to `0`.

## 7. Disable or restore

Stop the bridge with Ctrl+C and use `DXVK_GAZE_VRS=0` in the guest to disable DFR.
To restore the previous DXVK build, exit the game, restore the saved original
DLLs and launch the original setup.
This repository does not change device-wide power settings, clocks, resolution
or avatar visibility limits.

## 8. Standalone tests

```sh
python3 -m unittest discover -s frame/tests -p 'test_*.py' -v
mkdir -p frame/bin
c++ -std=c++17 -O2 -Isrc/dxvk frame/tools/frame_foveation_bench.cpp -o frame/bin/map-bench
frame/bin/map-bench 128
```

The benchmark reports CPU map-generation time, not game FPS or GPU rendering time.
