# Source provenance

This is an altered experimental DXVK fork for Steam Frame.

- DXVK: https://github.com/doitsujin/dxvk , base `0cfe0898289fb11e3d8d0e1c62cd344b825b7d57`.
  The original zlib/libpng notice remains in the root `LICENSE`.
- Gaze VRS integration: `patches/dxvk-gaze-vrs.patch` from
  https://github.com/ReinaS-64892/vrchat-dfr-linux at
  `e410c937d44f818c4de5806a7576f40381cec75f`.
  Original patch SHA256: `5a5d84b1b5d63c9d4f1c53699d0d78cde2bcfcfaead5b491c4d7b0c6b6c02b4a`.
  Credit includes ReinaS-64892 and poprox24 and their source contributors.
  The source repository states that DXVK patches use the same zlib/libpng
  license as DXVK; its GAZ2 protocol header is MIT. Its notice is retained in
  `licenses/DFR-UPSTREAM.txt`.
- Frame follow-ups by ji10me: tiler secondary-command-buffer handling,
  non-layered attachment fallback, analytic-boundary map kernel,
  OpenVR Background gaze publisher, NDC/UV conversion and calibration.
  They are marked as alterations and licensed under Zlib, like DXVK.

The upstream asynchronous worker, resource management, GAZ2 format and VRS
integration remain. This is not a clean-room rewrite or removal of all
upstream DFR code. Original copyrights and licenses are preserved.

No WiVRn patch, OpenXR bridge, headset provisioning, credentials, device logs,
recorded gaze, video decoder changes or proprietary runtime binaries are
included. Runtime paths are generated from each user's HOME, not an author's
machine configuration. The standalone PCVR kit is separate from this fork.
