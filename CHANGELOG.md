# Changelog

## [1.0.0] — 2026-10-12

First release.

### Added

- `docs/API-NOTES.md` — five verified VEGAS Pro 20 scripting pitfalls, each with
  the failing code, the observed failure count, and the working code.
- `docs/REFLECTION.md` — how to dump the real installed API surface by
  reflection, so you stop guessing at member names and units.
- `docs/SCRIPT-FOLDERS.md` — where VEGAS actually looks for scripts, and the
  `csc.exe` compile line with the required references.
- `examples/PanCropAnimation.cs` — a minimal, compile-ready example that sets a
  Pan/Crop keyframe pair **in the correct order**, with the frame-count check
  that proves it worked.

### Notes

- Every pitfall in this repository was observed on VEGAS Pro 20, build 411.
  Where a claim could not be reproduced, it is marked as such rather than
  repeated.
