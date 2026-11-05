# Validation

English | [日本語](VALIDATION.ja.md)

Checks performed on 2026-11-06 using a separate checkout prepared for publication.

| Check | Result |
| --- | --- |
| Renderer source comparison | All 20 files match the analytic-map renderer used on the headset, apart from the new kernel's license comment |
| Map compatibility | Passed 3,500 random cases, practical sizes, boundaries, padding and invalid-input checks against the previous scalar implementation |
| Memory safety | Map compatibility test passed with AddressSanitizer and UBSan |
| DXVK adapter | Published headers passed field-forwarding and invalid-map fallback checks |
| Projection and GAZ2 | Passed NDC/UV, vertical inversion, calibration, invalid-value, shutdown invalidation and 64-byte layout checks |
| Python unittest | Three tests passed |
| DXVK DLL build | LLVM-MinGW completed all 259 build steps; both d3d11.dll and dxgi.dll were verified as PE AMD64 |
| OpenVR publisher | Passed a syntax check with -Wall -Wextra -Werror against official SDK revision 0924064316de3effbcd1acf1e309182a2deb1c05 |
| Bounded CPU benchmark | Produced 20 records; unmeasured legacy timings are not reported as zero |

The DLL build completed with existing DXVK warnings concerning COM destructors.
Publication preparation did not restart the headset's game or change its power settings.
The publisher check above was a syntax check on macOS, not a new Linux ARM64 link or runtime test.

In the original headset setup, the user confirmed DFR rendering and gaze alignment
using the same renderer and OpenVR bridge. The newly generated DLLs from the public
checkout were not installed on the headset again during publication preparation.
Source hashes are recorded in [source-manifest.json](../source-manifest.json).

ARM64 CPU map microbenchmark improvements must not be treated as speedups for the
Windows x86-64/FEX DLL or for VRChat as a whole. Frame caps, CPU load and scene content
affect the result. New game versions, other devices and long-term stability have
not all been validated.

Published material consists of source, documentation and licenses. Personal device
configuration, gaze recordings, device logs, credentials, games and runtime binaries
are excluded.
