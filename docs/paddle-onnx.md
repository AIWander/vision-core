# PaddleOCR-ONNX backend (experimental)

`vision-core` ships an **experimental** cross-platform OCR backend built on
PaddleOCR's PP-OCRv4 models running under ONNX Runtime. It exists so that the
library can do OCR on **macOS and Linux**, where the default
`Windows.Media.Ocr` engine does not exist.

> **Status: experimental and unverified.** The pipeline compiles and is
> structurally complete, but it has **not** been validated end-to-end against
> real model files. The default `Windows.Media.Ocr` backend remains the only
> verified OCR path. Treat output from this backend as best-effort until you've
> confirmed it on your own models.

The whole backend is gated behind the `onnx` Cargo feature. A default
`cargo build` does **not** compile it and pulls in neither `ort` nor `ndarray`
nor the ONNX Runtime.

---

## 1. Build with the feature

```bash
cargo build --features onnx
```

The `ort` dependency uses its `download-binaries` feature, so the correct ONNX
Runtime shared library is fetched automatically at build time — you do **not**
need a preinstalled runtime.

## 2. Get the PP-OCRv4 ONNX models

You need three artifacts: a **detection** model, a **recognition** model, and a
**character dictionary**. PaddleOCR distributes Paddle inference models; convert
them to ONNX with [`paddle2onnx`](https://github.com/PaddlePaddle/Paddle2ONNX).

```bash
# 1. Download PP-OCRv4 inference models (English example) from the model zoo:
#    https://paddleocr.bj.bcebos.com/PP-OCRv4/english/
wget https://paddleocr.bj.bcebos.com/PP-OCRv4/english/en_PP-OCRv4_det_infer.tar
wget https://paddleocr.bj.bcebos.com/PP-OCRv4/english/en_PP-OCRv4_rec_infer.tar
tar xf en_PP-OCRv4_det_infer.tar
tar xf en_PP-OCRv4_rec_infer.tar

# 2. Convert each to ONNX (opset 11; keep dynamic shapes — the default in
#    paddle2onnx >= 1.2.3 — so results match Paddle):
pip install paddle2onnx
paddle2onnx --model_dir ./en_PP-OCRv4_det_infer \
    --model_filename inference.pdmodel \
    --params_filename inference.pdiparams \
    --save_file ./det.onnx --opset_version 11 --enable_onnx_checker True
paddle2onnx --model_dir ./en_PP-OCRv4_rec_infer \
    --model_filename inference.pdmodel \
    --params_filename inference.pdiparams \
    --save_file ./rec.onnx --opset_version 11 --enable_onnx_checker True
```

(Prebuilt ONNX exports also exist, e.g.
[RapidAI/PaddleOCRModelConvert](https://github.com/RapidAI/PaddleOCRModelConvert)
and the `RapidOCR` model releases — any standard PP-OCRv4 det/rec ONNX export
should work.)

The **character dictionary** is the `*_dict.txt` / `ppocr_keys_v1.txt` that
matches your recognition model — one token per line, in the class-index order
the model emits. It ships inside PaddleOCR
(`ppocr/utils/en_dict.txt`, `ppocr/utils/ppocr_keys_v1.txt`, etc.). Use the one
that corresponds to the language of your `rec` model.

## 3. Point vision-core at the models

Set these three environment variables to absolute paths:

| Variable                         | What it is                                  |
| -------------------------------- | ------------------------------------------- |
| `VISION_CORE_PADDLE_DET_MODEL`   | detection `.onnx` (DBNet)                    |
| `VISION_CORE_PADDLE_REC_MODEL`   | recognition `.onnx` (CRNN/SVTR)             |
| `VISION_CORE_PADDLE_DICT`        | character dictionary `.txt`                  |

If any is unset or points at a missing file, the backend returns a clear error
(it never panics).

## 4. Activate the backend at runtime

```bash
export VISION_CORE_OCR_BACKEND=paddle     # also accepts: onnx, paddleocr
export VISION_CORE_PADDLE_DET_MODEL=/abs/path/det.onnx
export VISION_CORE_PADDLE_REC_MODEL=/abs/path/rec.onnx
export VISION_CORE_PADDLE_DICT=/abs/path/en_dict.txt
```

With `VISION_CORE_OCR_BACKEND` unset (or any value other than
`paddle`/`onnx`/`paddleocr`), `ocr_image` / `ocr_image_with_positions` use the
default `Windows.Media.Ocr` engine exactly as before.

PowerShell equivalent:

```powershell
$env:VISION_CORE_OCR_BACKEND = "paddle"
$env:VISION_CORE_PADDLE_DET_MODEL = "C:\models\det.onnx"
$env:VISION_CORE_PADDLE_REC_MODEL = "C:\models\rec.onnx"
$env:VISION_CORE_PADDLE_DICT = "C:\models\en_dict.txt"
```

## 5. Check what's active

```rust
let info = vision_core::vision_ocr_backends();
println!("{}", serde_json::to_string_pretty(&info).unwrap());
```

```jsonc
{
  "default": "windows_media_ocr",
  "available": ["windows_media_ocr", "paddleocr_onnx"], // paddleocr_onnx only if built with --features onnx
  "onnx_compiled": true,
  "active": "paddleocr_onnx",
  "paddle_models_present": true,   // all 3 env paths set AND exist
  "note": "PaddleOCR-ONNX is experimental; ..."
}
```

The same data is available through the tool dispatch as `vision_ocr_backends`.

---

## How it works (pipeline)

1. **Detection (DBNet).** The image is resized so its longest side is at most
   960px and both sides are multiples of 32, normalized (ImageNet mean/std,
   NCHW), and run through the detection model. The output probability map is
   binarized and flood-filled into connected components; each component's
   axis-aligned bounding box (scaled back to source pixels) becomes a text
   region.
2. **Recognition (CRNN/SVTR).** Each region is cropped, resized to a fixed
   height of 48px (width capped at 320px), normalized to `[-1, 1]`, and run
   through the recognition model. The per-timestep class logits are CTC
   greedy-decoded (blank at index 0, repeats collapsed) against the dictionary.

Loaded `ort::Session`s are cached per model path, so repeated calls don't
re-parse the ONNX graph.

## Assumptions (PP-OCRv4 defaults)

These hold for standard `paddle2onnx` PP-OCRv4 exports but vary with some
exports. They are marked `// ASSUMPTION (PP-OCRv4 default): ...` in
`src/paddle_onnx.rs`; adjust there if your model differs:

- Detection and recognition inputs are both named **`x`** (one input each).
- The primary output is the model's **first** graph output.
- Detector normalization uses **ImageNet** mean/std; the recognizer normalizes
  to **`[-1, 1]`** (0.5 mean/std).
- CTC **blank is class index 0**, so dictionary line `i` maps to class `i + 1`.
- Recognition input height is **48px**.

If your export uses different input names, normalization, or blank position,
edit the corresponding constants/strings in `src/paddle_onnx.rs`.
