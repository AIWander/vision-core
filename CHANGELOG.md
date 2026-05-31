# Changelog

All notable changes to `vision-core` are documented here.
This project adheres to [Semantic Versioning](https://semver.org/).

## v0.2.0

### Added
- **Pluggable OCR backend** with simple env/enum dispatch (no async-trait, no
  trait objects). The public `ocr_image` / `ocr_image_with_positions` functions
  now route to a backend selected at runtime via the `VISION_CORE_OCR_BACKEND`
  environment variable.
- **Experimental PaddleOCR-ONNX backend** behind the `--features onnx` Cargo
  feature — the cross-platform OCR foundation for non-Windows targets (where
  `Windows.Media.Ocr` does not exist). Implements the PP-OCRv4 two-stage
  pipeline (DBNet detection -> CRNN/SVTR recognition + CTC greedy decode) on
  ONNX Runtime via the `ort` crate. See `docs/paddle-onnx.md`.
- `vision_ocr_backends()` capability reporter (also exposed through the
  `execute()` dispatch as the `vision_ocr_backends` tool) reporting the default
  backend, the compiled/available backends, the active backend, and whether the
  three Paddle model env vars are set and present.

### Changed
- **Default build is unchanged**: with no features enabled, `Windows.Media.Ocr`
  remains the one and only OCR path, behavior is identical, and the build pulls
  in neither `ort` nor `ndarray` nor the ONNX Runtime. The `onnx` feature is
  strictly additive.
- Version bumped `0.1.1` -> `0.2.0`.

### Stability
- **Public API is stable.** `ocr_image(image_path: &str, language: &str)` and
  `ocr_image_with_positions(image_path: &str)` keep their exact signatures;
  only their internals were refactored (the previous Windows bodies moved
  verbatim into private `windows_ocr_*` functions). Consumers (AI-Hands,
  browser-mcp, hands) require no changes.

### Notes / known limitations
- The PaddleOCR-ONNX backend is **experimental and not yet runtime-verified**
  against real model files. Tensor I/O names and a few shape conventions depend
  on how the PP-OCRv4 model was exported (paddle2onnx flags / PaddleOCR
  version); those spots are marked `// ASSUMPTION (PP-OCRv4 default): ...` in
  `src/paddle_onnx.rs`. When the three model env vars are unset or missing, the
  backend returns a clear error and never panics.
- `ndarray` is pinned to **0.17** (not 0.16) to match the version `ort` 2.x
  uses internally; mixing two `ndarray` major versions would prevent
  `Tensor::from_array` / `try_extract_array` from type-unifying.
- `ort` is pinned to the explicit pre-release `=2.0.0-rc.12` because Cargo will
  not select an `-rc` version from a plain `"2"` requirement. Bump to a caret
  range once `ort` cuts a stable 2.x release.

## v0.1.1
- Windows-only baseline: `Windows.Media.Ocr` text + word-box extraction,
  screenshot/region capture, image diff, and template matching.
