# yolov8m-pose

YOLOv8 pose estimation, 640x640. More accurate than the small model and about 2.4x the compute. Worth it when the frame rate budget allows.

**eRT `Model Type` = `1008`.** Output tensor is `[1, 56, 8400]` = 4 box + 1 class + 17 joints x 3.

## Originals

Downloaded from Ultralytics' published assets, release `v8.4.0`:

    https://github.com/ultralytics/assets/releases/download/v8.4.0/yolov8m-pose.pt

The `.onnx` is an export of that `.pt`, produced here - not a separate download.

| File | MB | SHA-256 (first 16) |
|------|----|--------------------|
| `yolov8m-pose.onnx` | 101.2 | `6445e635eabf2d0b` |
| `yolov8m-pose.pt` |  50.8 | `dbe539ea268db253` |

## Converted

| File | MB | Input | Layout | Output | SHA-256 (first 16) |
|------|----|-------|--------|--------|--------------------|
| `yolov8m-pose-dyn.tfl` |  25.7 | FLOAT32 `[1, 640, 640, 3]` | NHWC | FLOAT32 `[1, 56, 8400]` | `cc0bd5b33100383d` |
| `yolov8m-pose-int8.tfl` |  25.8 | UINT8 `[1, 640, 640, 3]` | NHWC | UINT8 `[1, 56, 8400]` | `617220094fbcff14` |

See the [naming convention](../README.md#file-naming) for what each suffix
means. In short: **use `-dyn`**; `-int8` is published as a labelled specimen of
a failure mode, not as a usable model.

## Reproducing these

```bash
# 1. weights -> SavedModel. Run LOCALLY: this produces NHWC, which eRT needs.
#    The Ultralytics web UI emits NCHW, which eRT cannot load.
yolo export model=yolov8m-pose.pt format=saved_model imgsz=640

# 2. SavedModel -> .tfl
python3 export_tflite_int8.py yolov8m-pose_saved_model \
        -o yolov8m-pose-dyn.tfl --quant dynamic

# 3. verify before it reaches a device (no dependencies)
python3 tfl_inspect.py yolov8m-pose-dyn.tfl --camera 640x640x3
```

Scripts live in `ert-components/scripts/ai-utils/model-conversion/`.

## Licence

AGPL-3.0, as a derivative of Ultralytics YOLOv8. See [NOTICE](../NOTICE).
