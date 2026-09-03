# yolov8m

YOLOv8 object detection, 640x640. Four conversions of the same weights, kept together because comparing them is the fastest way to understand what the converter settings do. **Two of them are NCHW and eRT cannot load them** - see the Layout column.

**eRT `Model Type` = `1006`.** Output tensor is `[1, 84, 8400]` = 4 box + 80 COCO classes.

## Originals

Downloaded from Ultralytics' published assets, release `v8.4.0`:

    https://github.com/ultralytics/assets/releases/download/v8.4.0/yolov8m.pt

The `.onnx` is an export of that `.pt`, produced here - not a separate download.

| File | MB | SHA-256 (first 16) |
|------|----|--------------------|
| `yolov8m.onnx` |  99.0 | `68dae8ae7689b6da` |
| `yolov8m.pt` |  49.7 | `5d4a90cdc7a21786` |

## Converted

| File | MB | Input | Layout | Output | SHA-256 (first 16) |
|------|----|-------|--------|--------|--------------------|
| `yolo8m8.tfl` |  25.3 | FLOAT32 `[1, 3, 640, 640]` | **NCHW** | FLOAT32 `[1, 84, 8400]` | `9c60a8afc9f178a4` |
| `yolo8md.tfl` |  25.1 | FLOAT32 `[1, 640, 640, 3]` | NHWC | FLOAT32 `[1, 84, 8400]` | `591f81593d8a924e` |
| `yolo8mu8.tfl` |  25.2 | UINT8 `[1, 640, 640, 3]` | NHWC | FLOAT32 `[1, 84, 8400]` | `e1045b9acc44207c` |
| `yolov8m32.tfl` |  99.0 | FLOAT32 `[1, 3, 640, 640]` | **NCHW** | FLOAT32 `[1, 84, 8400]` | `9fc8ff54dd4074d4` |

See the [naming convention](../README.md#file-naming) for what each suffix
means. In short: **use `-dyn`**; `-int8` is published as a labelled specimen of
a failure mode, not as a usable model.

## Reproducing these

```bash
# 1. weights -> SavedModel. Run LOCALLY: this produces NHWC, which eRT needs.
#    The Ultralytics web UI emits NCHW, which eRT cannot load.
yolo export model=yolov8m.pt format=saved_model imgsz=640

# 2. SavedModel -> .tfl
python3 export_tflite_int8.py yolov8m_saved_model \
        -o yolov8m-dyn.tfl --quant dynamic

# 3. verify before it reaches a device (no dependencies)
python3 tfl_inspect.py yolov8m-dyn.tfl --camera 640x640x3
```

Scripts live in `ert-components/scripts/ai-utils/model-conversion/`.

## Licence

AGPL-3.0, as a derivative of Ultralytics YOLOv8. See [NOTICE](../NOTICE).
